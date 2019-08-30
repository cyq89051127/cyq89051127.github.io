---
layout: post
title:  "为何Zookeeper的日志直接打印到控制台（console）"
date:   2018-07-17 12:19:12 +0800
tags:
      - Zookeeper
---

在开发hadoop应用时，为了便于开发，通常我们将日志打印到控制台来观察应用的运行情况。然而在使用到与zookeeper交互，在日志中经常性会打印一些info级别的日志在，即使设置了相关的日志级别，但依然可以打印出一下日志。下面对该现象的原因进行分析

Zookeeper客户端打印的日志情况如下：

    941 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:zookeeper.version=3.4.8--1, built on 02/06/2016 03:18 GMT
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:host.name=ocdt21.aicloud.local
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:java.version=1.8.0_77
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:java.vendor=Oracle Corporation
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:java.home=/usr/jdk64/jdk1.8.0_77/jre
    ......
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:java.io.tmpdir=/tmp
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:java.compiler=<NA>
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:os.name=Linux
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:os.arch=amd64
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:os.version=3.10.0-327.el7.x86_64
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:user.name=root
    942 [main] INFO org.apache.zookeeper.ZooKeeper - Client environment:user.home=/root


### 为何zookeeper的日志打印不受设定的日志级别的影响（使用的是slf4j日志模块，打印之前没有判定日志级别）

分析： 查看Zookeeper代码可以看出在Zookeeper初始化时会通过静态代码段打印出客户端的环境信息：

    static {
        //Keep these two lines together to keep the initialization order explicit
        LOG = LoggerFactory.getLogger(ZooKeeper.class);
        Environment.logEnv("Client environment:", LOG);
    }
继续追踪代码可以发现，在打印日志时使用到的是SimpleLogger类（使用了slf4j日志模块）的log方法打印日志（无论当前调用时，如何设置日志级别，源码中都是使用System.err.print完成打印），如下：

    private void log(String level, String message, Throwable t) {
        StringBuffer buf = new StringBuffer();
        long millis = System.currentTimeMillis();
        /*日志最初的数字：表示当前时间与Log系统初始化的时间间隔*/
        buf.append(millis - startTime);
        buf.append(" [");
        buf.append(Thread.currentThread().getName());
        buf.append("] ");
        buf.append(level);
        buf.append(" ");
        buf.append(name);
        buf.append(" - ");
        buf.append(message);
        buf.append(LINE_SEPARATOR);
        /*无论当前调用时，如何设置日志级别，源码中都是使用System.err.print完成打印*/
        System.err.print(buf.toString());
        if (t != null) {
          t.printStackTrace(System.err);
        }
        System.err.flush();
    }

### 为何通常我们打印的日志收到日志级别的影响（使用log4j模块打印日志）

以我们通常调用的log打印为例（一般使用log4j模块打印），通常的实现如下：

    private Logger logger = Logger.getLogger(ClientToCodis.class);
    logger.info("Take " + (endTime - startTime) + " ms");
    
点击进去之后可以发现，logger.info的实现保证了日志级别的有效性：

    void info(Object message) {
    if(repository.isDisabled(Level.INFO_INT))
      return;
    if(Level.INFO.isGreaterOrEqual(this.getEffectiveLevel()))
      forcedLog(FQCN, Level.INFO, message, null);
    }
