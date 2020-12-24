---
layout: post
title:  "Spark Shufflewrite分析"
date:   2020-09-13 19:20:12 +0800
tags:
      - Spark
---

分布式计算的shuffle操作通常是分布式应用计算性能的瓶颈点，因此一个好的shuffle实现（shuffle write和shuffle read）对于分布式计算引擎的性能起着至关重要的作用。

最新的Spark的shuffleWriter一共有三种（原有的Hash-Based Shuffle已经被删除）,分别对应不同的场景。这三种write分别是:

[UnsafeShuffleWriter](https://cyq89051127.github.io/2020/09/12/Spark-UnsafeShuffleWriter%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

[BypassMergeSortShuffleWriter](https://cyq89051127.github.io/2020/09/13/Spark-ByPassMergeSortShuffleWriter%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

[SortShuffleWriter](https://cyq89051127.github.io/2020/09/13/Spark-SortShuffleWriter%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

然后这三种write分别使用什么场景，spark又是如何实现shufflewrite的设定

1. 在driver端注册一个shuffle时，根据不同的场景，得到不同的shuffleHandler

![](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCEf46ad822360de2a452d0db2cd1c5fb7f/9698)

```
override def registerShuffle[K, V, C](
      shuffleId: Int,
      numMaps: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
      // If there are fewer than spark.shuffle.sort.bypassMergeThreshold partitions and we don't
      // need map-side aggregation, then write numPartitions files directly and just concatenate
      // them at the end. This avoids doing serialization and deserialization twice to merge
      // together the spilled files, which would happen with the normal code path. The downside is
      // having multiple files open at a time and thus more memory allocated to buffers.
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
      new SerializedShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // Otherwise, buffer map outputs in a deserialized form:
      new BaseShuffleHandle(shuffleId, numMaps, dependency)
    }
  }

```

2. 在executor端，进行数据shuffle写的时候，获取对应的writer，完成数据的写操作

handler和writer的对应关系如下

| Handler    | Writer |
| ---------|---- |
| SerializedShuffleHandle  | UnsafeShuffleWriter            |
| BypassMergeSortShuffleHandle |BypassMergeSortShuffleWriter| 
| BaseShuffleHandle | SortShuffleWriter |

三种handler写数据的异同点：

相同点：
 
    1. 在写数据时， 都有先将部分数据先落盘的流程
    2. 在数据处理完毕后，会将之前落盘的数据（可能也包括当前内存中未落盘的数据）进行读取，merge后落盘
    3 最终每个shuffleMapTask写出的文件都包含一个data文件和一个index文件
    4. 每个data文件中，都会根据partitionId进行排序
    5. 最终落盘数据（非index文件）只包含记录的key，value，不会包含每条记录的partitionId
    
差异：

    * BypassMergeSortShuffleWriter：写盘首先是针对每个分区写一个文件，不涉及内存空间申请及spill
    * SortShuffleWriter： spill出的文件，分区有序， 如果有ordering或者aggregator时，也会在分区内对key进行排序
    * UnsafeShuffleWriter： 在申请内存时，可能申请堆外内存；分区有序，但分区内无序
    
    