---
layout: post
title:  "Spark应用配置文件汇总"
date:   2018-07-20 10:14:12 +0800
tags:
      - Spark
---
在大数据应用开发过程中，会频发遇到不同底层“HADOOP”平台的问题；在不同厂家的平台，不同的部署模式（安全/非安全）下，不同的集群隔离性的情况下（是否严格的防火墙限制）等条件下，应用的部署也是较为复杂的问题。

本文旨在梳理在不同的场景下部署一个Spark应用都需要哪些前置条件。主要针对隔离集群下的部署要求和配置文件进行分析，以期找出部署Spark应用的充分必要条件。


## Spark应用需要集群的端口开通情况

### yarn-client模式

在集群隔离（设置严格防火墙的条件下），yarn-client模式的Spark应用提交有诸多限制，不建议在此场景下使用该模式运行Spark应用。

### yarn-cluster模式

    可参考yarn-cluster模式spark应用客户端与集群的通信端口 ： https://www.jianshu.com/p/d135f3267ab9



## Spark应用需要的配置文件

### 非安全模式下


配置文件 | 功能
---|---
core-site.xml | hadoop核心配置文件，缺少将导致无法找到hdfs集群，“fs.defaultFS”
hdfs-site.xml | hdfs配置文件，访问hdfs所需
yarn-site.xml | yarn配置文件，访问yarn所需


### 安全模式下

在安全模式下，提交Spark应用至HADOOP集群依然需要上一章节描述到的配置文件，除此之外，也需要kerberos相关的配置文件

通常hadoop集群在安全模式通常是基于kerberos服务。在访问kerberos时，基本的配置文件krb5.conf提供了kerbers服务信息

当然如果想要测试kerberos交互，验证等便捷操作，一些命令行如kinit,klist,kdestroy等也是必要的

如果有与zk后者kafka交互的需求，jaas.conf也是不可缺少的配置


配置文件 | 功能
---|---
core-site.xml | hadoop核心配置文件，缺少将导致无法找到hdfs集群，“fs.defaultFS”
hdfs-site.xml | hdfs配置文件，访问hdfs所需
yarn-site.xml | yarn配置文件，访问yarn所需
krb5.conf | kerberos服务认证使用
jaas.conf  | 在由kafka/zk访问需求时，使用该配置文件完成认证
user.keytab | 安全认证的秘钥，与principal需要匹配

    
    


