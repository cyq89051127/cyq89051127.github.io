---
layout: post
title:  "Sql-On-Hadoop的流程简要分析"
date:   2018-10-03 23:25:12 +0800
tags:
      - Others
---
基于Hadoop的sql方案如hive，sparksql架构一般如下：
* Server ： ThriftServer 完成sql的解析及应用（如MR，Spark，Tez）的提交
* 传统数据库 ： 用于存储表的元数据，常见的由Mysql，postgreSql等
* 管理元数据： MetaStore，作为ThriftServer和传统数据库的桥梁
* 数据存储 ： HDFS 

###  Hive Sql执行流程图
![HiveSql执行力流程.jpg](https://upload-images.jianshu.io/upload_images/9004616-9dde8769c03f6fd6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

      
  ###   SparkSql 执行流程图
     
  SparkSql是基于spark Core的 onHadoop的sql解决方案。有多种sql解决方案，如通过启动Server的方式对客户端提交sql方案，客户端sql可通过beeline，JDBC的接口完成sql的解析执行。也可以直接调用sparkApi完成sql执行。
#### ThriftServer模式的sql方案
  ![SparkSql流程.jpg](https://upload-images.jianshu.io/upload_images/9004616-567fe19e4ac1a54c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### SparkApi模式的sql方案
  ![Spark Sql 流程.jpg](https://upload-images.jianshu.io/upload_images/9004616-5fbfd924334564b8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
