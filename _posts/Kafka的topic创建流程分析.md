---
layout: post
title:  "Kafka的topic创建流程"
date:   2019-02-27 12:19:12 +0800
tags:
      - Kafka
---
# Kafka的topic创建流程

Kafka的topic创建一般通过调用客户端接口实现。接口通过获取集群信息将创建topic所需parttion, replica, leader， follower, isr等信息写入zk的topic相关目录，服务端通过zk的的listener机制，解析客户端写入的topic信息，完成topic的创建。主要逻辑如下：
## 客户端逻辑
* 解析参数，直接调用createtopic函数
* 根据传入参数调用AdminUtils.assignReplicasToBroker为每个partition的replia选择broker

      val brokerArray = brokerList.toArray
        val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
        var currentPartitionId = math.max(0, startPartitionId)
        var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
        for (_ <- 0 until nPartitions) {
          if (currentPartitionId > 0 && (currentPartitionId % brokerArray.length == 0))
            nextReplicaShift += 1
          val firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length
          val replicaBuffer = mutable.ArrayBuffer(brokerArray(firstReplicaIndex))
          for (j <- 0 until replicationFactor - 1)
            replicaBuffer += brokerArray(replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length))
          ret.put(currentPartitionId, replicaBuffer)
          currentPartitionId += 1
        }
    其中replicaIndex的函数如下：
    
        val shift = 1 + (secondReplicaShift + replicaIndex) % (nBrokers - 1)
        (firstReplicaIndex + shift) % nBrokers
      }
      
      由此可见，创建topic时，各partition的replica的分布由客户端决定

* 如果有指定相关配置，则将配置写相关目录下
* 将replica分配结果写入zk对应的目录中

## 服务端流程

*  kafkaServer启动时会通过选举启动controller，controller启动时会注册各种listener，其中包 括TopicChangeListener来监控topioc目录，当topic目录发生变动，会被对应的TopicChangelistener监听到，topic目录发生变化时，会触发如下流程：

#### 1：分析算出新增的newtopics，并获取每个newtopic的的每个partition的replic信息
#### 2： 调用controller的onNewTopicCreation方法

    2.1： 注册PartitionModificationsListener以监控topic的partition情况
    2.2： 分别调用partitionStateMachine将partition设置为newPartition状态
    2.3:  调用replicaStateMachine方法将所有replica设置为newState状态
    2.4： 调用partitionStateMachine将partition设置为OnlinePartition状态
    2.5： 调用replicaStateMachine方法将所有replica设置为OnlineReplica状态

### PartitionStateMachine 对partition的状态操作
    * NonExistPartition -> NewPartition
        直接设置partition状态为NewPartition
    * NewPartition -> OnlinePartition
        调用initializeLeaderAndIsrForPartition选出leader和isr列表，zk上创建目录并写入leaderAndIsr信息，向每一个replica所在节点发送LEADER_AND_ISR（PartitionStateInfo(主要消息为leaderIsrAndControllerEpoch, replicas.toSet)）请求，并像每个broker发送UPDATE_METADATA_KEY（主要消息为PartitionStateInfo(leaderIsrAndControllerEpoch, replicas)），同时将将要删除的partition消息向每个broker发送UPDATE_METADATA_KEY
    * offlinePartition -> OnlinePartition
        通过leaderSelect（此处为offlinePartitionSelector）选出leaderAndIsr，replicas，向每一个replica所在节点发送LEADER_AND_ISR（PartitionStateInfo(主要消息为leaderIsrAndControllerEpoch, replicas.toSet)）请求，并像每个broker发送UPDATE_METADATA_KEY（主要消息为PartitionStateInfo(leaderIsrAndControllerEpoch, replicas)），同时将将要删除的partition消息向每个broker发送UPDATE_METADATA_KEY
    * OnlinePartition ->  OnlinePartition
        通过leaderSelect（此处为offlinePartitionSelector）选出leaderAndIsr，replicas，向每一个replica所在节点发送LEADER_AND_ISR（PartitionStateInfo(主要消息为leaderIsrAndControllerEpoch, replicas.toSet)）请求，并像每个broker发送UPDATE_METADATA_KEY（主要消息为PartitionStateInfo(leaderIsrAndControllerEpoch, replicas)），同时将将要删除的partition消息向每个broker发送UPDATE_METADATA_KEY
    * NewPartition,onlinePartition,OfflinePartition -> offlinePartition
        直接设置partition状态为offlinePartition
    * OfflinePartition -> NonExistPartition
        直接设置partition状态为NonExistPartition
    
