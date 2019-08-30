
---
layout: post
title:  "Flink1.8-on-hdp-yarn踩坑记"
date:   2019-05-29 15:43:12 +0800
tags:
      - Flink
---

Flink1.8版本相比1.7在打包和jar包拆分上作出些许调整，对使用者有一定影响；如下是笔者在使用flink-1.8 on hdp yarn时踩的两个小坑

## 操作步骤

直接下载安装flink安装包flink-1.8.0-bin-scala_2.11.tgz，解压后，使用如下命令提交作业： 
        
    ./bin/flink run -m yarn-cluster ./examples/batch/WordCount.jar

## 坑一

作业会抛出异常： 

    Could not identify hostname and port in 'yarn-cluster'

### 原因：

Flink1.8中，FIX了[FLINK-11266](https://issues.apache.org/jira/browse/FLINK-11266),将flink的包中对hadoop版本进行了剔除，导致flink中直接缺少hadoop的client相关类，无法解析yarn-cluster参数

### 解决方法：

1. 执行命令前，导入hadoop的classpath

    export HADOOP_CLASSPATH=`hadoop classpath`

2. 在bin/config.sh添加如下语句，导入hadoop的classpath   

    export HADOOP_CLASSPATH=`hadoop classpath`
3. [下载官方预编译的hadoop相关的jar包](https://flink.apache.org/downloads.html)，下载时，选择对应的hadoop版本，如flink-shaded-hadoop2-uber-2.7.5-1.8.0.jar

## 坑二：
基于以上第三种方式的作业提交，会抛出如下异常：
    
        java.lang.NoClassDefFoundError: com/sun/jersey/core/util/FeaturesAndProperties
    	at java.lang.ClassLoader.defineClass1(Native Method)
    	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
    	at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    	at java.net.URLClassLoader.defineClass(URLClassLoader.java:467)
    	at java.net.URLClassLoader.access$100(URLClassLoader.java:73)
    	at java.net.URLClassLoader$1.run(URLClassLoader.java:368)
    	at java.net.URLClassLoader$1.run(URLClassLoader.java:362)
    	at java.security.AccessController.doPrivileged(Native Method)
    	at java.net.URLClassLoader.findClass(URLClassLoader.java:361)
    	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
    	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    	at org.apache.hadoop.yarn.client.api.TimelineClient.createTimelineClient(TimelineClient.java:55)
    	at org.apache.hadoop.yarn.client.api.impl.YarnClientImpl.createTimelineClient(YarnClientImpl.java:181)
    	at org.apache.hadoop.yarn.client.api.impl.YarnClientImpl.serviceInit(YarnClientImpl.java:168)
    	at org.apache.hadoop.service.AbstractService.init(AbstractService.java:163)
    	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.getClusterDescriptor(FlinkYarnSessionCli.java:1012)
    	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.createDescriptor(FlinkYarnSessionCli.java:274)
    	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.createClusterDescriptor(FlinkYarnSessionCli.java:454)
    	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.createClusterDescriptor(FlinkYarnSessionCli.java:97)
    	at org.apache.flink.client.cli.CliFrontend.runProgram(CliFrontend.java:224)
    	at org.apache.flink.client.cli.CliFrontend.run(CliFrontend.java:213)
    	at org.apache.flink.client.cli.CliFrontend.parseParameters(CliFrontend.java:1050)
    	at org.apache.flink.client.cli.CliFrontend.lambda$main$11(CliFrontend.java:1126)
    	at java.security.AccessController.doPrivileged(Native Method)
    	at javax.security.auth.Subject.doAs(Subject.java:422)
    	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1754)
    	at org.apache.flink.runtime.security.HadoopSecurityContext.runSecured(HadoopSecurityContext.java:41)
    	at org.apache.flink.client.cli.CliFrontend.main(CliFrontend.java:1126)
    Caused by: java.lang.ClassNotFoundException: com.sun.jersey.core.util.FeaturesAndProperties
    	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
    	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
    	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    	... 29 more

### 原因：
    
    在yarn的client提交作业时，会调用createTimelineClient，此时由于缺少相关jar包抛出异常，而如果之前有调用export HADOOP_CLASSPATH，则hdp自身的hadoop版本中包含有相关jar包，则可以正常运行。

### 解决方法：
    
    1. 修改HADOOP_CONF_DIR目录下yarn-site.xml中的yarn.timeline-service.enabled设置为false
    2. 创建flink自己的hadoop_conf,copy原有的HADOOP_CONF_DIR中的配置文件至hadoop_conf目录，并在flink-conf.yaml中设置env.hadoop.conf.dir变量，并指向新创建的hadoop_conf目录
