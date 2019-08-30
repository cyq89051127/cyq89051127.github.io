---
layout: post
title:  "跳坑Kafka"
date:   2019-07-27 18:52:12 +0800
tags:
      - Flink
---

## Flink对流流JOIN的支持

Flink对于join的支持有多种支持，可参考 [Flink Join类型](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/operators/joining.html)， 本文主要讨论Time interval join支持Table API的双流join，同时支持基于EventTime 和 ProcessingTime的的流流join。 Flink在TableApi中将流作为表使用，下文也不再区分流和表。

Flink对于interval join的支持从1.4版本开始，直到Flink1.6，经过几个版本的增强，形成了从开始的Table/Sql Api的支持，到后续DataStream Api的支持，从开始的inner join 到后来的left outer，right outer, full outerjoin的支持，算是完成了FLink对双流关联的支持，不同版本的功能支持如下：


Flink版本 | Join支持类型 | join API|
---|---|---
1.4 | inner | Table/SQL
1.5 | inner,left,right,full | Table/SQL
1.6 |inner,left,right,full  |Table/SQL/DataStream

**从官方给出的Release Note来看，Flink1.4，Flink1.5中的双流join是指windowed join，但从官方给出的样例以及源码来看，此处的Windowed Join 应当指的就是interval join;鉴于Flink版本近期的变更较大，笔者不再在原有老版本中测试相关功能，下文的介绍基于Flink最新release版本1.8**


## 关于Interval Join

在流与流的join中，与其他window join相比，window中的关联通常是两个流中对应的window中的消息可以发生关联， 不能跨window，Interval Join则没有window的概念，直接用时间戳作为关联的条件，更具表达力。由于流消息的无限性以及消息乱序的影响，本应关联上的消息可能进入处理系统的时间有较大差异，一条流中的消息，可能需要和另一条流的多条消息关联，因此流流关联时，通常需要类似如下关联条件：
    
    1. 等值条件如 a.id = b.id
    2. 时间戳范围条件 ： a.timestamp ∈ [b.timestamp + lowerBound; b.timestamp + upperBound]  b.timestamp + lowerBound <= a.timestamp and a.timestamp <= b.timestamp + upperBound
*其中lower bound,upperBound可设置为正值，负值，0* 
* 关联条件的含义

    如a.id = b.id and b.timestamp - 1 minutes <= a.timestamp and a.timestamp <= b.timestamp + 2 minutes 即表明两条流中的两条消息如果满足a.id = b.id 并且两条消息的时间戳满足a.timestamp在[b.timestamp-1minute, b.timestamp + 2 minutes] 之间，则两条消息应当发生关联

## Interval Join的实现

Interval join的实现基本逻辑比较简单，主要依靠[TimeBoundedStreamJoin](https://github.com/apache/flink/blob/release-1.8/flink-table/flink-table-planner/src/main/scala/org/apache/flink/table/runtime/join/TimeBoundedStreamJoin.scala)完成消息的关联，其核心逻辑主要包含消息的缓存，不同关联类型的处理，消息的清理，但实现起来并不简单，下面基于eventTime分别对以上进行分析：

*由于Flink对于流关联的处理逻辑是对于两条流的消息分别处理，但两条流的处理方式是完全一致的，一下基于第一条流（左流）进行分析*

假定左流中的消息l如`a,b,2019-07-22 00:00:00`，左流的可容忍乱序时间OutOfOrder时间设置为10s,其中第三个字段为时间戳字段

1. 更新当前的leftOperatorTime和rightOperatorTime值，更新其值为当前应用的CombineWatermark，当前应用watermark的获取方式如下：
    1. 两条流会分别基于接收到的消息计算每条流独立的watermark1,watermark2 
    2. 选取较小的watermark作为应用的CombineWatermark = min(watermark1,watermark2)
2.  找出消息时间戳，并结算右表中的能关联的时间戳范围
  
        val leftRow = cRowValue.row
        val timeForLeftRow: Long = getTimeForLeftStream(ctx, leftRow)
        val rightQualifiedLowerBound: Long = timeForLeftRow - rightRelativeSize // 2019-07-21 23:58:00.999
        val rightQualifiedUpperBound: Long = timeForLeftRow + leftRelativeSize // 2019-07-22 00:01:00.000
        表名右流中的消息，如果id满足需求，当其时间戳在[rightQualifiedLowerBound,rightQualifiedUpperBound]范围内时，将可以与左表发生关联


3.  将消息l与右表中的消息关联，并缓存l：

    当rightExpirationTime < rightQualifiedUpperBound时，将右表中的数据取出，判断是否可以与消息l发生关联：
    1. 首先计算新的rightExpirationTime：
    
        rightExpirationTime = leftOperatorTime - rightRelativeSize - allowedLateness - 1    
        
        其中rightExpirationTime表名由流中的有效消息，当右流中的消息m的时间戳小于rightExpirationTime时，表示不会再有左流中的消息可以与m发生关联，及m消息可以被清理
    2. 遍历rightCache，完成关联
        
            其中rightCache的数据结构为MapState[Long, JList[JTuple2[Row, Boolean]]]
            key为时间戳，value为对应时间戳的所有消息组成的List,其中List的元素为消息值和该消息是否被关联过的标记组成的tuple：
        遍历rightCache，完成如下操作：
        
            if（ 当其key值也就是时间戳rightTime满足时间的条件时） {
               遍历其对应的List,将消息值与l完成关联并输出，将其关联标记设置为true 
            }
                
            // 清理右表中已经不可能和左表中数据发生关联的消息
            if （当rightTime <= rightExpirationTime）{
                if (如果是right outer join或者full outer join) {
                    遍历JList,如果消息的关联标记为false，根据关联条件补齐空字段,并输出
                }
                移除List
            }
        
    3. 缓存消息l
        
            if (rightOperatorTime < rightQualifiedUpperBound){
                //表明消息l可能与后续右表中的消息发生关联，需要缓存消息l
                1. 在leftCache中缓存消息l
                2. 注册清理器TimerHeapInternalTimer(timeForLeftRow,...)，异步完成消息的清理，（清理器的触发由更新combinedwatermark时，当combinedwatermark>TimerHeapInternalTimer.timestamp将会触发清理器工作，和核心工作逻辑在TimeBoundedStreamJoin#onTimer方法中）
            }else{
                // 即该消息不需要缓存用于与右侧表的关联
                if (left outer join 或者 full outer join ) && if (消息未被关联过){
                    根据关联条件补齐空字段并输出
                }
            }
            
## Interval join 总结

* Flink的流关联当前只能支持两条流的关联
* Flink同时支持基于EventTime和ProcessingTime的流流join
* Interval join 已经支持inner ,left outer, right outer , full outer 等类型的join，由此来看官网对interval join类型支持的说明不够准确。
* 当前版本Interval join的两条流的消息清理是基于两条流共有的combinedWatermark（较小的流的watermark）
* 流的watermark不会用于将消息`直接`过滤掉，即时消息在本流中的watermark表示中已经迟到，但会直接将迟到的消息根据相应的join类型或输出或丢弃

