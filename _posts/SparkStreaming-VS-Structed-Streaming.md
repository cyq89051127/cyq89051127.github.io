---
layout: post
title:  "SparkStreaming-VS-Structed-Streaming"
date:   2018-07-09 21:54:12 +0800
tags:
      - Spark
---

## 导言



Spark在2.*版本后加入StructedStreaming模块，与流处理引擎Sparkstreaming一样，用于处理流数据。但二者又有许多不同之处。

Sparkstreaming首次引入在0.*版本，其核心思想是利用spark批处理框架，以microbatch（以一段时间的流作为一个batch）的方式，完成对流数据的处理。

StructedStreaming诞生于2.*版本，主要用于处理结构化流数据，除了与Sparkstreaming类似的microbatch的处理流数据方式，也实现了long-running的task，可以"不停的"循环从数据源获取数据并处理，从而实现真正的流处理。以dataset为代表的带有结构化（schema信息）的数据处理由于钨丝计划的完成，表现出更优越的性能。同时Structedstreaming可以从数据中获取时间（eventTime），从而可以针对流数据的生产时间而非收到数据的时间进行处理。


StructedStreaming的相关介绍可参考(http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
SparkStreaming的相关介绍可参考(http://spark.apache.org/docs/latest/streaming-programming-guide.html)。

## SparkStreaming与Structed Streaming对比

流处理模式| SparkStreaming | Structed streaming
---|--- | ----
执行模式 | Micro Batch | Micro batch / Streaming  
 API| Dstream/streamingContext | Dataset/DataFrame,SparkSession
Job 生成方式 | Timer定时器定时生成job | Trigger触发
支持数据源 |Socket,filstream,kafka,zeroMq,flume,kinesis | Socket,filstream,kafka,ratesource
executed-based |Executed based on dstream api |Executed based on sparksql
Time based | Processing Time | ProcessingTime & eventTIme
UI | Built-in |  No 


### 执行模式

Spark Streaming以micro-batch的模式

    以固定的时间间隔来划分每次处理的数据，在批次内以采用的是批处理的模式完成计算

Structed streaming有两种模式：

Micro-batch模式

    一种同样以micro-batch的模式完成批处理，处理模式类似sparkStreaming的批处理，可以定期（以固定间隔）处理，也可以处理完一个批次后，立刻进入下一批次的处理

Continuous Processing模式

    一种启动长时运行的线程从数据源获取数据，worker线程长时运行，实时处理消息。放入queue中，启动long-running的worker线程从queue中读取数据并处理。该模式下，当前只能支持简单的projection式（如map,filter,mappartitions等）的操作

### API

Sparkstreaming : 

    Sparkstreaming框架基于RDD开发，自实现一套API封装，程序入口是StreamingContext，数据模型是Dstream，数据的转换操作通过Dstream的api完成，真正的实现依然是通过调用rdd的api完成。

StructedStreaming ：

    Structed Streaming 基于sql开发，入口是sparksession，使用的统一Dataset数据集，数据的操作会使用sql自带的优化策略实现


### Job生成与调度

SparkStreamingJob生成：

    job通过定时器根据batch duration定期生成Streaming的Job（org.apache.spark.streaming.scheduler.Job）该Job是对Spark core的job的封装，在run方法中会完成对sparkcontext.runJob的调用
    // 通过timer定时器定期发生generatorJobs的消息    
        timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
        longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
    //调用eventLoop的processEvent方法处理根据消息类型处理消息
        private def processEvent(event: JobGeneratorEvent) {
        logDebug("Got event " + event)
        event match {
          case GenerateJobs(time) => generateJobs(time)
          case ClearMetadata(time) => clearMetadata(time)
          case DoCheckpoint(time, clearCheckpointDataLater) =>
            doCheckpoint(time, clearCheckpointDataLater)
          case ClearCheckpointData(time) => clearCheckpointData(time)
        }
      }
Sparkstreaming的Job的调度：

    生成Job之后，通过调用JobScheduler的submitJobSet方法提交job，submitJobSet通过jobExecutor完成调用job的run方法，完成spark Core job的调用
        def submitJobSet(jobSet: JobSet) {
        if (jobSet.jobs.isEmpty) {
          logInfo("No jobs added for time " + jobSet.time)
        } else {
          listenerBus.post(StreamingListenerBatchSubmitted(jobSet.toBatchInfo))
          jobSets.put(jobSet.time, jobSet)
          jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
          logInfo("Added jobs for time " + jobSet.time)
        }
        }
        其中JobExecutor是一个有numCorrentJob（由spark.streaming.concurrentJobs决定，默认为1）个线程的线程池，定义如下：
        jobExecutor = ThreadUtils.newDaemonFixedThreadPool(numConcurrentJobs, "streaming-job-executor")

StructedStreaming job生成与调度 ： 

    Structed Streaming通过trigger完成对job的生成和调度。具体的调度可参考https://www.jianshu.com/p/d4e2a2be9a10中的About trigger章节


### 支持数据源

SparkStreaming出现较早，支持的数据源较为丰富：Socket,filstream,kafka,zeroMq,flume,kinesis

Structed Streaming 支持的数据源 ： Socket,filstream,kafka,ratesource

### executed-based

SparkStreaming在执行过程中调用的是dstream api，其核心执行仍然调用rdd的接口

Structed Streaming底层的数据结构为dataset/dataframe，在执行过程中可以使用的钨丝计划，自动代码生成等优化策略，提升性能

### Time Based

Sparkstreaming的处理逻辑是根据应用运行时的时间（Processing Time）进行处理，不能根据消息中自带的时间戳完成一些特定的处理逻辑

Structed streaming，加入了event Time，weatermark等概念，在处理消息时，可以考虑消息本身的时间属性。同时，也支持基于运行时时间的处理方式。

### UI 

Sparkstreaming 提供了内置的界面化的UI操作，便于观察应用运行，批次处理时间，消息速率，是否延迟等信息。

Structed streaming 暂时没有直观的UI
