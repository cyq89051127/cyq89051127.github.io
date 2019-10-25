---
layout: post
title:  "Flink-Kafka-Exactly-Once 测试"
date:   2019-10-25 17:20:12 +0800
tags:
      - Flink
---

Flink 从kafka读消息处理后写入kafka样例 测试



## 测试环境

| 组件         | 版本  |
| :----------- | ----- |
| Flink        | 1.8.0 |
| Kafka client | 2.0.1 |
| kakfa Server | 2.3.0 |

### 测试步骤

1. 编写Flink程序，

   1. 程序一 ：实现从kafka的in主题读消息，处理并写入kafka out 主题
   2. 程序二：从kafka的out主题中消费消息
   3. 程序三：从kafka的out主题中消费消息，只消费commit过的消息

2. 创建kafka中相关topic

3. 分别启动三个程序

4. 往kafka的in主题中发送消息，观察两个从kafka中消费的应用的输出情况

   1. 使用console producer 往in主题中发送如下消息：

      > aa,bb
      > Cc,dd

   2. 查看程序二和三的输出情况

      程序二输出：

      > Reading uncommitted messages , `now time is Fri Oct 25 15:12:19 CST 2019 ` value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 0, CreateTime = 1571987538865, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(aa,bb))
      > Reading uncommitted messages , `now time is Fri Oct 25 15:12:23 CST 2019`  value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 1, CreateTime = 1571987543269, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(cc,dd))

      程序三过一段时间(2分钟之内)后才会输出：

      > Reading committed messages , `now time is Fri Oct 25 15:13:25 CST 2019 ` value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 0, CreateTime = 1571987538865, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(aa,bb))
      > Reading committed messages , `now time is Fri Oct 25 15:13:25 CST 2019`  value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 1, CreateTime = 1571987543269, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(cc,dd))

      查看输出的时间为`now time is Fri Oct 25 15:13:25 CST 2019`：

      查看程序一的日志可以看到，这个时间点，程序进行了commit操作。

      > `2019-10-25 15:13:25,231` DEBUG [org.apache.flink.streaming.api.functions.sink.TwoPhaseCommitSinkFunction] - FlinkKafkaProducer 0/1 - committed checkpoint transaction......

   3. 使用console producer 往in主题中发送如下消息：

      > xx,yy
      > zz,aa
      > cc,dd
      > mm,nn

   4. 观察程序二和程序三的输出情况
      程序二输出如下：

      > Reading uncommitted messages , `now time is Fri Oct 25 15:13:44 CST 2019`  value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 3, CreateTime = 1571987624830, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(xx,yy))
      > Reading uncommitted messages , `now time is Fri Oct 25 15:13:47 CST 2019`  value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 4, CreateTime = 1571987627633, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(zz,aa))
      > Reading uncommitted messages , `now time is Fri Oct 25 15:14:27 CST 2019`  value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 5, CreateTime = 1571987644627, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(cc,dd))
      > Reading uncommitted messages , `now time is Fri Oct 25 15:14:28 CST 2019`  value is 
      >  	ConsumerRecord(topic = out, partition = 0, offset = 6, CreateTime = 1571987668194, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(mm,nn))

      程序三在短时间内，未能输出

