---
layout: post
title:  "Spark SortShuffleWriter流程分析"
date:   2020-09-13 17:25:12 +0800
tags:
      - Spark
---

Spark的shuffleWriter一共有三种，本文分析 SortShuffleWriter的shuffle写数据过程. SortShuffleWriter是最为复杂的shuffle writer。 在ShuffleMapTask中需要对数据分区内进行排序或者预聚合的场景下，都是使用该writer完成shuffle数据的写盘。

其核心流程分为如下几步：

1. 在ExternalSorter中插入数据
2. 将每个spill文件读取合并重新生成新的数据文件，在合并的过程中，如果有预聚合或者排序的操作，也会进行相关操作
3. 生成数据文件对应的index文件。

以上步骤中最为复杂的即是将数据插入externalSorter中，本文重点分析此处逻辑。

1. 首先将数据插入array中

```
  def insertAll(records: Iterator[Product2[K, V]]): Unit = {
    // TODO: stop combining if we find that the reduction factor isn't high
    val shouldCombine = aggregator.isDefined
    if (shouldCombine) {
      // Combine values in-memory first using our AppendOnlyMap
      val mergeValue = aggregator.get.mergeValue
      val createCombiner = aggregator.get.createCombiner
      var kv: Product2[K, V] = null
      val update = (hadValue: Boolean, oldValue: C) => {
        if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
      }
      while (records.hasNext) {
        addElementsRead()
        kv = records.next()
        // 针对需要预聚合的场景，通过一个PartitionedAppendOnlyMap完成数据插入及聚合，其本质也是将数据存入array中
        map.changeValue((getPartition(kv._1), kv._1), update)
        maybeSpillCollection(usingMap = true)
      }
    } else {
      // Stick values into our buffer
      while (records.hasNext) {
        addElementsRead()
        val kv = records.next()
        // 数据插入PartitionedPairBuffer中，本质是将数据存入Array中，实现较为简单
        buffer.insert(getPartition(kv._1), kv._1, kv._2.asInstanceOf[C])
        maybeSpillCollection(usingMap = false)
      }
    }
  }
```

本质上来讲，是将数据放入一个array中，对于输入的第n条记录，:
 
 #### 包含map端预聚合的数据写入
  
  在有预聚合的场景下，核心逻辑是： 数据写入时，需要判断是否之前已经存在该key的记录，如果存在，则找出并进行聚合，如果不存在，则直接写入该记录。
    `由于涉及到查询之前是否已经存在该key，因此使用了hash值，这也是针对该场景使用PartitionedAppendOnlyMap的原因`
    
   数据插入的核心流程如下图所示：
   ![数据插入](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/603511E8E7A3445BA6C12E98C58098AD/9696) 
  数据插入的代码逻辑如下：
  
  ```
   def changeValue(key: K, updateFunc: (Boolean, V) => V): V = {
      ... 空值处理
      var pos = rehash(k.hashCode) & mask
      var i = 1
      while (true) {
        val curKey = data(2 * pos)
        
        if (curKey.eq(null)) {
        // 当前位置存储数据为空，即未与数据存储
          val newValue = updateFunc(false, null.asInstanceOf[V])
          data(2 * pos) = k
          data(2 * pos + 1) = newValue.asInstanceOf[AnyRef]
          incrementSize()
          return newValue
        } else if (k.eq(curKey) || k.equals(curKey)) {
           // 当前位置有值，且已存储值与当前值相等
          val newValue = updateFunc(true, data(2 * pos + 1).asInstanceOf[V])
          data(2 * pos + 1) = newValue.asInstanceOf[AnyRef]
          return newValue
        } else {
         // 当前位置有值，且已存储值与当前值不等，则继续查找下一个可用存储位置
          val delta = i
          pos = (pos + delta) & mask
          i += 1
        }
      }
      null.asInstanceOf[V] // Never reached but needed to keep compiler happy
    }
  ```
  
  
 
 ### 不需要map端预聚合的数据写入场景
  该实现简单
    
    def insert(partition: Int, key: K, value: V): Unit = {
        if (curSize == capacity) {
        //如果array空间不足，则直接扩容，并将原有数组中数据copy至新数据即可
          growArray()
        }
         //  在位置array[2n]存入`(partition, key.asInstanceOf[AnyRef])`
        data(2 * curSize) = (partition, key.asInstanceOf[AnyRef])
        //  在位置array[2n+1]的位置存入`value.asInstanceOf[AnyRef]`，存入
        data(2 * curSize + 1) = value.asInstanceOf[AnyRef]
        curSize += 1
        afterUpdate()
      }
   
2. 将当前内存中数据spill至磁盘

每插入32条记录，则查看当前的内存使用量是否已经超过申请到的可用内存大小，如果超过则再次进行内存申请，如果此时再次申请到的内存依然无法满足使用，则触发spill落盘操作。

在每次触发spill落盘时，会对array中的数据进行排序落盘（在真正落盘时，只会将key和value写入磁盘，partitionId不会落盘），排序的规则如下：

1. 首先根据分区对记录进行排序

```
  /**
   * A comparator for (Int, K) pairs that orders them both by their partition ID and a key ordering.
   */
  def partitionKeyComparator[K](keyComparator: Comparator[K]): Comparator[(Int, K)] = {
    new Comparator[(Int, K)] {
      override def compare(a: (Int, K), b: (Int, K)): Int = {
        val partitionDiff = a._1 - b._1
        if (partitionDiff != 0) {
          partitionDiff
        } else {
          keyComparator.compare(a._2, b._2)
        }
      }
    }
  }
}
```
2. 再在同一分区内，如果有定义ordering或者aggregator则会根据如下keyComparator对同一分区的记录进行排序

```
private val keyComparator: Comparator[K] = ordering.getOrElse(new Comparator[K] {
    override def compare(a: K, b: K): Int = {
      val h1 = if (a == null) 0 else a.hashCode()
      val h2 = if (b == null) 0 else b.hashCode()
      if (h1 < h2) -1 else if (h1 == h2) 0 else 1
    }
  })

  private def comparator: Option[Comparator[K]] = {
    if (ordering.isDefined || aggregator.isDefined) {
      Some(keyComparator)
    } else {
      None
    }
  }
```

3. 在所有的数据都处理完之后，会将所有spill至磁盘及内存中array中的数据merge到同一个文件中，并生产index文件

和以上的数据spill值磁盘一样，在归并时，也会根据是否有ordering，aggregator等场景，确认各个spill文件归并时是否需要分区内有序以及书否需要merge。 经过归并可以得到一个保存所有shuffle数据有序文件。
