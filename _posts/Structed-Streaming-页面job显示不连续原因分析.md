---
layout: post
title:  "Structed-Streaming-页面job显示不连续原因分析"
date:   2018-06-27 13:20:12 +0800
tags:
      - Spark
---

问题现象：

提交Structed Streaming应用，查看job页面信息，job编号显示不连续，如下图所示：
![屏幕快照 2018-06-25 下午5.21.25.png](https://upload-images.jianshu.io/upload_images/9004616-1e2a131e7d3b2b62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下文将对如下三个问题进行分别分析，以便完整解释job显示不连续：

* 是否真正产生了两个个job
* Job是如何产生的
* 为何页面只显示一个job

### 确实产生了两个job
记忆中，spark的job是顺序增加的。显示的时候少了一部分，于是再次查看spark提交job的逻辑，在DAGScheduler的submitJob方法中，会调用如下逻辑增加jobId，可见jobId一定是按顺序产生。

    val jobId = nextJobId.getAndIncrement()

查看driver日志，发现job调度时，一个批次内触发两个job，编号为“奇数”发job，提交后，立刻打印运行结束。
    
    18/06/25 15:49:01 INFO SparkContext: Starting job: start at StreamApp.scala:271
    18/06/25 15:49:16 INFO DAGScheduler: Job 0 finished: start at StreamApp.scala:271, took 14.945671 s
    18/06/25 15:49:16 INFO SparkContext: Starting job: start at StreamApp.scala:271
    18/06/25 15:49:16 INFO DAGScheduler: Job 1 finished: start at StreamApp.scala:271, took 0.000045 s


### Job是如何产生的

查看microBatchExecution的调度逻辑，在每个批次内只会触发一次batch的操作（nextBatch.collet）。如下：
    
    reportTimeTaken("addBatch") {
      SQLExecution.withNewExecutionId(sparkSessionToRunBatch, lastExecution) {
        sink match {
          case s: Sink => s.addBatch(currentBatchId, nextBatch)
          case _: StreamWriteSupport =>
            // This doesn't accumulate any data - it just forces execution of the microbatch writer.
            nextBatch.collect()
        }
      }
    }

从调度层无法看出为何会触发两个job，于是使用Debug模式，将断点设置在DAGScheduler的runJob方法中submit这一行。Debug时，两次观察运行到此处的线程堆栈如下：

![屏幕快照 2018-06-25 下午4.58.58.png](https://upload-images.jianshu.io/upload_images/9004616-28586210b9fc3ae6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![屏幕快照 2018-06-25 下午5.00.02.png](https://upload-images.jianshu.io/upload_images/9004616-dc67e086c25a006b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看出，一次collect触发了两次job提交，均在SparkPlan的executeCollect方法中。


两个job是如何触发的：

一次Job在SparkPlan的executeCollect方法中，通过调用getBytesRdd方法中的execute完成job的提交。
![屏幕快照 2018-06-27 上午11.17.03.png](https://upload-images.jianshu.io/upload_images/9004616-b353cdd355933be6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


第二次是调用bytesArrayRdd.collect方法，显然会触发一次action。
![屏幕快照 2018-06-27 上午11.17.09.png](https://upload-images.jianshu.io/upload_images/9004616-d70e378130f1617a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 为何每个批次只在页面显示一个Job

#### Spark页面

Spark应用在运行时，driver直接会将任务信息直接展示在页面上，同时将运行的任务信息记录在event文件中，以便在应用运行结束后，JobHistory进程可以根据event文件将任务运行情况展示在页面上，以便观察和分析业务运行。

以SparkListenerJobStart方法为例，
![屏幕快照 2018-06-27 上午11.17.14.png](https://upload-images.jianshu.io/upload_images/9004616-dd77465c79ee71d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


页面展示原理如下：

Spark定义一系列的SparkListenerEvent事件，在执行动作如提交job，stage，task以及完成时，都会将相关的事件通过Listenerbus的put方法放入一个linkedBlockingQueue中

##### 运行中的应用的UI展示

事件信息的存储：

Spark driver中启动一个spark-listener-group-appStatus线程，从linkedBlockingQueue中获取数据，通过相应listener的处理时间的方法，完成时间的相应处理。里onJobStart事件为例，会调用appStatusListener的onJobEvent方法处理该事件，该方法将事件写入InMemoryStore中一个名为data的hashMap中
![屏幕快照 2018-06-27 上午11.24.11.png](https://upload-images.jianshu.io/upload_images/9004616-926103d822eb7df4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



事件信息的展示：

在点击ui页面的job页面时，会调用到AllJobsPage.scala中的render方法，通过store.jobList调用InMemorystore.view方法从data中获取job信息


##### 运行结束后的应用UI展示

事件信息的存储：

在应用运行时driver内部启动一个spark-listener-group-eventLog线程，从上述queue中获取event事件并通过listener处理相关请求，eventLoggingListener会调用doPostEvent方法根据消息类型分别处理，对于SparkListenerJobStart事件，会将消息转化为json串并写入event日志。

如下以SparkListenerJobStart事件为例,记录event事件流程如下：
![屏幕快照 2018-06-27 上午11.21.05.png](https://upload-images.jianshu.io/upload_images/9004616-272f647c8dd04ce7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


事件信息的展示：

在jobHistory页面点击app，进入app UI展示， 在job页面，会调用到AllJobsPage.scala中的render方法，通过store.jobList调用elementTrackingstore.read方法从eventLog文件中获取job信息




而在第一个job的WriteToDataSourceV2Exec中返回的rdd为SprakConText.emptyRDD,也就是执行上述collect时，返回的bytesArrayRdd为EmptyRDD，查看代码可以看到RDD的partitions为0，查看DagScheduler的submit方法

    if (partitions.size == 0) {
      // Return immediately if the job is running 0 tasks
      return new JobWaiter[U](this, jobId, 0, resultHandler)
    }
可以看到在提交job生成jobid后，如果RDD的partitions为0，则直接返回，并没有触发真正的job提交，stage划分，task调度等操作，也就不会有相应的job相关的信息生成，自然也就不会展示在页面上。
