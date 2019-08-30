---
layout: post
title:  "Kafka沟通机制"
date:   2018-02-27 12:19:12 +0800
tags:
      - Kafka
---
## Kafka不同进程“沟通机制”
Kafka服务的定位是一种高吞吐量的分布式消息订阅系统。服务再运行过程中，不同的进程（broker和controller之间，客户端和集群）之间需要进行“沟通”来保证功能的可用性。

Kafka主要通过两种方式进行“沟通”，以保证自身状态或需求被感知：

### 通过nio方式完成“沟通”

controller通过ControllerChannelManager发送消息到各broker，主要有ApiKeys.LEADER_AND_ISR, ApiKeys.STOP_REPLICA, ApiKeys.UPDATE_METADATA_KEY 三种消息。其中仅有ApiKeys.STOP_REPLICA有相关callback操作。broker通过KafkaRequestHandlerPool接收消息并调用kafkaApi进行消息相应。其他如producer发送消息，consumser消费消息，获取集群信息等操作，也是客户端通过nio的方式发送请求。Broker可处理的消息有如下：


消息 | 备注
------------|-----
ApiKeys.LEADER_AND_ISR | Controller 发送给broker
ApiKeys.STOP_REPLICA | Controller 发送给broker
ApiKeys.UPDATE_METADATA_KEY  | Controller 发送给broker
ApiKeys.PRODUCE | producer发送生产数据请求给broker
ApiKeys.FETCH |  consumer发送消费数据请求给broker
ApiKeys.LIST_OFFSETS |  客户端／consumer 发送查询offset请求
ApiKeys.METADATA | 获取集群topic信息
ApiKeys.CONTROLLED_SHUTDOWN_KEY  | brokershutdown时向controller发送消息
ApiKeys.OFFSET_COMMIT | consumer向coordinator发送的offset commit 请求
ApiKeys.OFFSET_FETCH |  consumer发送的请求partitions的当前的offset
ApiKeys.GROUP_COORDINATOR | consumer向broker发送group的coordinator是哪个节点的请求
ApiKeys.JOIN_GROUP | consumer向groupcoordinator发送 join group的请求
ApiKeys.HEARTBEAT | consumer定期发送heartbeat请求到groupcoordinator
ApiKeys.LEAVE_GROUP | consumer停止消费时发送退出group请求到groupcoordinator
ApiKeys.SYNC_GROUP | consumer变为leader获取follower时发送的已同步请求到groupcoordinator
ApiKeys.DESCRIBE_GROUPS | 客户端工具kafka-consumer-groups.sh ConsumerGroupCommand中使用describe group信息所用
ApiKeys.LIST_GROUPS | 客户端工具kafka-consumer-groups.sh ConsumerGroupCommand中使用list group信息所用 
ApiKeys.SASL_HANDSHAKE | nio的select发送消息/请求时，需要先发送sasl——handshake请求
ApiKeys.API_VERSIONS |  客户端发送的kafka版本请求

### 通过zookeeper注册监听器完成“沟通”
客户端（broker／client）如果需要将诉求（如topic创建，删除，reblance，partition reassign等）告知服务，所采取的方案是再ZooKeeper中创建目录，写入数据，服务端（此处指controller）通过listener监控到zk中的变化，并作出响应

#### Zookeeper的watch机制与第三方封装    
* Zookeeper是分布式应用程序协调服务。不同的进程通过约定的目录的写入和监控操作完成消息的互相感知。开源大数据多数服务的高可用性实现都是依靠zookeeper的协调功能完成。zookeeper客户端通过watcher机制完成对zookeeper的集群的监控。
    
* 由于zookeeper应用开发的复杂性，出现了第三方对zk的应用开发的封装。常用的有Curator FrameWork，I0Itec-zkclient。其中hive，hbase，spark等都是通过Curator FrameWork完成和zk的交互。Kafka则选用的是I0Itec-zkclient与zk交互。
 
#### Kafka中实现listener的方法：   
* Kafka通过zkUtils中调用org.I0Itec.zkclient.zkCLient完成listener(listener中相关的handle方法有相关的处理逻辑)的注册。
* zkClient继承Watcher，再监控的对象发生变化时，zkCLient中的process方法会被调用
* 在该方法中会完成对不同的listener的handle方法的调用


I0Itec-zkclient主要有如下三种listener，分别实现不同的监听方式。

