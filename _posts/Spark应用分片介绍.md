---
layout: post
title:  "Spark 应用分片介绍"
date:   2018-05-11 13:57:12 +0800
tags:
      - Spark
---
## 引言
分布式计算的基本思路是将数据分为多个部分，将同样的数据操作方式在数据的不同部分上执行，分别获得结果，然后通过“汇聚处理”的方式得到结果。如何将数据分为多个部分（也就是“分片”）便是其中的一个重要组成部分。Spark框架同样对使用分片的操作，将数据分片（partition）处理。本文对Spark框架中的数据分片作简单介绍。

## 输入数据的分片
对于读取批数据生成rdd的操作，数据的分片都是通过输入文件格式本身提供的getSplit方法来对数据进行分片。
本部分主要介绍对于不同数据源的数据，spark如何定义/获取数据的分片数。

###  text文件分片（sc.textFile为例）:

     
    def textFile(
      path: String,
      minPartitions: Int = defaultMinPartitions): RDD[String] = withScope {
    assertNotStopped()
    hadoopFile(path, classOf[TextInputFormat]  /* 数据文件的输入格式 ：org.apache.hadoop.mapred.TextInputFormat */
    , classOf[LongWritable], classOf[Text],
      minPartitions).map(pair => pair._2.toString).setName(path)
    }
     hadoopFile方法生成HadoopRDD
     HadoopRDD(
      this,
      confBroadcast,
      Some(setInputPathsFunc),
      inputFormatClass,
      keyClass,
      valueClass,
      minPartitions) /* minPartitions为生成该RDD的最小分片数，表示该RDD的分片数最小值，默认为2*/
      .setName(path)

在执行action方法（如count）时，spark应用才真正开始计算，通过调用rdd.partitions.length计算出分片数

        def runJob[T, U: ClassTag](rdd: RDD[T], func: Iterator[T] => U): Array[U] = {
    runJob(rdd, func, 0 until rdd.partitions.length)
    }

通过跟踪该方法可以看出该函数最终会调用到HadoopRDD的getPartitions方法,在该方法中通过inputFormat的getSplit方法计算分片数
        
        getInputFormat(jobConf).getSplits(jobConf, minPartitions)

TextInputFormat继承至FileInputFormat，FileInputFormat的getSplit方法网上有许多分析，这里不再展开，大致的原理是根据文件个数，传入的minpartitions，mapreduce.input.fileinputformat.split.minsize等参数计算出分片数。

### hbase表分片
在读取HBase数据时，没有类似textFile的接口的封装，可调用如下接口生成给予hbase数据的RDD，

    val hBaseRDD = sc.newAPIHadoopRDD(conf, 
    classOf[TableInputFormat]， /*该类的全类名为：  org.apache.hadoop.hbase.mapreduce.TableInputFormat */
      classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],  
      classOf[org.apache.hadoop.hbase.client.Result])  
      
      该方法生成new NewHadoopRDD(this, fClass, kClass, vClass, jconf)

在执行action操作时，同样调用到rdd.partitions方法，跟踪至newHadoopRDD之后，发现调用到
    
    inputFormat.getSplits(new JobContextImpl(_conf, jobId))
    
查看对应的getSplits方法可以看出：

默认情况下（hbase.mapreduce.input.autobalance的值为false）hbase表如果存在多个region，则每个region设置为一个split。

如果设置了开启均衡（设置hbase.mapreduce.input.autobalance的值为true：在hbase的region大小不均衡引发的数据倾斜，将导致不同的region处理耗时较多，该参数为了解决此场景），则会在每个region对应一个split的基础上，将较小（小于平均大小）的region进行合并作为一个split，较大（大于平均size的三倍（其中三可配置））的region拆分为两个region。
        
        splits伪代码如下(源码可参考TableInputFormatBase.calculateRebalancedSplits)：
        
        while ( i < splits.size)
        {
            if（splits(i).size > averagesize * 3） {
            if(! splitAble)
                resultsplits.add(split(i))
            else{
                (split1,split2) = Split(splits(i))
                resultsplits.add(split1)
                resultsplits.add(split2)
            }
            i++ 
            }
            else if(splits(i).size > averagesize) {
                resultsplits.add(split(i))
                i++
            }else{
                startKey = split(i).getStartRow
                i++;
                while(totalSize + splits(i).size < averagesize * 3){
                    totalSize += splits(i).size
                    endKey = splits(i).getEndRow
                }
                resultsplits.add(new TableSplit(startKey,endKey,*))
            }
        }
        