5. 停掉程序一（使用kill -9）命令，过一段时间后，重新启动程序一，观察程序二和程序三的输出

   程序二的输出：

   > Reading uncommitted messages , `now time is Fri Oct 25 15:16:12 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 8, CreateTime = 1571987772686, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(xx,yy))
   > Reading uncommitted messages ,` now time is Fri Oct 25 15:16:12 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 9, CreateTime = 1571987772694, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(zz,aa))
   > Reading uncommitted messages , `now time is Fri Oct 25 15:16:12 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 10, CreateTime = 1571987772695, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(cc,dd))
   > Reading uncommitted messages , `now time is Fri Oct 25 15:16:12 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 11, CreateTime = 1571987772695, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(mm,nn))

   程序三的输出：

   > Reading committed messages , `now time is Fri Oct 25 15:17:14 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 8, CreateTime = 1571987772686, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(xx,yy))
   > Reading committed messages , `now time is Fri Oct 25 15:17:14 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 9, CreateTime = 1571987772694, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(zz,aa))
   > Reading committed messages , `now time is Fri Oct 25 15:17:14 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 10, CreateTime = 1571987772695, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(cc,dd))
   > Reading committed messages , `now time is Fri Oct 25 15:17:14 CST 2019`  value is 
   >  	ConsumerRecord(topic = out, partition = 0, offset = 11, CreateTime = 1571987772695, serialized key size = -1, serialized value size = 12, headers = RecordHeaders(headers = [], isReadOnly = false), key = null, value = EVENT(mm,nn))

    

6. 查看out主题的log日志

   > baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 9041 producerEpoch: 698 partitionLeaderEpoch: 0 `isTransactional: true isControl: false` position: 0 CreateTime: 1571987538865 size: 80 magic: 2 compresscodec: NONE crc: 949199944 isvalid: true
   > | offset: 0 CreateTime: 1571987538865 keysize: -1 valuesize: 12 sequence: 0 headerKeys: [] payload: EVENT(aa,bb)
   > baseOffset: 1 lastOffset: 1 count: 1 baseSequence: 1 lastSequence: 1 producerId: 9041 producerEpoch: 698 partitionLeaderEpoch: 0 `isTransactional: true isControl: false` position: 80 CreateTime: 1571987543269 size: 80 magic: 2 compresscodec: NONE crc: 2784371633 isvalid: true
   > | offset: 1 CreateTime: 1571987543269 keysize: -1 valuesize: 12 sequence: 1 headerKeys: [] payload: EVENT(cc,dd)
   > baseOffset: 2 lastOffset: 2 count: 1 baseSequence: -1 lastSequence: -1 producerId: 9041 producerEpoch: 698 partitionLeaderEpoch: 0` isTransactional: true isControl: true` position: 160 CreateTime: 1571987842040 size: 78 magic: 2 compresscodec: NONE crc: 2286899176 isvalid: true
   > | offset: 2 CreateTime: 1571987842040 keysize: 4 valuesize: 6 sequence: -1 headerKeys: [] `endTxnMarker: COMMIT` coordinatorEpoch: 0
   > baseOffset: 3 lastOffset: 3 count: 1 baseSequence: 0 lastSequence: 0 producerId: 9042 producerEpoch: 383 partitionLeaderEpoch: 0 `isTransactional: true isControl: false` position: 238 CreateTime: 1571987624830 size: 80 magic: 2 compresscodec: NONE crc: 2997257542 isvalid: true
   > | offset: 3 CreateTime: 1571987624830 keysize: -1 valuesize: 12 sequence: 0 headerKeys: [] payload: EVENT(xx,yy)
   > baseOffset: 4 lastOffset: 4 count: 1 baseSequence: 1 lastSequence: 1 producerId: 9042 producerEpoch: 383 partitionLeaderEpoch: 0` isTransactional: true isControl: false` position: 318 CreateTime: 1571987627633 size: 80 magic: 2 compresscodec: NONE crc: 3872892493 isvalid: true
   > | offset: 4 CreateTime: 1571987627633 keysize: -1 valuesize: 12 sequence: 1 headerKeys: [] payload: EVENT(zz,aa)
   > baseOffset: 5 lastOffset: 5 count: 1 baseSequence: 2 lastSequence: 2 producerId: 9042 producerEpoch: 383 partitionLeaderEpoch: 0` isTransactional: true isControl: false` position: 398 CreateTime: 1571987644627 size: 80 magic: 2 compresscodec: NONE crc: 1294856295 isvalid: true
   > | offset: 5 CreateTime: 1571987644627 keysize: -1 valuesize: 12 sequence: 2 headerKeys: [] payload: EVENT(cc,dd)
   > baseOffset: 6 lastOffset: 6 count: 1 baseSequence: 3 lastSequence: 3 producerId: 9042 producerEpoch: 383 partitionLeaderEpoch: 0` isTransactional: true isControl: false `position: 478 CreateTime: 1571987668194 size: 80 magic: 2 compresscodec: NONE crc: 1466989230 isvalid: true
   > | offset: 6 CreateTime: 1571987668194 keysize: -1 valuesize: 12 sequence: 3 headerKeys: [] payload: EVENT(mm,nn)
   > baseOffset: 7 lastOffset: 7 count: 1 baseSequence: -1 lastSequence: -1 producerId: 9042 producerEpoch: 384 partitionLeaderEpoch: 0 `isTransactional: true isControl: true `position: 558 CreateTime: 1571988002767 size: 78 magic: 2 compresscodec: NONE crc: 1414789916 isvalid: true
   > | offset: 7 CreateTime: 1571988002767 keysize: 4 valuesize: 6 sequence: -1 headerKeys: [] `endTxnMarker: ABORT` coordinatorEpoch: 0
   > baseOffset: 8 lastOffset: 11 count: 4 baseSequence: 0 lastSequence: 3 producerId: 9025 producerEpoch: 731 partitionLeaderEpoch: 0 `isTransactional: true isControl: false` position: 636 CreateTime: 1571987772695 size: 137 magic: 2 compresscodec: NONE crc: 2741031897 isvalid: true
   > | offset: 8 CreateTime: 1571987772686 keysize: -1 valuesize: 12 sequence: 0 headerKeys: [] payload: EVENT(xx,yy)
   > | offset: 9 CreateTime: 1571987772694 keysize: -1 valuesize: 12 sequence: 1 headerKeys: [] payload: EVENT(zz,aa)
   > | offset: 10 CreateTime: 1571987772695 keysize: -1 valuesize: 12 sequence: 2 headerKeys: [] payload: EVENT(cc,dd)
   > | offset: 11 CreateTime: 1571987772695 keysize: -1 valuesize: 12 sequence: 3 headerKeys: [] payload: EVENT(mm,nn)
   > baseOffset: 12 lastOffset: 12 count: 1 baseSequence: -1 lastSequence: -1 producerId: 9025 producerEpoch: 731 partitionLeaderEpoch: 0 `isTransactional: true isControl: true` position: 773 CreateTime: 1571988071061 size: 78 magic: 2 compresscodec: NONE crc: 320860475 isvalid: true
   > | offset: 12 CreateTime: 1571988071061 keysize: 4 valuesize: 6 sequence: -1 headerKeys: [] `endTxnMarker: COMMIT` coordinatorEpoch: 0



### 分析：

* 从4.1，4.2及6日志中的offset 0~1消息可以看出，在往in主题发送消息之后，程序一立刻对消息进行消费，处理并输出到out主题，此时消息处于uncommit状态，此时程序二可以完成读取；
* 从4.1，4.2及6日志中的offset 0~2消息可以看出，在0~1的offset的消息被commit之后（从offset 2的消息`endTxnMarker: COMMIT`可知），这两条消息对于程序三可见，及程序三可完成消息的消费
* 从4.3, 4.4, 5及6中的3-6中offset信息可知，在发送消息后，在没有程序一commit之前停掉程序一，由于此时对于out主题来说，offset 3~6的消息已经接收到，但是没有收到commit请求，因此程序二可以立刻消费，而程序三无法消费
* 从5,6的7~12的offset消息可知，重新启动程序一之后，程序一会先发送请求至kafka将之前的offset为3~6的的消息abort掉（从offset为7的消息是`endTxnMarker: ABORT`）; 程序从checkpoint恢复，由于之前已经从in主题消费的消息并没有完成checkpoint，因此程序一会再次消费3中的消息。当消费并处理发送至out主题后，消息对程序二可见，因此程序二可以消费该消息；等程序一再次进行checkpoint并成功后，可以看到offset为12的`endTxnMarker: COMMIT`消息，此时程序三可以消费该消息。
* 对于步骤三种发送的4条消息来说，程序一进行两次计算并输出，由于我们开启了Kafka的事务模式

### 结论

Flink结合Kafka的Exactly-Once并非真正意义的Exactly-Once，本质上指的是端到端Effective-Once





Flink应用程序如下：

> ```scala
> object PipelineExect {
>   def main(args: Array[String]): Unit = {
> 
>     val env = StreamExecutionEnvironment.getExecutionEnvironment
> 
>     env.setParallelism(1)
> 
>     env.enableCheckpointing(120000)
> 
>     env.setStateBackend(new FsStateBackend("file:/Users/cyq/IdeaProjects/FlinkTest/checkpointdata1", true))
> 
>     val properties = new Properties()
>     properties.setProperty("bootstrap.servers", "10.1.236.66:9092")
>     properties.setProperty("group.id", "exactly-once-test")
> 
>     val stream = env
>       .addSource(new FlinkKafkaConsumer[String]("in", new SimpleStringSchema(), properties))
> 
>     val eventSource = stream.map(record => {
>       val fields = StringUtils.split(record, ",")
>       EVENT(fields(0), fields(1))
>     })
> 
>     val props = new Properties
>     props.setProperty(BOOTSTRAP_SERVERS_CONFIG, "10.1.236.66:9092")
>     props.setProperty("transaction.timeout.ms", String.valueOf(10 * 60 * 1000))
> 
>     val myProducer = new FlinkKafkaProducer[String](
>       "out", // target topic
>       new KeyedSerializationSchemaWrapper[String](new SimpleStringSchema),
>       props,
>       FlinkKafkaProducer.Semantic.EXACTLY_ONCE
>     );
>     eventSource.map(r => r.toString).addSink(myProducer);
>     env.execute("Exactly-Once-Test")
>   }
> }
> ```

Kafka 消费程序（read所有消息）模式

> ```scala
> 
> object KafkaUncommitReader {
>   def main(args: Array[String]): Unit = {
>     val properties = new Properties()
>     properties.setProperty("bootstrap.servers", "10.1.236.66:9092")
>     properties.setProperty("group.id", "uncommit_read_test")
>     properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
>     properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
> 
>     val consumer = new KafkaConsumer[String, String](properties)
>     consumer.subscribe(util.Arrays.asList("out"))
>     while (true) {
>       val records = consumer.poll(Duration.ofMillis(100))
>       import scala.collection.JavaConversions._
>       for (record <- records) {
>         println("Reading uncommitted messages , now time is " + new Date() + "  value is \n \t" + record)
>       }
>     }
>   }
> }
> ```

Kafka 消费程序（read_committed）模式



> ```scala
> object KafkaCommitReader {
>   def main(args: Array[String]): Unit = {
>     val properties = new Properties()
>     properties.setProperty("bootstrap.servers", "10.1.236.66:9092")
>     properties.setProperty("group.id", "commit_read_test")
>     properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
>     properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
>     properties.setProperty("isolation.level", "read_committed")
> 
>     val consumer = new KafkaConsumer[String, String](properties)
>     consumer.subscribe(util.Arrays.asList("out"))
>     while (true) {
>       val records = consumer.poll(Duration.ofMillis(100))
>       import scala.collection.JavaConversions._
>       for (record <- records) {
>         println("Reading committed messages , now time is " + new Date() + "  value is \n \t" + record)
>       }
>     }
>   }
> }
> ```