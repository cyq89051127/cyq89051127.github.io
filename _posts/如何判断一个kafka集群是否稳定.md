---
layout: post
title:  "如何判断一个kafka集群是否稳定"
date:   2019-01-05 23:39:12 +0800
tags:
      - Kafka
---

在流作业的生产环境中，作为应用最广泛的消息中间件，kafka集群的稳定性对业务的平稳起到重要作用。然而如何判断一个kafka集群的稳定性是一个运维人员的重要技能。

笔者结合经验总结了如下查看一个kafka不稳定状态下可能出现的现象：

* 应用运行过程中经常性发生leader找不到异常，如“LEADER_NOT_AVAILABLE，NOT_LEADER_FOR_PARTITION”等异常
* 应用运行过程中消费消息时抛出无法poll到数据的异常，如sparkstreaming应用抛出“assertion failed: Failed to get records for (...) after polling for 512 ms”的异常
* 消费过程中出现 类似offsets out of range异常日志
* 客户端与kafka交互式“卡死”的异常，如Sparkstreaming在JobGenerator的线程中无限“卡死”，类似堆栈如下：
   ![image.png](https://upload-images.jianshu.io/upload_images/9004616-3423cf2dd4b9d47b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



如果应用层看到如上类似异常，则需要考虑排查kafka集群是否存在性能/不稳定问题，排查方式可参考如下：
    
1. Kafka的broker进程所在节点的load值（该值越小，节点约稳定），改值应低于节点的processors（通过cat /proc/cpuinfo | grep processor查看），如果load较高，则需要考虑降低该值
2. kafka进程的gc情况（jstat -gcutils $kafka_pid 1000）, 观察FGC列（full gc次数）以及FGCT列（full gc总耗时），如果这两列的值增大较快，则需要考虑调整broker进程GC，否则可能出现无法访问的情况
3. 查看kafka进程中各线程运行的cpu消耗(top -H -p $broker_pid),如果存在部分线程cpu利用率居高不下（长期在90%），则需要查看分析对应线程[（找出进程中消耗CPU较多的线程的方法）](https://www.jianshu.com/p/eabd24d67e56)，如果对应的线程均在同一个线程池（如kafka-request-handler线程池），则需要考虑调整相关线程池的线程数（如num.io.threads）
4. 通过kafka-topic.sh --describe --topic命令查看部分topic的信息，在多副本的topic中，如果存在部分partition的ISR列表中的副本数 < 设置的replication数，则需要分析并调整相关参数（如num.replica.fetchers或num.io.threads）
  ![image.png](https://upload-images.jianshu.io/upload_images/9004616-43a2cebd20de1112.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


5. 查看coordinate日志，是否频繁打印shrinking,expanding相关的日志，如果存在，则同样表示集群当前处于不稳定状态
  ![image.png](https://upload-images.jianshu.io/upload_images/9004616-497bf207c2d3e279.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    

  
