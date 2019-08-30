
---
layout: post
title:  "Spark 配置"
date:   2018-01-13 13:20:12 +0800
tags:
      - Spark
---

## 配置方法

```
a）spark-defaults.conf配置文件
b）--conf制定
c）new SparkConf().set("key","value")
d） --properties 指定配置文件用来替代spark-defaults.conf
```

## [](#配置注意事项)配置注意事项

```
1: 优先级： a < b < c 
2: 使用4）制定配置文件后，spark-defaults.conf配置文件失效
3: 不建议使用方法c来制定配置
4：使用--conf name=value指定配置时，
    a: 如果有多对配置，则每对配置都需要使用--conf指定
    b: 如果一个配置中value有多个，则需要使用引号“”将value包含。 如--conf spark.driver.extraJavaOptions="-Da=b -Dx=y"
5: 应用开发过程中，如有大量配置需要定制，需要应用层使用灵活的配置指定方法。
```

## [](#配置踩过的坑)配置踩过的坑

```
spark配置项可参考http://spark.apache.org/docs/latest/configuration.html#spark-ui。
如下介绍部分遇到过问题的部分配置：
spark.eventLog.enabled，spark.eventLog.dir spark应用是否记录evnetLog，以及eventLog日志记录的目录，需要与JObHistory进程的spark.history.fs.logDirectory指定同样的路径，才能在JobHistory页面展示该应用。
JobHistory通过解析应用的event文件在ui上展示应用信息。 在获取该文件后，可将该文件放入JobHistory的目录下重启JobHistory来查看应用运行情况
在1.2之前的spark版本，在集群外yarn-client模式下部署spark应用需要添加配置spark.driver.host，将值设置为本机ip。后续版本无需指定
SPARK_LOCAL_IP 在一般的集群环境中，无需指定。如果在节点中设置集群（am节点）该环境变量，将导致应用运行异常。
在代码中指定的与JVM相关的参数，如container内存，不会生效
使用conf.setMaster("local")后将无法再yarn页面查看到相关应用
JobHistory通过解析应用的event文件在ui上展示应用信息。 在获取该文件后，可将该文件放入JobHistory的目录下重启JobHistory来查看应用运行情况
Spark应用由非安全集群切换至安全集群时，需要根据集群部署情况调整相关参数。如果集群没有HDFS，HBase，Hive等组件时，将相关配置设置为false，否则应用会由于无法获取响应组件的token而启动失败。其中$service 分别为hdfs，hbase，hive。
    2.1之前版本： spark.yarn.security.tokens.${service}.enabled
    2.1之后版本： spark.yarn.security.credentials.${service}.enabled
安全相关的配置(principal和keytab)可参考https://www.jianshu.com/p/b6dfa8cd6d4a
```
