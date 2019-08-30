---
layout: post
title:  "Spark-on--hdp-Yarn-Cluster-踩坑(timeline-service)"
date:   2018-04-06 10:30:12 +0800
tags:
      - Spark
---
###  部署方案
 
 * spark官网下载基于hdp的Hadoop版本的pre-built的spark安装包
 * 在机器上解压，并在spark-env中配置HADOOP_CONF_DIR，SPARK_CONF_DIR，spark-defaults中添加相关配置
 
        为方便使用，设置HADOOP_CONF_DIR指向目录为/usr/hdp/current/hadoop-client/conf目录

### 异常

使用./spark-shell --master yarn 启动sparkshell终端，抛出异常ClassNotFound ：com/sun/jersey/api/client/config/ClientConfig如下：
 ![D3033034@9C7B674C.7B52C65A.png.jpg](https://upload-images.jianshu.io/upload_images/9004616-07c36f500db41e48.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 在jar目录下搜索发现，确实不存在该类
    
     [ocsp@ocdt31 jars]$ pwd
    /data/cyq/spark-2.3.0-bin-hadoop2.7/jars
    [ocsp@ocdt31 jars]$ grep "ClientConfig"  ./ -Rn
    [ocsp@ocdt31 jars]$ 

于是网上查询发现该类在jersey-client包1.9版本中存在，而spark当前依赖的的jersery版本为2.2，jersey版本升级后，剔除了相关jar包。然而此处的调用是初始化yarnclientimpl时，调用，显然是yarn服务使用的。此处说明存在jar包冲突问题。只好将yarn服务中的jersey-client的jar包放在spark的jars目录下。

再次启动sparkshell命令，抛出异常“com/sun/jersey/api/client/config/ClientConfig”，如下：
![6E912E64@FD27C231.7B52C65A.png.jpg](https://upload-images.jianshu.io/upload_images/9004616-057125f2d4449d25.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   
 在jar目录下搜索发现，确实不存在该类
    
    [ocsp@ocdt31 jars]$ pwd
    /data/cyq/spark-2.3.0-bin-hadoop2.7/jars
    [ocsp@ocdt31 jars]$ grep "FeaturesAndProperties"  ./ -Rn
    [ocsp@ocdt31 jars]$ 
    
只好将yarn服务中的jersey-core的jar包放在spark的jars目录下。

再次启动./spark-shell --master yarn 启动成功。然而查看spark应用原生页面时发现点击executor页面无法打开，且在driver中抛出异常：

   ![3302915E@6210844B.7B52C65A.png.jpg](https://upload-images.jianshu.io/upload_images/9004616-2a53634c3f75a2fb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    	
 多次点击则抛出空指针异常，如下：
![6C5FD42F@A57E7F06.7B52C65A.png.jpg](https://upload-images.jianshu.io/upload_images/9004616-9702c7abbdc89ef9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


分析后，发现jersery-core的1.9版本中存在ws相关j类，与javax.ws.rs-api-2.0.1.jar中的类类名一致。classloader加载时加载了jerseyclient中的类，由于版本原因，导致运行异常。
![屏幕快照 2018-04-06 上午12.52.58.png](https://upload-images.jianshu.io/upload_images/9004616-855e02c4826eedf2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![屏幕快照 2018-04-06 上午12.53.24.png](https://upload-images.jianshu.io/upload_images/9004616-5832fe71dd2c72cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 冲突点
    如果使用jersey-client，jersey-core的1.9版本，可以使应用运行起来，但由于jar包冲突，导致executors页面无法查看，这对分析应用有较大麻烦。
    如果不使用jersey-client,jersey-core 1.9版本，则应用启动异常。

尴尬了。。。

### 问题再分析

基于如上分析，spark应用启动运行有问题，还怎么好好的玩耍呢。

可是，我是直接取的spark社区2.3基于hadoop2.7版本编译好的包，如果无法使用，这还得了。

于是使用社区的spark在社区的hadoop上运行，运行成功。于是怀疑问题出在配置上面。然而是哪个配置呢？分析堆栈可以发现，yarnclientimpl初始化时，调用的createTimelineClient服务时异常，查看HADOOP_CONF_DIR下的yarn-site配置文件，果然有yarn.timeline-service.enabled参数，且该值配置为true。 查看yarnclient的源码发现，确实通过读取该值

    if (conf.getBoolean(YarnConfiguration.TIMELINE_SERVICE_ENABLED,
        YarnConfiguration.DEFAULT_TIMELINE_SERVICE_ENABLED)) {
      timelineServiceEnabled = true;
      timelineClient = createTimelineClient();
      timelineClient.init(conf);
      timelineDTRenewer = getTimelineDelegationTokenRenewer(conf);
      timelineService = TimelineUtils.buildTimelineTokenService(conf);
    }

于是将yarn.timeline-service.enabled值设置为false，并删除之前copy到SPARK_HOME/jars目录下的jersey1.9相关jar包。启动./spark-shell --master yarn 启动成功，executor页面可以正常查看。

至此spark on hdp 启动成功。且页面查看正常。


### PS ： 最后一问

使用社区Spark在hdp的yarn中运行起初设置的yarn.timeline-service.enable为ture时，由于jersey版本不兼容问题，启动是异常的， 那使用hdp的spark2可以在yarn上运行，为何是正常的呢？

显然在配置yarn.timeline-service.enable为ture时，根据yarnclientimpl的代码，一定会走到createTImelineclient方法，此时初始化时也会遭遇类不存在的问题从而导致启动失败， 然而现实是hdp的spark客户端提交应用时，运行是正常的。。。。

于是开始怀疑hdp做过某种修改，查看hdp的源码发现在submitApp之前，强制设置yarn.timeline-service.enable为false。

![屏幕快照 2018-04-05 下午3.35.19.png](https://upload-images.jianshu.io/upload_images/9004616-5816198fb0a86d35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看HDP的代码和配置发现hdp专门为适配timeline-service服务针对tez和spark专门开发了插件。



