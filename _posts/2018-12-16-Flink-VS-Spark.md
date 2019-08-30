---
layout: post
title:  "Flink-VS-Spark"
date:   2018-12-16 22:58:12 +0800
tags:
      - Flink
---

本文基于Spark最新2.4版本及Flink最新1.6，从生态圈，部署模式，架构原理，基础API，流处理等方面对比二者相似及不同之处，由于笔者水平限制，不当之处，敬请批评指正。

* Spark和Flink均出自世界顶尖大学实验室
* 受MR启发，其理念，架构又比MR更进一步
* 均可提供基于批量数据的处理和基于流数据的处理
* Apache社区顶级明星项目
* 功能栈完备，生态圈均非常广泛
 
### 生态圈对比：

大数据领域一个项目的火热离不开相关的技术栈，Spark和Flink基于对底层数据和计算调度的高度抽象的内核（Core）开发出了批处理，流处理，结构化数据，图数据，机器学习等不同套件，完成对绝大多数数据分析领域的场景的支持，意欲一统大数据分析领域。统计作为计算引擎，也很好的支持了与周边大数据分析项目的兼容，

####  功能栈对比
功能  | Flink |Spark
---|---|----- | 
批处理 | Dataset API |RDD/Dataset API| 
微批处理 |  | Dstream
流处理 | Dstream | Dataset
结构化分析（SQL） |  TABLE API /SQL | SQL
图数据分析 | Geely | GraphX
机器学习 | FlinkML | MLlib&SparkR

#### 语言实现与支持

* 实现语言

   Spark和Flink均有Scala/Java混合编程实现，Spark的核心逻辑由Scala完成，Flink的主要核心逻辑由Java完成
* 支持应用语言
    Flink主要支持Scala，和Java编程，部分API支持python应用
    Spark主要支持Scala，Java，Python,R语言编程，部分API暂不支持Python和R
    
#### 第三方项目集成


    Flink&Spark官方支持支持与存储系统如HDFS,S3集成，资源管理/调度Yarn，Mesos，K8s等集成，数据库Hbase,Cassandra,消息系统Amazon Kinesis,Kafka等
    
    Flink还官方支持ElasticSearch,Twitter,RabbitMQ,NiFi,S3等，
    Spark官方支持与Hive,Alluxio，在2.0版本之前也官方支持与Twitter，ZeroMQ等消息流，在2.0之后移除了相关支持。
    其他一些大数据官方/私人项目也都纷纷开发了与Spark的集成，如数据库Ignite，Redis，GemFire/Geode等

从生态圈来看，FLink&Spark都有活跃的社区支持，广泛的大数据项目集成，较好的易用性

### 部署模式对比：

部署模式 | Flink | Spark | 备注
---|--- | --- | ----|
LOCAL | 支持| 支持 |
Standalone | 支持| 支持 
Yarn|支持| 支持 |  Flink支持yarn-sessionb模式及yarn-cluster模式，Spark支持yarn-client,yarn-cluster模式
K8s | 支持 | 支持 | 
Mesos | 支持 | 支持 | 
Docker | 支持| | 1.2版本开始支持，镜像非Flink官方发布
EC2 | | 支持|

