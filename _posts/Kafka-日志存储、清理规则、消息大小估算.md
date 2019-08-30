---
layout: post
title:  "Kafka-日志存储、清理规则、消息大小估算"
date:   2019-03-21 10:04:12 +0800
tags:
      - Kafka
---

### kafka的日志：

kafka消息存储在kafka集群中（分parition存储，每个partition对应一个目录。目录名为${topicName}-${partitionId}，kafka接收到的消息存放于此目录下，包含log文件，index文件，timeindex索引文件（0.10.1后的版本）

![image](http://upload-images.jianshu.io/upload_images/9004616-8565d9cd7da99a4f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

名字 | 含义 | 备注
---|--- |-----
00000000000009475939  | 文件中第一条消息的offset
*.log  | 存储消息实体的文件
*.index | 记录消息的offset以及消息在log文件中的position的索引 | 稀疏存储
*.timeindex | 记录消息的timestamp和offset的索引 | 稀疏存储

### kafka消息查看

使用kafka-run-class工具调用kafka.tools.DumpLogSegments,查看kafka消息落盘后信息。 如下 ： 
        
    /usr/hdp/current/kafka-broker/bin/kafka-run-class.sh kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files ****.log(index,timeindex)
    
如：

![image](http://upload-images.jianshu.io/upload_images/9004616-8da7322398d74ecd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 日志/消息清理（delete）
kafka消息日志的清理逻辑是启动线程定期扫描日志文件，将符合清理规则的消息日志文件删除。

* 清理规则有两种：


    基于日志量大小的清理：当消息日志总量大于设定的最大消息日志阈值时，删除老旧日志以维持消息日志总量小于设定的阈值
    基于日志修改时间的清理：time.millSeconds - _.lastModified > log.config.retentionMs ，清理该日志文件

* 清理： 

 
    给文件加上后缀名.delete
    异步删除，等待一定时间后，将文件清理
    清理时，会将统一名称的日志和索引文件同时清理。
   
    
日志清理主要参数  

线程  | 参数/名称 | 默认值
---|---|---
线程  | kafka-log-retention
检测周期 |  log.retention.check.interval.ms | 5 * 60 * 1000L 
保留时间阈值 | retention.ms | 7 * 24 * 60 * 60 * 1000L
日志量阈值大小 | retention.bytes | -1 
kafka单个日志文件大小 | log.segment.bytes | 1024 * 1024 * 1024L
待删除文件异步删除，等待时间 | file.delete.delay.ms | 60000

由上图可知，kafka默认的清理策略是基于文件修改时间戳的清理策略，默认会保留七天的消息日志量，基于消息日志总量大小的清理规则不生效。

**在磁盘总量不足，消息量浮动较大的场景下并非最佳的日志清理策略（可能撑爆磁盘），在该场景下，可以考虑使用基于消息日志总量的清理策略。然后如何估算kafka消息的磁盘占用呢？**


### kafka消息大小估算：

发送一条消息（uncompressed） ： 

    消息如下： 
        ab,1552981106583,testInput_20,ab_minus,1552981126583
    在Log日志中：
        offset: 9475167 position: 8694 CreateTime: 1552981126583 isvalid: true payloadsize: 52 magic: 1 compresscodec: NoCompressionCodec crc: 3704994927 keysize: 9 key: Message_3 payload: ab,1552981106583,testInput_20,ab_minus,1552981126583
    占用空间：
        110条消息占用磁盘10206byte，单条消息约0.09k

如果是压缩格式的消息，可能不同的压缩算法，不同的消息格式有较大差别，需要实测估算
        
PS ： 在存在多replica的常见下，还需要在此次评估基础上乘以replica的副本数
    
    
    