###  ReplicaStateMachine 对每个replica的状态操作
    * NonExistReplica -> NewReplica
        直接将replica设置为NewReplica状态
    * OfflineReplica -> ReplicaDeletionStarted
        将replica设置为ReplicaDeletionStarted状态，向replicaId的broker发送STOP_REPLICA（主要内容为StopReplicaRequestInfo(PartitionAndReplica(topic, partition, brokerId)）消息
    * ReplicaDeletionStarted -> ReplicaDeletionIneligible
        直接将replica设置为ReplicaDeletionIneligible状态
    * ReplicaDeletionStarted -> ReplicaDeletionSuccessful
        直接将replica设置为ReplicaDeletionSuccessful状态
    * ReplicaDeletionSuccessful -> NonExistentReplica
        在partitionReplicaAssignment和replicaState中去除该replica
    * NewReplica -> OnlineReplica
        将partitionReplicaAssignment中的添加该partition的replica
    * OnlineReplica, OfflineReplica, ReplicaDeletionIneligible ->OnlineReplica
        if（当前存在partition的leader）
            向该replica的broker发送送LEADER_AND_ISR（PartitionStateInfo(主要消息为leaderIsrAndControllerEpoch, replicas.toSet)）请求，，并像每个broker发送UPDATE_METADATA_KEY（主要消息为PartitionStateInfo(leaderIsrAndControllerEpoch, replicas)），同时将将要删除的partition消息向每个broker发送UPDATE_METADATA_KEY，并将relica状态设置为OnlineReplica
        else
            将relica状态设置为OnlineReplica
    * NewReplica, OnlineReplica, OfflineReplica, ReplicaDeletionIneligible -> offlineReplica
        向该replica的broker发送送LEADER_AND_ISR（PartitionStateInfo(主要消息为leaderIsrAndControllerEpoch, replicas.toSet)）请求，，并像每个broker发送UPDATE_METADATA_KEY（主要消
        为PartitionStateInfo(leaderIsrAndControllerEpoch, replicas)），同时将将要删除的partition消息向每个broker发送UPDATE_METADATA_KEY，并将relica状态设置为OnlineReplica
        if （当前存在partition的leader）
            调用controller.removeReplicaFromIsr中去除该replica，
                if （删除成功）
                    如果partition不是出于被删除状态，则向所有该partition但非该replica的broker发送送LEADER_AND_ISR（PartitionStateInfo(主要消息为leaderIsrAndControllerEpoch, replicas.toSet)）请求，，并像每个broker发送UPDATE_METADATA_KEY（主要消息为PartitionStateInfo(leaderIsrAndControllerEpoch, replicas)），同时将将要删除的partition消息向每个broker发送UPDATE_METADATA_KEY，并将relica状态设置为OfflineReplica
            
    
### LeaderAndIsr消息处理
当调用replicaManager的becomeLeaderOrFollower方法将对应的partition设置为leader或folloerw状态。主要逻辑如下：

     //分别找出要变为leader或follower状态的partitions
     val partitionsTobeLeader = partitionState.filter { case (partition, stateInfo) =>
              stateInfo.leader == config.brokerId
            }
            val partitionsToBeFollower = (partitionState -- partitionsTobeLeader.keys)
        // 分别调用makeLeaders和makeFollowers方法，将对应partition设置为对应的状态
        val partitionsBecomeLeader = if (!partitionsTobeLeader.isEmpty)
          makeLeaders(controllerId, controllerEpoch, partitionsTobeLeader, correlationId, responseMap)
        else
          Set.empty[Partition]
        val partitionsBecomeFollower = if (!partitionsToBeFollower.isEmpty)
          makeFollowers(controllerId, controllerEpoch, partitionsToBeFollower, correlationId, responseMap, metadataCache)
        else
          Set.empty[Partition]
        
        // we initialize highwatermark thread after the first leaderisrrequest. This ensures that all the partitions
        // have been completely populated before starting the checkpointing there by avoiding weird race conditions
        //如果没有启动hwcheckpoint线程，则启动
        if (!hwThreadInitialized) {
          startHighWaterMarksCheckPointThread()
          hwThreadInitialized = true
        }
        //关闭掉处于idle状态的fetcher线程
        replicaFetcherManager.shutdownIdleFetcherThreads()
        // 调用回调函数，仅针对内部保留的topic（_offset_consumer）进行迁入和迁出处理
        //其操作也即是load或者remove掉groupAndOffset信息，以便consumer消费
        onLeadershipChange(partitionsBecomeLeader, partitionsBecomeFollower)
        BecomeLeaderOrFollowerResult(responseMap, Errors.NONE.code)
makeLeaders方法会针对要变为leader状态的partition停止掉对对应aprtition的fetch操作。调用partition的makeleader方法将partition设置为leader状态。该方法主要逻辑如下：

    def makeLeader(controllerId: Int, partitionStateInfo: PartitionState, correlationId: Int): Boolean = {
        val (leaderHWIncremented, isNewLeader) = inWriteLock(leaderIsrUpdateLock) {
          val allReplicas = partitionStateInfo.replicas.asScala.map(_.toInt)
          // record the epoch of the controller that made the leadership decision. This is useful while updating the isr
          // to maintain the decision maker controller's epoch in the zookeeper path
          controllerEpoch = partitionStateInfo.controllerEpoch
          // add replicas that are new
          allReplicas.foreach(replica => getOrCreateReplica(replica))
          val newInSyncReplicas = partitionStateInfo.isr.asScala.map(r => getOrCreateReplica(r)).toSet
          // remove assigned replicas that have been removed by the controller
          (assignedReplicas().map(_.brokerId) -- allReplicas).foreach(removeReplica(_))
          inSyncReplicas = newInSyncReplicas
          leaderEpoch = partitionStateInfo.leaderEpoch
          zkVersion = partitionStateInfo.zkVersion
          //根据之前的leader是否是该replica，判断本次是否是新leader
          val isNewLeader =
            if (leaderReplicaIdOpt.isDefined && leaderReplicaIdOpt.get == localBrokerId) {
              false
            } else {
              leaderReplicaIdOpt = Some(localBrokerId)
              true
            }
          val leaderReplica = getReplica().get
          // we may need to increment high watermark since ISR could be down to 1
          if (isNewLeader) {
            // construct the high watermark metadata for the new leader replica
            //为当前的leader使用当前的highWatermarkMetadata.messageOffset设置为highWatermarkMetadata
            leaderReplica.convertHWToLocalOffsetMetadata()
            // reset log end offset for remote replicas
            assignedReplicas.filter(_.brokerId != localBrokerId).foreach(_.updateLogReadResult(LogReadResult.UnknownLogReadResult))
          }
          (maybeIncrementLeaderHW(leaderReplica), isNewLeader)
        }
        // some delayed operations may be unblocked after HW changed
        if (leaderHWIncremented)
        // 执行一些delayed的请求
          tryCompleteDelayedRequests()
        isNewLeader
      }

makeFollowers方法会针对要变为follower状态的partition有如下操作：调用partition的makeleader方法将partition设置为follower状态。

     private def makeFollowers(controllerId: Int,
                                epoch: Int,
                                partitionState: Map[Partition, PartitionState],
                                correlationId: Int,
                                responseMap: mutable.Map[TopicPartition, Short],
                                metadataCache: MetadataCache) : Set[Partition] = {
      ......

        try {
    
          // TODO: Delete leaders from LeaderAndIsrRequest
          partitionState.foreach{ case (partition, partitionStateInfo) =>
            val newLeaderBrokerId = partitionStateInfo.leader
            metadataCache.getAliveBrokers.find(_.id == newLeaderBrokerId) match {
              // Only change partition state when the leader is available
              case Some(leaderBroker) =>
                if (partition.makeFollower(controllerId, partitionStateInfo, correlationId))
                  partitionsToMakeFollower += partition
                else
                  ...
              case None =>
                ...
            }
          }
        //删除掉原有的该partition的fetch
          replicaFetcherManager.removeFetcherForPartitions(partitionsToMakeFollower.map(new TopicAndPartition(_)))
          partitionsToMakeFollower.foreach { partition =>
            stateChangeLogger.trace(("Broker %d stopped fetchers as part of become-follower request from controller " +
              "%d epoch %d with correlation id %d for partition %s")
              .format(localBrokerId, controllerId, epoch, correlationId, TopicAndPartition(partition.topic, partition.partitionId)))
          }
            // 将replica的offset修剪到highWatermark.messageOffset
          logManager.truncateTo(partitionsToMakeFollower.map(partition => (new TopicAndPartition(partition), partition.getOrCreateReplica().highWatermark.messageOffset)).toMap)
          partitionsToMakeFollower.foreach { partition =>
            val topicPartitionOperationKey = new TopicPartitionOperationKey(partition.topic, partition.partitionId)
            //完成（实质为清理）delayed的一些produce和fetch请求
            tryCompleteDelayedProduce(topicPartitionOperationKey)
            tryCompleteDelayedFetch(topicPartitionOperationKey)
          }
    
          ...
          if (isShuttingDown.get()) {
            partitionsToMakeFollower.foreach { partition =>
              stateChangeLogger.trace(("Broker %d skipped the adding-fetcher step of the become-follower state change with correlation id %d from " +
                "controller %d epoch %d for partition [%s,%d] since it is shutting down").format(localBrokerId, correlationId,
                controllerId, epoch, partition.topic, partition.partitionId))
            }
          }
          else {
            // we do not need to check if the leader exists again since this has been done at the beginning of this process
            //为变为follower状态的replica添加fetch线程
            val partitionsToMakeFollowerWithLeaderAndOffset = partitionsToMakeFollower.map(partition =>
              new TopicAndPartition(partition) -> BrokerAndInitialOffset(
                metadataCache.getAliveBrokers.find(_.id == partition.leaderReplicaIdOpt.get).get.getBrokerEndPoint(config.interBrokerSecurityProtocol),
                partition.getReplica().get.logEndOffset.messageOffset)).toMap
            replicaFetcherManager.addFetcherForPartitions(partitionsToMakeFollowerWithLeaderAndOffset)
    
            ...
          }
        } catch {
          case e: Throwable =>
            ...
            throw e
        }
    
        ...
    
        partitionsToMakeFollower
      }

MakeLeader方法流程比较简单，将isr列表设置为空，删除之前存在但当前请求中没有的replica

### UPDATE_METADATA_KEY消息的处理
updateMetaData逻辑比较简单，主要是更新broker维护的集群状态信息（主要包括aliveNodes ：.Map[Int, collection.Map[SecurityProtocol, Node]]，aliveBrokers ：Map[Int, Broker]，cache： Map[String, mutable.Map[Int, PartitionStateInfo]]）

    def updateCache(correlationId: Int, updateMetadataRequest: UpdateMetadataRequest) {
        inWriteLock(partitionMetadataLock) {
          controllerId = updateMetadataRequest.controllerId match {
              case id if id < 0 => None
              case id => Some(id)
            }
          aliveNodes.clear()
          aliveBrokers.clear()
          updateMetadataRequest.liveBrokers.asScala.foreach { broker =>
            val nodes = new EnumMap[SecurityProtocol, Node](classOf[SecurityProtocol])
            val endPoints = new EnumMap[SecurityProtocol, EndPoint](classOf[SecurityProtocol])
            broker.endPoints.asScala.foreach { case (protocol, ep) =>
              endPoints.put(protocol, EndPoint(ep.host, ep.port, protocol))
              nodes.put(protocol, new Node(broker.id, ep.host, ep.port))
            }
            aliveBrokers(broker.id) = Broker(broker.id, endPoints.asScala, Option(broker.rack))
            aliveNodes(broker.id) = nodes.asScala
          }
      updateMetadataRequest.partitionStates.asScala.foreach { case (tp, info) =>
        val controllerId = updateMetadataRequest.controllerId
        val controllerEpoch = updateMetadataRequest.controllerEpoch
        if (info.leader == LeaderAndIsr.LeaderDuringDelete) {
          removePartitionInfo(tp.topic, tp.partition)
          stateChangeLogger.trace(s"Broker $brokerId deleted partition $tp from metadata cache in response to UpdateMetadata " +
            s"request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
        } else {
          val partitionInfo = partitionStateToPartitionStateInfo(info)
          addOrUpdatePartitionInfo(tp.topic, tp.partition, partitionInfo)
          stateChangeLogger.trace(s"Broker $brokerId cached leader info $partitionInfo for partition $tp in response to " +
            s"UpdateMetadata request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
        }
      }
    }
