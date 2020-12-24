---
layout: post
title:  "SparkSQL JOIN选择策略"
date:   2020-09-13 17:25:12 +0800
tags:
      - Spark
---

JOIN是SQL中的重要语义，同时也是SQL操作中逻辑复杂，对性能影响也较大。Spark中也对SQL的JOIN进行了大量的优化。在Spark的优化策略中，有专门针对JOIN的优化（`JoinSelection`）。
其核心优化逻辑可参考下图：

![JOIN](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE5458285a2ca8f4a5773e523a581e2b86/9700)

> 由于不带等值条件（On 条件）的Join在实际使用中，较少使用，本文只对带有等值条件的join进行分析

一下针对上图中的各个分支进行解析，其实主要关注几组函数即可：
Spark在处理两张表的join时，将两张表分别定义为StreamIter表（遍历表）和Build表（查找表），在实现时，是通过依次遍历StreamIter中的记录，从Build表中查找匹配记录完成join操作。通常用户侧无序关注表。由Spark内部处理相关逻辑。

判断一张表是否可用作build表(包括左表能否用于build，右表能否用于build)：

Broadcast join 
-----------------

### 通过显示标记指定使用broadcat join

该分区需要带编写sql语句或者调用api时，显式指定使用broadcast join的模式。

```java
private def canBroadcastByHints(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
        : Boolean = {
        // 左表可以作为build表，并且使用了broadcast hint
        val buildLeft = canBuildLeft(joinType) && left.stats.hints.broadcast
        // 左表可以作为build表，并且使用了broadcast hint
        val buildRight = canBuildRight(joinType) && right.stats.hints.broadcast
        buildLeft || buildRight
      }
```
其中canBuildLeft和canBuildRight的实现如下：
```
    //只有inner join 和right join 才能将左表视为build表
    private def canBuildLeft(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | RightOuter => true
      case _ => false
    }
```

```
    // inner， left outer, left semi, leftanti 的场景下，支持右表作为build表
    private def canBuildRight(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | LeftOuter | LeftSemi | LeftAnti | _: ExistenceJoin => true
      case _ => false
    }
```


### 根据表信息自动判断是否使用broadcast join 


其核心逻辑是只要一个表可以作为build表且表大小满足广播条件

```
private def canBroadcastBySizes(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : Boolean = {
      // 左表可以作为build表，并且满足canBroadcast条件
      val buildLeft = canBuildLeft(joinType) && canBroadcast(left)
      // 右表可以作为build表，并且满足canBroadcast条件
      val buildRight = canBuildRight(joinType) && canBroadcast(right)
      buildLeft || buildRight
    }
```

广播条件的要求是表的大小小于参数`spark.sql.autoBroadcastJoinThreshold`对应的值，默认是10M，如下：

```
    private def canBroadcast(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold
    }
```

Shuffle Hash Join
-----------------------

*  设置优先使用SortMerge的参数为false，其中一张表可以用作build表，且该表可用来创建LocalHashMap，该表比另一张表muchSmaller，
*  左表的key是不能排序的。

如下，以right表作为build表，判断条件的源码如下：

```java
    !conf.preferSortMergeJoin  // spark.sql.join.preferSortMergeJoin 的值为false（默认为true）
    && canBuildRight(joinType)  // 右表可用于build表
    && canBuildLocalHashMap(right)  // 表大小 < 广播阈值 * shuffle的partition数目 plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
    && muchSmaller(right, left)   //  右表小于左表的1/3 ： right.stats.sizeInBytes * 3 <= left.stats.sizeInBytes
    || !RowOrdering.isOrderable(leftKeys)  // 或者左表的key不能排序
```


SortMerge Join
------------------

左表的key可以排序，即优先选用Sort Merge Join