### Kafka数据的分片
Spark框架在读取Kafka消息时，将Kafka数据抽象为KafkaRDD（SparkStreaming）或者KafkaSourceRDD（StructedStreaming），查看对应RDD的getPartitions方法和定义:


#### KafkaSourceRDD:

        override def getPartitions: Array[Partition] = {
    offsetRanges.zipWithIndex.map { case (o, i) => new KafkaSourceRDDPartition(i, o) }.toArray
     }
     offsetRanges的数据结构为
     private[kafka010] case class KafkaSourceRDDOffsetRange(
    topicPartition: TopicPartition,
    fromOffset: Long,
    untilOffset: Long,
    preferredLoc: Option[String]) 
可以看出partition个数为对应的TopicPartition的个数

#### KafkaRDD 

    override def getPartitions: Array[Partition] = {
        offsetRanges.zipWithIndex.map { case (o, i) =>
            new KafkaRDDPartition(i, o.topic, o.partition, o.fromOffset, o.untilOffset)
        }.toArray
      }
      offsetRanges数据结构为：
      final class OffsetRange private(
            val topic: String,
            val partition: Int,
            val fromOffset: Long,
            val untilOffset: Long) 
可以看出partition个数为对应的partition的个数

#### 总结
在spark框架中，对于输入数据获取RDD的处理：
* 读取数据时的分片由数据量，数据"存储格式"决定，框架/应用并不能真正决定分片数。
* 对于通过数据生成的RDD，如makeRDD，parallize等方法生成的RDD，则可以指定相应的RDD的分片数。
* 对于FileInputFormat格式的数据，可通过设置最小的分片数来扩大RDD分片数，但不能决定最终由多少分片数（最终分片数 >= 设置的最小分片数）
* 其他类型的数据/文件的分片方法也是通过输入文件格式的getSplit方法来获取分片
* Split方法直接决定了输入数据的分片数，影响应用并行度，在一些场景下，应用可以定制特定的getSplits方法来实现一些特殊需求。如hive在处理小文件时自定义了combineFileInputForamt，Hbase在以region为单位划分split之后，再跟进每个region数据量来合并/分拆split来优化性能

#### PS: 其他相关的数据分片
对于输入文件的分片，不同的文件格式使用的分片方法不尽相同。 如hive中使用的parquet，RCFIle格式文件，其getsplits方法直接使用的是FileInputFormat.getSplits， 而orc格式文件的getsplits方法则是继承于InputFormat

在Hive中默认使用的是CombineFileInputFormat，它的作用是在启动map时，会将多个小文件进行合并，已启动较少的map提升应用运行速度。其getsplits方法在合并小文件时会考虑更多的因素，如：
           
    mapreduce.input.fileinputformat.split.minsize，
    mapreduce.input.fileinputformat.split.minsize.per.node
    mapreduce.input.fileinputformat.split.minsize.per.rack
    mapreduce.input.fileinputformat.split.maxsize
 

## 经过转换的分片

* Spark框架中，RDD的分片数决定了对RDD处理时的并发度，因此合理的RDD分片数，对应用的性能有较大影响。

* RDD的转换通常不会改变RDD的partition数，如map，flatmap，mappartitions等操作并没有传入partition数的API，无法修改新生成的RDD的的分片数。可参考org.apache.spark.rdd.RDD

 
* 如果需要强制修改新生成RDD的分片数，可直接调用RDD.repartition,RDD.coalesce强制修改新生成RDD的分片数

* 对于RDD[KEY,VALUE]类型的RDD的操作如join，reduceByKey,aggregateByKey,combineByKey等接口可通过传入分片数/设置partitioner等方式设置shuffle之后的RDD的partition个数，从而调整后续的stage的并发task个数.可参考org.apache.spark.rdd.PairRDDFunctions

* 对于需要进行shuffle操作的算子，在变换的过程中，会自动生成shuffledRDD，该RDD的分片数可通过触发shuffle操作的算子调用时设置。如果没有设置时，则会使用默认的分片数。

* 对于普通的应用shuffle后的默认的分片数由spark.default.parallelism参数决定，默认200 对于sql相关的操作，shuffle后的默认分片数由spark.sql.shuffle.partitions操作决定，默认为200

* 对于某些特殊的操作，sql的内部优化可能会触发shuffle操作。如使用到treeaggregate会触发shuffle操作，shuffle后的partition数目默认为原始的开方。即原有2000个partition时，shuffle后的partition为44个。
