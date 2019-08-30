---
layout: post
title:  "Spark-on--hdp-Yarn-Cluster-踩坑(hdp-version)"
date:   2018-03-30 10:30:12 +0800
tags:
      - Spark
---
 ### 开源Spark运行在hdp的yarn集群失败分析：
 
 ###  部署方案
 
 * spark官网下载基于hdp的Hadoop版本的pre-built的spark安装包
 * 在机器上解压，并在spark-env中配置HADOOP_CONF_DIR，SPARK_CONF_DIR，spark-defaults中添加相关配置
 
### 测试情况： 

    a ) : local模式运行sparkPi 成功
    b ) : 使用yarn-client模式运行异常，下面分析该异常
 
 ### 问题现象 ： 
 
    在hdp的yarn集群时由于am启动异常而失败，异常“ ERROR SparkContext:91 - Error initializing SparkContext.org.apache.spark.SparkException: Yarn application has already ended! It might have been killed or unable to launch application master.”
    
    查看am日志发现am启动失败点原因为"Error: Could not find or load main class org.apache.spark.deploy.yarn.ExecutorLauncher" 
    该类时am启动的核心类，排查jar包异常发现该类所在jar包 spark-yarn*.jar存在且包含该类。
    
    查看Yarn原生页面抛出打印异常： “Exception message: /data/hadoop/yarn/local/usercache/ocsp/appcache/application_1519982778829_0171/container_e37_1519982778829_0171_02_000001/launch_container.sh: line 21: $PWD:$PWD/__spark_conf__:$PWD/__spark_libs__/*:$HADOOP_CONF_DIR:/usr/hdp/current/hadoop-client/*:/usr/hdp/current/hadoop-client/lib/*:/usr/hdp/current/hadoop-hdfs-client/*:/usr/hdp/current/hadoop-hdfs-client/lib/*:/usr/hdp/current/hadoop-yarn-client/*:/usr/hdp/current/hadoop-yarn-client/lib/*:$PWD/mr-framework/hadoop/share/hadoop/mapreduce/*:$PWD/mr-framework/hadoop/share/hadoop/mapreduce/lib/*:$PWD/mr-framework/hadoop/share/hadoop/common/*:$PWD/mr-framework/hadoop/share/hadoop/common/lib/*:$PWD/mr-framework/hadoop/share/hadoop/yarn/*:$PWD/mr-framework/hadoop/share/hadoop/yarn/lib/*:$PWD/mr-framework/hadoop/share/hadoop/hdfs/*:$PWD/mr-framework/hadoop/share/hadoop/hdfs/lib/*:$PWD/mr-framework/hadoop/share/hadoop/tools/lib/*:/usr/hdp/${hdp.version}/hadoop/lib/hadoop-lzo-0.6.0.${hdp.version}.jar:/etc/hadoop/conf/secure:$PWD/__spark_conf__/__hadoop_conf__: bad substitution”
    该异常是yarn在启动container时，调用launch_container.sh:脚本，该脚本返回点异常信息。 此处执行点时export CLASSPATH 也就是为调用container启动准备环境变量时，该行执行异常，导致添加classpath失败。进而导致找不到executor启动类。
    
    该命令执行失败时由于命令中包含/usr/hdp/${hdp.version}/hadoop/lib/hadoop-lzo-0.6.0.${hdp.version}.jar。 该操作时为进程添加lzo包，已实现lzo的压缩格式。查看hdp的hadoop lib目录，然而并没有该包。可能时为了客户方便使用默认将该目录导入
        [root@hosttest lib]# ll | grep lzo
        [root@hosttest lib]# pwd
        /usr/hdp/2.6.0.3-8/hadoop/lib
        [root@hosttest lib]# ll | grep -i lzo
        [root@hosttest lib]# 

### 解决方法 
    由于没有找到将改目录从classpath中移除的方法，就采用添加—Dhdp.version的方式，让此命令可以正常执行
      添加方法： 在spark-default.conf配置文件中添加: spark.driver.extraJavaOptions -Dhdp.version=2.6.5 spark.yarn.am.extraJavaOptions -Dhdp.version=2.6.5
    缺陷： 由于缺少真正的jar包。 因此lzo压缩算法不可用。