Spark 与 Flink部署较为复杂，架构图对比可参考 [Flink VS Spark 部署架构](https://www.jianshu.com/p/0f4725f1b7d8) 


除以上模式外，Flink官方也给出了在MapR，Google Compute Engine AWS等平台/服务中部署Flink的指导。关于部署方法：可参考官网

[Flink部署](https://ci.apache.org/projects/flink/flink-docs-release-1.6/ops/deployment/docker.html)

[Spark部署](http://spark.apache.org/docs/latest/cluster-overview.html)



### API对比

![Flink-Spark API.png](https://upload-images.jianshu.io/upload_images/9004616-9232cfa8354d5640.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 批处理：

Spark批处理的数据表示经历了从RDD -> DataFrame -> Dataset的变化，均具有不可变，lazy执行，可分区等特性，是Spark框架的核心，rdd经过map等函数操作后，并没有改变而是生成新的RDD，Spark的Dataset（DataFrame是一种特殊的Dataset，已经不推荐使用）还包含数据类型信息

Flink批处理的API是Dataset,同样具有不可变，lazy执行，可分区等特性，是Flink框架的核心，Dataset经过map等函数操作后，并没有改变而是生成新的Dataset


#### 流处理

* SparkStreaming
 
    Spark在1.*版本引入的sparkstreaming作为流处理模块，抽象出Dstream的API来进行流数据处理，同时抽象出通过receiver获取消息数据，然后启动task处理的模式，以及直接启动task消费处理两种方式的流式数据处理。receiver模式由于稳定性不足被遗弃，推荐使用的是直接消费模式；然而本质上讲，Sparkstreaming的流处理是micro-batch的处理模式，将一定时间的流数据作为一个block/RDD，然后使用批处理的rdd的api来完成数据的处理。

* Structed streaming
 
    随着Spark在2.*版本的Structed streaming的推出，Sparkstreaming模块进入了维护模式，从Spark2.*版本以来没有已经没有更新，当前社区主推使用Structedstreaming进行流处理。Structed streaming在流处理中有两种流处理模式，一种是microbatch模式；一种是continuous模式；

        microbatch模式与sparkstreaming的microbatch模式大致相当，分批处理消息，但可通过设置连续的批次处理，即一个批次执行完之后立即进入下一个批次的处理
        
        continuous模式，可以实现真正的流数据处理，端到端的毫秒级，当前处于Experiment状态，也只能支持简单的map,filter操作，当前不支持聚合，current_timestamp，current_date等操作
            
        PS : microbatch <----> continuous 两种模式可以相互切换且无需改动代码

* Flink Streaming

    Flink Streaming以流的方式处理流数据，可以实现简单map,fliter等操作，也可以实现复杂的聚合，关联操作，以完善的处理模型及high throughout得到了广泛的应用。
    
    
        
* 三者对比

 window 支持性对比 

对于流数据来说，数据具有实时产生，无限多的特性，无法向批数据那样对数据进行统计分析，因此产生了在一定时间内进行统计分析的功能，所谓的一定时间在流中抽象成window表示，window的startTime表示统计的开始时间，endTime表示统计的结束时间，在该时间内可以对接收到的数据进行统计分析。


Window 类型 | Window 含义 | Flink Streaming| SparkStreaming | Structed Streaming | 备注
---|---|----|----|----|---
tumblingWindow | 一个滚动的window | 支持 | 支持 | 支持 | 
Sliding window | 滑动的window | 支持 | 支持  | 支持 | 
Global window |  全局window | 支持 | 间接实现 | 间接支持 | 间接支持的含义是可以时间类似功能，但没有抽象出该window
Session window | 以接收到数据开始，一定时间没有接收到数据，则结束 | 支持 | 不支持 | 不支持  





流join分析：

由于Sparkstreaming中不支持eventtime的概念，其只能支持window不同Dstream的RDD的join，不同window间无法join

模块 | event-time | 流join | join实现方式 | 处理方式 | 备注 |
------------|----- | ----- | ---- | ------ | -----
Sparkstreaming | 不支持 | 支持 | window内 | processingTime | micro-batch处理
FLink1.5之前 | 支持 | 支持 | window内 | native处理，join时(window触发)，watermark灵活| Processing Time／ EventTime ／ element Number| 
FLink1.6之后 | 支持 | 支持 | window内，跨window | native处理，join时(window触发)，watermark灵活| Processing Time／ EventTime ／ element Number| 
Structed Streaming 2.2 |支持 | 不支持 | 仅支持流数据和静态数据的join | native处理，join时(window触发)，watermark灵活| Processing Time／ EventTime |
Structed Streaming 2.3+ |支持 | 支持 | 跨window | native处理，join时（proocessingTime（interval）触发）|  Processing Time／ EventTime |

PS: 
* Flink／structed streaming开发难度相当，FLink略复杂，但灵活度更高
* Flink的inteval join
* StructedStreaming支持数据去重（同个imsi的数据的多个不同join结果的去重）
* FLink的窗口操作相当于structedstreaming的update模式
* Flink的单流的watermark更新时实时的，有专门线程处理
* Structedstreaming的watermark更新时间基于批的，每个批次共用同一个watermark，如果有多个流，多个流共用一个watermark
* structedStreaming的watermark更新方法：
        基于每个流找出该流的watermark：Max_event_time - lateness
        找出所有流中最小/最大的watermark设置为batch的watermark
* Flink专门抽象了类以便不同场景下使用自定义的eventTime的waterMark获取/设置方法,且提供了一般场景下的的类以便使用
* Flink抽象了trigger和evictor来实现触发计算和清理数据的逻辑，以便自定义相关逻辑 
* FLink 支持sideoutput输出，如迟到的数据可以单独输出
