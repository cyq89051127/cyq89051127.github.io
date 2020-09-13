---
layout: post
title:  "Spark UnsafeShuffleWriter流程分析"
date:   2020-09-12 17:25:12 +0800
tags:
      - Spark
---

Spark的UnsafeShuffleWriter是Tungsten-Project（[内存管理](https://www.bilibili.com/video/BV1kx411E7WW?from=search&seid=6945519803380330698)）引入的新的Shuffle Writer。
该writer在写数据到磁盘时，会将数据有序写入（`仅仅分区间有序，分区内无序`）。 本文主要介绍其写数据实现，并讨论其缓存友好设计及实现。以下介绍其相关实现：

## UnsafeShuffleWriter的实现


![写数据相关类](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE9dee39377690808e1fb07cd5f722859c/9689)

各函数类的职责：

    UnsafeShuffleWriter ： 数据写的入口，并将最后内存数据以及spill至磁盘的数据读取并以分区merge后再次写入磁盘
    ShuffleExternalSorter ：内存申请，管理，写入数据至内存页，代理ShuffleInMemorySorter的写入
    ShuffleInMemorySorter ： 将分区号及数据指针写入LongArray

其核心流程主要包括如下：
1. 将输入数据（record）遍历
2. 将`分区号`和`数据指针`写入ShuffleInMemorySorter的array:LongArray,序列化的数据写入page页
3. 如果其中内存不足时，会将序列化数据根据分区号spill至磁盘
4. 数据操作完毕后，将当前内存中的数据以及spill至磁盘的数据重新merge，按分区号排序写如磁盘。
 

### UnsafeShuffleWriter 遍历数据

核心代码逻辑如下：
```java
public void write(scala.collection.Iterator<Product2<K, V>> records) throws IOException {
    boolean success = false;
    try {
      while (records.hasNext()) {
          // 插入数据
        insertRecordIntoSorter(records.next());
      }
      // 将内存数据及spill之后的数据重新merge并写入磁盘，同时也生成index文件
      closeAndWriteOutput();
      success = true;
    } finally {
      ...
    }
  }
```

其中insertRecordIntoSorter实现如下：
```java
void insertRecordIntoSorter(Product2<K, V> record) throws IOException {
    assert(sorter != null);
    // 将key,value写入serBuffer
    final K key = record._1();
    final int partitionId = partitioner.getPartition(key);
    serBuffer.reset();
    serOutputStream.writeKey(key, OBJECT_CLASS_TAG);
    serOutputStream.writeValue(record._2(), OBJECT_CLASS_TAG);
    serOutputStream.flush();

    final int serializedRecordSize = serBuffer.size();
    // 调用ShuffleExternalSorter的insertRecord方法插入数据
    sorter.insertRecord(
      serBuffer.getBuf(), Platform.BYTE_ARRAY_OFFSET, serializedRecordSize, partitionId);
  }
```

调用ShuffleExternalSorter的insertRecord的方法存入数据及分区号，数据指针，主要包含以下步骤：

1. 如果空间不足，将触发spill落盘
2. 申请LongArray用于存储partitionId和指针
3. 当前page也不足以缓存数据时，申请新page页
4. 将数据存入缓存页
5. 缓存指针和partitionId

```java
public void insertRecord(Object recordBase, long recordOffset, int length, int partitionId)
    throws IOException {
    if (inMemSorter.numRecords() >= numElementsForSpillThreshold) {
      logger.info("Spilling data because number of spilledRecords crossed the threshold " +
        numElementsForSpillThreshold);
      // 1. 如果存储空间不足，则触发spill操作
      spill();
    }
    // 2. 申请LongArray用于存储partitionId和指针
    growPointerArrayIfNecessary();
    // Need 4 bytes to store the record length.
    final int required = length + 4;
    // 3. 为缓存数据 获取申请新page页
    acquireNewPageIfNecessary(required);

    final Object base = currentPage.getBaseObject();
    final long recordAddress = taskMemoryManager.encodePageNumberAndOffset(currentPage, pageCursor);
    Platform.putInt(base, pageCursor, length);
    pageCursor += 4;
    // 4. 将数据存入page中
    Platform.copyMemory(recordBase, recordOffset, base, pageCursor, length);
    pageCursor += length;
    // 5. 将数据指针和partitionId写入inMemSorter中的Array中
    inMemSorter.insertRecord(recordAddress, partitionId);
  }
``` 


### spill数据

核心调用关系：ShuffleExternalSorter.spill ->  ShuffleExternalSorter.writeSortedFile(false).  
其核心实现逻辑是：
1. 根据分区id对inMemSort中的数据LongArray的进行排序
2. 根据排序好的LongArrary中数据找出对应的真实数据对应的page页及在该页的位置，也就是找出真实数据
3. 将数据落盘

writeSortedFile 方法实现如下：

```java
private void writeSortedFile(boolean isLastFile) throws IOException {
    ...
    // This call performs the actual sort.
    // 将数据跟分区号排序
    final ShuffleInMemorySorter.ShuffleSorterIterator sortedRecords =
      inMemSorter.getSortedIterator();
    final Tuple2<TempShuffleBlockId, File> spilledFileInfo =
      blockManager.diskBlockManager().createTempShuffleBlock();
    final File file = spilledFileInfo._2();
    final TempShuffleBlockId blockId = spilledFileInfo._1();
    final SpillInfo spillInfo = new SpillInfo(numPartitions, file, blockId);

    final SerializerInstance ser = DummySerializerInstance.INSTANCE;

    final DiskBlockObjectWriter writer =
      blockManager.getDiskWriter(blockId, file, ser, fileBufferSizeBytes, writeMetricsToUse);

    int currentPartition = -1;
    while (sortedRecords.hasNext()) {
      sortedRecords.loadNext();
      // 计算数据分区号
      final int partition = sortedRecords.packedRecordPointer.getPartitionId();
      assert (partition >= currentPartition);
      if (partition != currentPartition) {
        // Switch to the new partition
        if (currentPartition != -1) {
          final FileSegment fileSegment = writer.commitAndGet();
          spillInfo.partitionLengths[currentPartition] = fileSegment.length();
        }
        currentPartition = partition;
      }
      //通过指针（page页及offset）计算数据地址
      final long recordPointer = sortedRecords.packedRecordPointer.getRecordPointer();
      final Object recordPage = taskMemoryManager.getPage(recordPointer);
      final long recordOffsetInPage = taskMemoryManager.getOffsetInPage(recordPointer);
      int dataRemaining = Platform.getInt(recordPage, recordOffsetInPage);
      long recordReadPosition = recordOffsetInPage + 4; // skip over record length
      // 将数据通过writer写出
      while (dataRemaining > 0) {
        final int toTransfer = Math.min(DISK_WRITE_BUFFER_SIZE, dataRemaining);
        Platform.copyMemory(
          recordPage, recordReadPosition, writeBuffer, Platform.BYTE_ARRAY_OFFSET, toTransfer);
        writer.write(writeBuffer, 0, toTransfer);
        recordReadPosition += toTransfer;
        dataRemaining -= toTransfer;
      }
      writer.recordWritten();
      ...
    }
```

### 为LongArray申请内存空间

当LongArray剩余空间不足以满足新增数据时，会对LongArary扩容，扩容会会将原有LongArray中的数据copy至新的Array中，并释放源有LongArray

```java
  private void growPointerArrayIfNecessary() throws IOException {
    assert(inMemSorter != null);
    if (!inMemSorter.hasSpaceForAnotherRecord()) {
      long used = inMemSorter.getMemoryUsage();
      LongArray array;
      try {
        // could trigger spilling  // 申请内存空间
        array = allocateArray(used / 8 * 2);
      } catch (OutOfMemoryError e) { //...
        }
        return;
      }
      // check if spilling is triggered or not
      if (inMemSorter.hasSpaceForAnotherRecord()) {
        freeArray(array);
      } else {
        // 完成扩容后的老数据迁移
        inMemSorter.expandPointerArray(array);
      }
    }
  }
```

### 申请内存页用户存放真实数据

如果currentPage为空或者currentPage不足以存放数据时，则重新申请缓存page，并将其设置为currentPage. 内存页的申请，我们后续单独讨论

```java
 private void acquireNewPageIfNecessary(int required) {
    if (currentPage == null ||
      pageCursor + required > currentPage.getBaseOffset() + currentPage.size() ) {
      // TODO: try to find space in previous pages
      currentPage = allocatePage(required);
      pageCursor = currentPage.getBaseOffset();
      allocatedPages.add(currentPage);
    }
  }
```

### 数据写入page页

1. 先将数据长度写入page页
2. 将真实数据写入page页

```java
    final Object base = currentPage.getBaseObject();
    // 先放入长度，占用4个字节
    Platform.putInt(base, pageCursor, length);
    pageCursor += 4;
    //放入真实的数据, 占用length个字段
    Platform.copyMemory(recordBase, recordOffset, base, pageCursor, length);
    pageCursor += length;
```

### 将分区id,page页数,page页的offset写入LongArray

调用inMemSorter.insertRecord(recordAddress, partitionId)方法完成，本质是放入LongArray数组中

```java
public void insertRecord(long recordPointer, int partitionId) {
    if (!hasSpaceForAnotherRecord()) {
      throw new IllegalStateException("There is no space for new record");
    }
    array.set(pos, PackedRecordPointer.packPointer(recordPointer, partitionId));
    pos++;
  }
```

## 缓存友好/感知及其实现

CPU从高速缓存中读取数据的效率与从内存中读取数据的效率相差较大。从高速缓存读取数据的性能是内存中读取数据的几十倍上百倍。缓存友好/感知的含义是指CPU访问数据时，更多/更高概率的从高速缓存中（L1，L2,L3级缓存效率依次递减）命中数据，此时CPU的数据读取性能更好。在大批量数据处理时，如果能够提升高速缓存命中的概率，则可以显著提升应用运行性能。

在Spark的Tungsten-Project的内存管理中就对提升数据的缓存做了优化，极大提升了在高速缓存中被命中的概率。通常情况下，计算机的缓存空间较小，如果将整条数据都进行缓存，则有缓存数据的记录数就很少。因此Spark采用只缓存记录的部分关键字段/信息的方式。

### 缓存友好的实现

Spark在内存中inMemSort中使用一个LongArray来保存每条每条记录的partitionId和该记录的指针，其中partitionId用于对记录进行分区排序，记录的指针用于指向数据的地址，通过指针可以访问到该数据。因此只需要在LongArray中放入一个long型数据，即可同时存放partitionId和`数据地址`

![LongArray设计](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCEf732c11c8b98c5f88883d7845f935a62/9691)
LongArray中存储的数据的结构如下：

    从上图可以看出，有24个bit位用于存放分区号，这也是为什么使用UnsafeShuffleWriter需要保证分区数小于2^24
    从上图可以看出page页使用13个bit位表示因此可以推算出 page页不能多于2^13
    从上图可以offset使用的是27个bit位，也就是单个page也不能存储超过其所能存储的数据

在将数据spill磁盘时，我们只需要对LongArray中的记录进行排序（每个记录中都包含该一个真实数据的partitionId和指向该数据的地址指针），有序（基于分区号有序）的LongArray的顺序访问即可保证读取的page也中的数据是有序的。

![基于分区的排序](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE51fe1ae331e05da3ec4fb800b7d6d285/9692)

有了上图中的设计之后，我们就可以做到：

    无序访问真实的数据，仅仅通过访问LongArray中的数据即可完成基于分区的排序

在高速缓存中不用缓存真实数据（一般远远大于LongArray中存储的一个long型数据），仅仅缓存LongArray（包括分区号，真实数据的地址指针），极大提升了高速缓存命中率，加速了排序过程

从以上分析可以看出，此处的实现结合了计算机硬件的设计，高效利用硬件，完成了程序的性能提升。如果没有该设计，那么在对数据spill的过程中，在数据量较大的情况下，由于无法对大量数据进行高速缓存，与以上方案相比，性能应当有明显的降低。