消息 | handle的方法 | 备注
------------|----- | -----
IZkChildListener | handleChildChange | listening on zk child changes for a given path
IZkDataListener | handleDataChange | listening on zk data changes for a given path
IZkDataListener |handleDataDeleted | listening on zk data changes for a given path
IZkStateListener  | handleStateChanged |Called when the zookeeper connection state has changed.
IZkStateListener  | handleNewSession | Called after the zookeeper session has expired and a new session has been created
IZkStateListener  | handleSessionEstablishmentError |  Called when a session cannot be re-established

Kafka内部有大量的listener机制完成不同的功能。
listener接口 | listener |  listener 模块 | 监控对象 | 备注
------------|----- | ----- | ----- | -----
IZkChildListener | DeleteTopicsListener  | PartitionStateMachine | DeleteTopicsPath = "/admin/delete_topics" |
IZkChildListener | IsrChangeNotificationListener  | KafkaController |  IsrChangeNotificationPath = "/isr_change_notification" |
IZkChildListener | zkTopicEventListener  | ZookeeperTOpicEventWatcher |BrokerTopicsPath = "/brokers/topics" |
IZkChildListener | NodeChangeListener  | ZkNodeCHangeNotificaionListener  in SimpleAclAuthorizer| AclChangedZkPath = "/kafka-acl-changes"  AclChangedPrefix = "acl_changes_" |
IZkChildListener | NodeChangeListener  | ZkNodeCHangeNotificaionListener  In DynamicConfigManager|EntityConfigChangesPath = "/config/changes"  EntityConfigChangeZnodePrefix = "config_change_"  |
IZkChildListener | ZKRebalancerListener  | ZookeeperConsumerConnector |BrokerIdsPath = "/brokers/ids"  | 同时也会被由ZKTopicPartitionChangeListener触发，然后调用syncedRebalance进行rebalance
IZkChildListener | TopicChangeListener  | PartitionStateMachine | BrokerTopicsPath = "/brokers/topics" |
IZkChildListener | BrokerChangeListener  | ReplicaStateMachine |  BrokerIdsPath = "/brokers/ids"|
IZKDateListener  | PreferredReplicaElectionListener | KafkaController | PreferredReplicaLeaderElectionPath = "/admin/preferred_replica_election" |
IZKDateListener  | ZKTopicPartitionChangeListener | ZookeeperConsumerConnector |"/brokers/topics/${topic_name}" | consumer监控的partition的变化并作出处理。 高级api
IZKDateListener  | PartitionsReassignedListener | KafkaController |ReassignPartitionsPath = "/admin/reassign_partitions"  |
IZKDateListener  | LeaderChangeListener | ZookeeperLeaderElector | ControllerPath = "/controller" | Kafka的broker服务的升controller的竞选目录
IZKDateListener  | ReassignedPartitionsIsrChangeListener | KafkaController |"/brokers/topics/${topic_name}/partitions/${partition_id}/state"  |
IZKDateListener  | PartitionModificationsListener | PartitionStateMachine |"/brokers/topics/${topic_name} |
IZkStateListener | SessionExpireListener | KafkaHealthcheck |  | kafka健康检查会检查进程和zk的session状态
IZkStateListener | ZKSessionExpireListener | ZookeeperConsumerConnector | |consumser初始化时注册，在新建session时，也会调用到用syncedRebalance进行rebalance
IZkStateListener | SessionExpirationListener | KafkaController |  | kafkacontroller会监听与zk的链接状态，如果新建session，会调用onControllerResignation，并重新竞选leader
IZkStateListener | ZkStateChangeListener | ZkNodeChangeNotificationListener In SimpleAclAuthorizer | AclChangedZkPath = "/kafka-acl-changes"  AclChangedPrefix = "acl_changes_"  | 在新建seesion时会通过注册的notificaionhandler处理所有notifications
IZkStateListener | ZkStateChangeListener | ZkNodeChangeNotificationListener In DynamicConfigManager | EntityConfigChangesPath = "/config/changes"  EntityConfigChangeZnodePrefix = "config_change_"  | 在新建seesion时会通过注册的notificaionhandler处理所有notifications
IZkStateListener | ZkSessionExpireListener | ZookeeperTopicEventWatcher | | 消费时监控和zk的链接状态，低级api有效

