---
layout: post
title:  "Spark-VS-Flink---流处理中的Time&Window"
date:   2019-04-06 16:22:12 +0800
tags:
      - Flink
---

本文主要对Flink和Spark集群的standalone模式及on yarn模式进行分析对比。Flink与Spark的应用调度和执行的核心区别是Flink不同的job在执行时，其task同时运行在同一个进程TaskManager进程中；Spark的不同job的task执行时，会启动不同的executor来调度执行，job之间是隔离的。

### Standalone模式

Flink 和Spark均支持standalone模式（不依赖其他集群资源管理和调度）的部署，启动自身的Master/Slave架构的集群管理模式，完成应用的调度与执行。

* Flink

### ![standalone-flink.jpg](https://upload-images.jianshu.io/upload_images/9004616-266d33e185f515f0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* Spark 
  ![standalone-spark.jpb.jpg](https://upload-images.jianshu.io/upload_images/9004616-a0ab57344809695b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

On-yarn模式

Flink on Yarn 模式，其ApplicationMaster实现对JobManager的封装，作为该job的核心，完成executionGraph的生成，task的分发，运行结果的处理等；而YarnTaskManager则继承至TaskManager，完成task的运行。

Spark on Yarn 模式下，根据driver及业务逻辑运行的进程不同分为yarn-client和yarn-cluster模式；

#### Flink on Yarn

* yarn-cluster模式

   Yarn-cluster模式下，Flink提交应用至Yarn集群，类似MR job，运行完后结束
    ![yarn-cluster_flink.jpg](https://upload-images.jianshu.io/upload_images/9004616-933253dc895b6cc6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* yarn-session模式

  Yarn-session模式下，首先向Yarn提交一个长时运行的空应用，运行起来之后，后分别启动YarnApplicationMasterRunner/ApplicationMaster/JobManager，和N个YarnTaskManager/Container，但此时没有任务运行；
  其他Flink客户端可通过制定ApplicationId的方式提交Flink Job到此JobManager,由该JobManager完成应用的解析和调度执行。
    ![yarn-session_flink.jpg](https://upload-images.jianshu.io/upload_images/9004616-5b08b83be399b238.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Spark on Yarn

Yarn-client和yarn-cluster的主要区别在于driver运行的进程不一样：
           

      在yarn-client模式下，driver及业务代码逻辑运行在yarn client进程中，与applicationMaster及executor交互完成应用的调度和执行。
    在Yarn-cluster模式下，应用提交至Yarn集群后，yarn client进程可以退出，driver及业务代码逻辑运行在applicationMaster进程中，与executor完成应用的调度执行。

* Yarn-client 
  ![yarn-client_spark2.jpg](https://upload-images.jianshu.io/upload_images/9004616-c84c6bfbbc5eaadc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* Yarn-cluster 
  ![yarn-cluster_spark.jpg](https://upload-images.jianshu.io/upload_images/9004616-2c771d4f7997ccde.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


PS ：

Flink和Spark在On yarn模式下的各进程核心功能对比如下

应用模块 |  Flink(yarn-cluster)  | Flink(yarn-session)|Spark(Yarn-client) | Spark (yarn-cluster)
---|---|---|---|---|
job提交 | flink client| flink client  | spark client | spark client |
job逻辑解析与调度| YarnApplicationMasterRunner| yarn-session中的YarnApplicationMasterRunner |spark client (driver) | ApplicationMaster  
task的执行|YarnTaskManager| yarn-session中的YarnTaskManager | Executor |Executor
