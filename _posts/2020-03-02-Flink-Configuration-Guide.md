---
layout: post
title:  "Flink Configuration Guide"
date:   2020-03-02 16:40:12 +0800
tags:
      - Flink
---

### Flink重要配置：


#### Flink重要的配置类

|配置类|说明|备注|
|---|----|----|
|ResourceManagerOptions.java | The set of configuration options relating to the ResourceManager|
|CoreOptions.java | The set of configuration options for core parameters|
|YarnConfigOptions.java | This class holds configuration constants used by Flink's YARN runners.|These options are not expected to be ever configured by users explicitly.|
|RestOptions.java |Configuration parameters for REST communication.||
|AkkaOptions.java| Akka configuration options.|
|BlobServerOptions.java|Configuration options for the BlobServer and BlobCache.|
|ClusterOptions.java|Options which control the cluster behaviour.|
|ExecutionOptions.java|specific for a single execution of a user program.|
|DeploymentOptions.java|The {@link ConfigOption configuration options} relevant for all Executors.|
|ExecutionConfigOptions.java|This class holds configuration constants used by Flink's table module.| This is only used for the Blink planner.All option keys in this class must start with "table.exec".|
|ExecutionCheckpointingOptions.java | Execution {@link ConfigOption} for configuring checkpointing related parameters|
|CheckpointingOptions.java | A collection of all configuration options that relate to checkpoints and savepoints.|
|TaskManagerOptions.java | The set of configuration options relating to TaskManager and Task settings.|
|JobManagerOptions.java | Configuration options for the JobManager.|
|HeartbeatManagerOptions|The set of configuration options relating to heartbeat manager settings.|
|WebOptions.java | Configuration options for the WebMonitorEndpoint.|

#### Common configs
flink程序在运行时，flink的客户端进程加载的log4j文件的配置不会打印应用侧的日志，需要加上相关配置才会打印


| 参数                 | 含义                              | 作用                               | 配置方法 |
| ------------------------------ | ----------------------------------- | --------------------------------------- | ------------------------------ |
| env.java.opts.jobmanager       | Flink Job manager进程使用的jvm参数  | 比如远程debug之类的参数可以加在此配置中 | 1.-yD<br />2. flink-conf.yaml |
| env.java.opts.taskmanager      | Flink Task manager进程使用的jvm参数 | 比如远程debug之类的参数可以加在此配置中 | 1.-yD<br />2. flink-conf.yaml |
| containerized.master.env.      | 维表关联后可进行窗口统计分析        | 本质jobmanager使用,如配置了containerized.master.env.AAA=BBB,在启动jobmanager之前执行了Export AAA=BBB,同时配置用也包含一个containerized.master.env.AAA=BBB | 1.-yD<br />2. flink-conf.yaml |
| containerized.taskmanager.env. | container进程加载的配置             | 本质taskmanager使用，同containerized.master.env. | 1.-yD<br />2. flink-conf.yaml |
| FLINK_CONF_DIR                 | Flink 配置的目录                    | 可根据此参数设置配置目录                | export |
