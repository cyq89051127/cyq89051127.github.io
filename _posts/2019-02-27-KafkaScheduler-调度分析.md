---
layout: post
title:  "KafkaScheduler 调度分析"
date:   2019-02-27 12:19:12 +0800
tags:
      - Kafka
---
## kafkaScheduler调度模块
KafkaScheduler作为broker进程的调度模块，提供对线程池的封装，对于一些周期性/非周期性执行的逻辑，可用于周期性调度/非周期调度。主要包含如下流程：
### 周期的调度

模块 | 定时任务 | 参数 | 默认 | 执行逻辑 | 备注
------------|--------|-------|--------|------|-----
LogManager | CleanupLogs | log.retention.check.interval.ms | 300s | 过期日志文件清理 |
LogManager |  flushDirtyLogs | log.flush.scheduler.interval.ms | Integer.Max | 刷日志到日盘 |
LogManager | checkpointRecoveryPointOffsets | log.flush.offset.checkpoint.interval.ms |60s | 将topicandpartition的checkpoint写入磁盘 | 写入文件recovery-point-offset-checkpoint
ReplicaManager | checkpointHighWatermarks | replica.high.watermark.checkpoint.interval.ms | 5s | 对hw进行checkpoint | 写入文件replication-offset-checkpoint
ReplicaManager | maybeShrinkIsr | replica.lag.time.max.ms | 10s | 检查是否需要减少isr列表中的replica | 
ReplicaManager | maybePropagateIsrChanges | NA | 2.5s | 检查是否生成/广播ISr列表/需要写入zk | zk目录为/isr_change_notification/isr_change_
GroupMetadataManager | deleteExpiredOffsets | offsets.retention.check.interval.ms | 600s | 删除过期的offset | 一天之后失效
ZookeeperConsumerConnector | autoCommit | auto.commit.interval.ms | 60s | | 打开自动commit的场景下有效，默认值AutoCommitInterval，默认存储在zk，可以设置存储在kafka，以及zk
KafkaController | checkAndTriggerPartitionRebalance | leader.imbalance.check.interval.seconds | 300s | 执行分区均衡 | 打开自动分区均衡的场景下有效，当分区leader不在perferred节点比例大于leader.imbalance.per.broker.percentage/100时（默认10%），进行

### 非周期调度
模块 | 定时任务 | 参数 | 默认 | 执行逻辑 | 备注
------------|--------|-------|--------|------|-----
Log | flush(newOffset) | NA | NA | flush日志到磁盘 | roll方法中调用
Log |  deleteSeg | NA | NA | 删除segment | 删除前会先修改log和index文件后缀.deleted
GroupMetadataManager | loadGroupsAndOffsets | NA | NA | 加载partition的group和offset信息 | 在_offset_consumer的partition变为leader时执行
GroupMetadataManager | removeGroupsAndOffsets | NA | NA | 去除partition的group和offset信息 | 在_offset_consumer的partition变为follower时执行
