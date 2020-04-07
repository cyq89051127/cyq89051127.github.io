---
layout: post
title:  "Zeppelin on Flink小试牛刀"
date:   2020-04-02 16:40:12 +0800
tags:
      - Flink
---

Zeppelin on Flink 尝鲜



Zepplin 从0.9 版本（当前该版本还未release，只有预览版）开始支持Flink最新版本1.10，鉴于Flink1.10版本全面合入了Blink能力，在sql使用上展现出强大实力，笔者决定使用其预览版尝鲜。



### 下载安装

1. 在[Zeppelin](http://zeppelin.apache.org/)官网下载页面可以下载到当前zeppelin-0.9预览版

2. 放置在节点，并解压后查看到目录结构如下：

3. 进入zeppelin配置目录，将

        1. 执行cp zeppelin-site.xml.template zeppelin-site.xml可以服务的配置ip，port等
        3. 执行cp zeppelin-env.sh.template zeppelin-env.sh 可以配置相关服务的环境变量
   
4. 启动zeppenlin服务

   ```shell
    ${ZEEPELIN_HOME}/bin/zeppelin-daemon.sh start
   ```


### Flink服务的安装与配置

​	Flink的安装可以参考[Flink on yarn setup](https://cyq89051127.github.io/2020/04/02/Flink-on-yarn-setup/)

### Zeppelin on flink 配置

使用浏览器登录zeppelin页面（默认是localhost:8080），点击右侧用户名进入`interpreter`页面，找到flink栏相关内容，设置如下几个参数：

| 配置（key）          | 值(value)                           |
| -------------------- | ----------------------------------- |
| FLINK_HOME           | /data/cyq/flink-1.10.0              |
| HADOOP_CONF_DIR      | /data/cyq/flink-1.10.0/hadoop_conf/ |
| flink.execution.mode | yarn                                |
| flink.yarn.appName   | your app name                       |
| flink.yarn.queue     | your yarn queue name                |

 ### Zeppelin on flink 作业提交

登录zeppelin页面，点击Notebook -> Flink Tutorial -> Streaming ETL进入demo页面（如下），点击相关启动即可。

![Zeppelin](https://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE7846fa09c0fbdf6da2f044a6d78bda92/9683)