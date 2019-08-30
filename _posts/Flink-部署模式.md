---
layout: post
title:  "Flink-部署模式"
date:   2018-11-25 19:23:12 +0800
tags:
      - Flink
---
本文仅对Flink集群的部署模式作简单探讨，并未对如何部署（如环境变量，节点互信，hadoop集成配置）作说明，相关方法可百度获取。

## Flink Local模式
在Local模式下仅模拟cluster集群，仅启动JobManager完成应用的运行。JobManager进程信息如下:
  ![jobmanager_local.jpg](https://upload-images.jianshu.io/upload_images/9004616-5f8110309feee3d9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

提交应用方式：
  ./flink run -p 1 ../examples/batch/WordCount.jar

## Flink StandAlone 模式
使用./start-cluster.sh 启动flink集群
JobManager进程信息 ： 
![jobManager_standalone.jpg](https://upload-images.jianshu.io/upload_images/9004616-7d3dcd42a845d070.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

TaskManager进程信息 ：
    
   ![taskmanager_standalone.jpg](https://upload-images.jianshu.io/upload_images/9004616-8b9827a7b8f61e72.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


提交Flink作业：
    
    ./flink run -p 2  ../examples/batch/WordCount.jar --input hdfs://192.168.242.133:9000/tmp/input  --output hdfs://192.168.242.133:9000/tmp/output
    
    
    
## flink-on-yarn 模式

FLink on  yarn 有两种运行模式
    
    
    * 创建一个yarn-session，后续提交应用至该session
    * 直接 yarncluster模式提交应用至yarn服务

### yarn-session模式：

该模式下，创建session以后，会启动一个Yarn applicationMaster和指定数目的Yarn Container。查看ApplicationMaster及Container进程信息可以看出，二者启动类分别为YarnAPplicationMasterRunner和YarnTaskManager。查看源码可以看出，本质上ApplicationMaster扮演的是JobManager的角色，Contaienr扮演的是TaskManager的角色，类似于启动一个Flink的集群。

然后可以通过直接提交应用至该session，由该Session负责运行应用

#### 创建yarn-session

启动flink的yarn-session，可以使用如下命令：

    ./yarn-session.sh -n 3 -jm 768 -tm 768

ApplicationMaster/JobManager 进程信息
    
  ![jobmanager_session.jpg](https://upload-images.jianshu.io/upload_images/9004616-689658afc0ee75a3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Container/TaskManager进程信息：
   ![taskmanager_session.jpg](https://upload-images.jianshu.io/upload_images/9004616-afca4fcdbe33d814.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 提交Flink作业命令：

* 使用相同用户，在启动yarn-sesion的节点上可以通过如下命令提交应用至yarn-session：
  

    ./flink run -p 2  ../examples/batch/WordCount.jar

* 指定yarn服务中yarn-session对应的applicationId，提交作业


    ./flink run -p 2 -yid application_1543030627541_0003  ../examples/batch/WordCount.jar

### yarn-cluster模式

   可以直接通过yarn-cluster模式提交应用至yarn服务，可使用如下命令：

    ./flink run -m yarn-cluster -yn 1 -yjm 1024 -ytm 1024  ../examples/batch/WordCount.jar

ApplicationMaster/JobManager
  
![jobmanager_cluster.jpg](https://upload-images.jianshu.io/upload_images/9004616-e28030fcd25d3c65.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Container/TaskManager进程信息：
  ![taskmanager_cluster.jpg](https://upload-images.jianshu.io/upload_images/9004616-a61c1cec5f791821.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

PS： 以上各部署模式架构图可参考：https://www.jianshu.com/p/0f4725f1b7d8

