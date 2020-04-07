---
layout: post
title:  "Flink on yarn setup Guide"
date:   2020-04-02 16:40:12 +0800
tags:
      - Flink
---
## 版本

| 组件         | 版本 | 包名                                      |
| ------------ | :---- | ----------------------------------------- |
| Flink版本    | 1.8.2 | flink-1.8.2-bin-scala_2.11.tgz            |
| Hadoop-shade | 2.7.5 | flink-shaded-hadoop2-uber-2.7.5-1.8.0.jar |

## 安装步骤

1. 将jar包上传至要部署的节点的目录（如/opt/flink）下

2. 登录节点，进入/opt/flink目录下，解压flink安装包，并将Hadoop-shade包放入flink解压后的lib目录下    

   ```
   cd /opt/flink
   tar -xf flink-1.8.2-bin-scala_2.11.tgz
   mv flink-shaded-hadoop2-uber-2.7.5-1.8.0.jar flink-1.8.2/lib
   ```

3. 创建hadoop_conf目录，并拷贝hadoop配置文件至该目录

   ```
   mkdir flink-1.8.2/hadoop_conf
   cp /path/to/yarn-site.xml /path/to/core-site.xml /path/to/hdfs-site.xml flink-1.8.2/hadoop_conf
   注意： 如果配置文件是hdp集群中的配置文件，请确保yarn-site.xml中的属性yarn.timeline-service.enabled的值为false
   ```

4. 编辑flink-conf.yaml文件加入hadoop的配置路径

   ```
   env.hadoop.conf.dir: /data/flink-1.8.2/flink-1.8.2/hadoop_conf
   ```

5. 修改文件属组

   ```
   修改安装目录中文件用户属组，如使用root用户安装，则修改如下：
   chown root:root /opt/flink/ -R
   ```
   
6. 修改目录权限

   ```
   chmod 755 /opt/flink/ -R
   chmod 777 /opt/flink/logs -R
   chmod 777 /opt/flink/flink-1.8.2/log -R
   ```

7. 切换至相应用户，如cyq，使用如下命令，测试flink on yarn 作业提交运行情况

   ```
    /opt/flink/flink-1.8.2/bin/flink run -m yarn-cluster  /opt/flink/flink-1.8.2/examples/batch/WordCount.jar
   ```