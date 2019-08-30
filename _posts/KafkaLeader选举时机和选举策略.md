---
layout: post
title:  "KafkaLeader选举时机和选举策略"
date:   2018-04-26 12:19:12 +0800
tags:
      - Kafka
---
## Kakfa的集中leader选举过程


* Kafka是分布式的消息分发系统，通过引入topic的概念来区分消息类型，引入partition的概念来增加消息的吞吐量，引入replica的概念来提高消息的可靠性。在多replica的场景下，消息的读写都是通过leader来完成，其他replica则是通过从leader读取数据来完成消息的同步以保证leader异常时消息的完善性。 然而在多个replica的场景下，谁能成为leader？ leader异常后，又如何选出新的leader呢这是kafka提供可靠服务的关键所在。下面就对此作简要分析。

### Leader选举器

Kafka通过leaderSelector完成leader的选举。

可能触发为partition选举leader的场景有: 新创建topic，broker启动，broker停止，controller选举，客户端触发，reblance等等 场景。在不同的场景下选举方法不尽相同。Kafka提供了五种leader选举方式，继承PartitionLeaderSelector，实现selectLeader方法完成leader的选举， 下面对这五种leader选举方式给予说明：

#### NoOpLeaderSelector
    该选举器不会调用到，大概是拿来测试用的。 该选举器直接返回当前的leader Irs，AR。

#### OfflinePartitionLeaderSelector
    触发场景：
        * 新创建topic
        * PartitionStateMachine启动
        * broker启动时
        * ReplicaStateMachine检测到broker的znode“被删除”
    选举：
        1） Isr列表中有存货的replica，直接选出
        2） 否则，unclean.leader.election.enable 为false，抛出异常
        3） 存活的ar中有replica，选出，否则抛出异常

#### ControlledShutdownLeaderSelector
    触发场景：
        * kafka的broker进程政策退出发送消息给controller，controller触发
    选举：
        * 在isr列表中的选出存活的replica，否则抛出异常
     
#### PreferredReplicaPartitionLeaderSelector
    触发场景：
        * znode节点/admin/preferred_replica_election写入相关数据
        * partition-rebalance-thread线程进行触发reblance时
        * 新产生controller
    选举 ：
        1） AR中取出一个作为leader，如果与原有leader一样，抛出异常
        2） 新leade的replica的broker存活且replica在isr中，选出，否则抛出异常
        
#### ReassignedPartitionLeaderSelector
    触发场景:
        * znode节点LeaderAndIsr发生变化
        * Broker启动时
        * zknode节点/admin/reassign_partitions变动
        * 新产生controller时
    选举：
        * 新设置的ar中，存在broker存活的replica且replica在isr中则选出为leader，否则抛出异常

PS： 在所有的leader选举策略中，如果符合条件的replica有多个，如Seq[int]，则使用的是Seq.head，取的是seq的第一个

Kafka的leader选举过程逻辑较为简单，参考kafka.controller.PartitionLeaderSelector.scala即可
