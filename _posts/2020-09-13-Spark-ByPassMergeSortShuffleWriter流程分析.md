---
layout: post
title:  "Spark ByPassMergeSortShuffleHandler流程分析"
date:   2020-09-13 12:30:21 +0800
tags:
      - Spark
---

Spark的shuffleWriter一共有三种，本文分析 ByPassMergeSortShuffleWriter的shuffle写数据过程

从使用场景来看，ByPassMergeSortShuffleWriter主要使用在在ShuffleMapTask侧没有`预聚合`的场景，且resultTask的个数少于阈值（默认200，由spark.shuffle.sort.bypassMergeThreshold控制）的场景。
因此其实现较为简单，分为如下几步：

1. 为每个分区（基于resultTask个数）生成一个DiskBlockObjectWriter
2. 针对每条数据，根据其key计算并找出其应当写入的分区，并调用对应的write直接写出数据
3. 针对每个分区对应的文件，按照分区号依次读取并写入总的数据文件（其中涉及到是否使用Zero copy技术）
4. 同时将每个分区数据的offset写入index文件


数据文件的写入流程如下：

![流程](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCEeb5b415588dd2c0c4f14fa46b62de8bd/9694)


核心代码如下：

```
public void write(Iterator<Product2<K, V>> records) throws IOException {
    ...
    //1. 为每个要输出的分区生成一个partitionWriter
    partitionWriters = new DiskBlockObjectWriter[numPartitions];
    partitionWriterSegments = new FileSegment[numPartitions];
    for (int i = 0; i < numPartitions; i++) {
      final Tuple2<TempShuffleBlockId, File> tempShuffleBlockIdPlusFile =
        blockManager.diskBlockManager().createTempShuffleBlock();
      final File file = tempShuffleBlockIdPlusFile._2();
      final BlockId blockId = tempShuffleBlockIdPlusFile._1();
      partitionWriters[i] =
        blockManager.getDiskWriter(blockId, file, serInstance, fileBufferSize, writeMetrics);
    }
    //2. 针对每条记录的key，计算出其分区找到其writer对数据进行写入
    while (records.hasNext()) {
      final Product2<K, V> record = records.next();
      final K key = record._1();
      partitionWriters[partitioner.getPartition(key)].write(key, record._2());
    }
    // 关闭并提交writer
    for (int i = 0; i < numPartitions; i++) {
      final DiskBlockObjectWriter writer = partitionWriters[i];
      partitionWriterSegments[i] = writer.commitAndGet();
      writer.close();
    }

    File output = shuffleBlockResolver.getDataFile(shuffleId, mapId);
    File tmp = Utils.tempFileWith(output);
    try {
    // 3. 将所有写出的文件都copy至总的文件中，并返回每个分区的长度
      partitionLengths = writePartitionedFile(tmp);
      //4. 根据每个分区长度，写index文件
      shuffleBlockResolver.writeIndexFileAndCommit(shuffleId, mapId, partitionLengths, tmp);
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logger.error("Error while deleting temp file {}", tmp.getAbsolutePath());
      }
    }
```

查看第三步中写文件代码逻辑如下，如果transferToEnabled为true，将通过Zero copy的方式提升分区文件合并的性能

```
private long[] writePartitionedFile(File outputFile) throws IOException {
    //..m.nanoTime();
    try {
      for (int i = 0; i < numPartitions; i++) {
        final File file = partitionWriterSegments[i].file();
            // 将每个partition对应的文件流'in' 写入out
            lengths[i] = Utils.copyStream(in, out, false, transferToEnabled);
            copyThrewException = false;
   // 返回每个partittion的长度
    return lengths;
  }
```
