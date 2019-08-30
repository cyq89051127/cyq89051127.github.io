---
layout: post
title:  "Spark metrics"
date:   2017-12-21 12:19:12 +0800
tags:
      - Spark
---

## Spark Metric／restapi
服务运行时将服务信息展示出来方便用户查看时服务易用性的重要组成部分。特别时对于分布式集群服务。

spark服务本身有提供获取应用信息对方法，方便用户查看应用信息。Spark服务提供对master，worker，driver，executor，Historyserver进程对运行展示。对于应用（driver／executor）进程，主要提供metric和restapi对访问方式以展示运行状态。

### Metric信息：
服务/进程通过Metric将自身运行信息展示出来。spark基于Coda Hale Metrics Library库展示。需要展示的信息通过配置source类，在运行时通过反射实例化并启动source进行收集。然后通过配置sink类，将信息sink到对应的平台。

### Metrics的source和sink模块启动流程
以driver为例：driver进程启动metricSystem的流程：

#### 初始化：

SparkContext在初始化时调用 ： MetricsSystem.createMetricsSystem("driver", conf, securityManager)

然后等待ui启动后启动并绑定webui（executor则是初始化后直接启动）

metricsSystem.start()

metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler)))

#### 进一步查看MetricSystem创建过程：

创建MetricConfig，  val metricsConfig = new MetricsConfig(conf)

初始化MetricConfig，首先设置默认的属性信息：

prop.setProperty("*.sink.servlet.class","org.apache.spark.metrics.sink.MetricsServlet")

prop.setProperty("*.sink.servlet.path","/metrics/json")

prop.setProperty("master.sink.servlet.path","/metrics/master/json")

prop.setProperty("applications.sink.servlet.path","/metrics/applications/json")

加载conf/metric.properties文件或者通过spark.metrics.conf制定的文件。读取相关配置，metricsConfig.initialize()

在启动metricSystem时，则会注册并启动source和sink

registerSources()

registerSinks()

sinks.foreach(_.start)

默认启动对source如下：

Source | 服务／进程 | 收集信息 |
-------------|-------------|---------------------------------|
MasterSource     | Master进程Standalone模式生效| workers，aliveWorkers，apps，waitingApps|
ApplicationSource     | Master进程Standalone模式生效      |  application，status，runtime_ms，cores |
WorkerSource | Worker进程，Standalone模式生效     |    executors，coresUsed，memUsed_MB，coresFree，memFree_MB |
StreamingSource    | Streaming应用，driver进程      |  receivers，lastReceivedBatch_records...|
DAGSchedulerSource| driver进程    |    failedStages，runningStages，waitingStages，allJobs...|
CodegenMetrics     | sql应用，driver进程      |  sourceCodeSize，compilationTime，generatedClassSize，generatedMethodSize|
MesosClusterSchedulerSource | mesos调度模式下，drvier启用   |    WatingDrivers,LaunchedDrivers,retryDrivers|
CacheMetrics    | jobhistory进程      |  ookup.count. |
ExecutorAllocationManagerSource | driver进程，executor动态调度使用      |   numberExecutorsToAdd，numberMaxNeededExecutors，numberTargetExecutors.. |
ExecutorSource | executor进程     |   read_bytes，write_bytes，largeRead_ops，write_ops，read_ops，activeTasks... |

可配置的source如下：

Source | 服务／进程 | 收集信息 
-------------|-------------|---------|
| JvmSource | driver,executor,master,worker| 收集jvm运行信息|

配置方法：修改$SPARK_HOME/conf目录下的metrics.properties文件：

默认相关source已经统计在列。可添加source为jvmsource。添加之后则相关进程的jvm信息会被收集。配置方法

添加如下行：

driver.source.jvm.class=org.apache.spark.metrics.source.JvmSource

executor.source.jvm.class=org.apache.spark.metrics.source.JvmSource

或者*.source.jvm.class=org.apache.spark.metrics.source.JvmSource

source信息的获取比较简单，以DAGSchedulerSource的runningStages为例，直接计算dagscheduler的runningStages大小即可。override def getValue: Int = dagScheduler.runningStages.size

通过这些收集的信息可以看到，主要是方便查看运行状态，并非提供用来监控和管理应用
 Metric信息展示方法：
收集的目的是方便展示，展示的方法是sink。

常用的sink如下：

a) metricserverlet

 spark默认的sink为metricsserverlet，通过driver服务启动的webui绑定，然后展示出来。ip:4040/metrics/json(ip位driver节点的ip)展示：由于executor服务没有相关ui，无法展示metricsource的信息。 下图是配置过JVMsource后，通过driver节点的看到的metric信息。
       ![屏幕快照 2017-12-21 下午1.28.47.png](http://upload-images.jianshu.io/upload_images/9004616-d54439cf6d85e250.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

b) CSV方式(将进程的source信息，写入到csv文件,各进程打印至进程节点的相关目录下，每分钟打印一次)：

*.sink.csv.class=org.apache.spark.metrics.sink.CsvSink

*.sink.csv.period=1

*.sink.csv.directory=/tmp/

c) console方式（将进程的source信息写入到console/stdout

,输出到进程的stdout）：

*.sink.console.class=org.apache.spark.metrics.sink.ConsoleSink

*.sink.console.period=20

*.sink.console.unit=seconds

d) slf4j方式（直接在运行日志中查看）：

*.sink.slf4j.class=org.apache.spark.metrics.sink.Slf4jSink

*.sink.slf4j.period=10

*.sink.slf4j.unit=seconds

e) JMX方式（此情况下，相关端口要经过规划，不同的pap使用不同的端口，对于一个app来说，只能在一个节点启动一个executor，否则会有端口冲突）：

executor.sink.jmx.class=org.apache.spark.metrics.sink.JmxSink

JMX方式在配置后，需要在driver/executor启动jmx服务。 可通过启动应用时添加如下操作实现--conf "spark.driver.extraJavaOptions=-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=8090 -Dcom.sun.management.jmxremote.rmi.port=8001 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false  --conf "spark.executor.extraJavaOptions=-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=8002 -Dcom.sun.management.jmxremote.rmi.port=8003 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"

可通过jconsole工具链接至对应driver进程所在ip和端口查看jmx信息。

![屏幕快照 2017-12-21 下午1.36.25.png](http://upload-images.jianshu.io/upload_images/9004616-b9c12712d40cf59e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### restApi信息

#### 通过RestApi可查看信息如下：
除例metrics之外，用户还可以通过restApi接口查看应用运行信息。可以查询的信息如下（参见 http://spark.apache.org/docs/latest/monitoring.html）：

url | 描述
------------|-------- 
 /applications | A list of all applications
/applications/[app-id]/jobs | A list of all jobs for a given application  | 
/applications/[app-id]/jobs/[job-id] |Details for the given job | 
/applications/[app-id]/stages    | A list of all stages for a given application     | 
/applications/[app-id]/stages/[stage-id]| A list of all attempts for the given stage
/applications/[app-id]/stages/[stage-id]/[stage-attempt-id] | Details for the given stage attempt
/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskSummary | Summary metrics of all tasks in the given stage attempt |
/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskList    | A list of all tasks for the given stage attempt      | 
/applications/[app-id]/executors | A list of all executors for the given application 
/applications/[app-id]/storage/rdd | A list of stored RDDs for the given application      |   
/applications/[app-id]/storage/rdd/[rdd-id] | Details for the storage status of a given RDD   |  
/applications/[app-id]/logs| dDownload the event logs for all attempts of the given application as a zip file
/applications/[app-id]/[attempt-id]/logs | Download the event logs for the specified attempt of the given application as a zip file

#### 通过RestApi查看信息方法：

运行中的应用：通过driver进程查看：
ip:port/api/v1/....

其中Ip为driver所在节点ip，端口为4040. 如果一个节点运行多个driver，端口会以此累加至4040，4041，4042 . 如：10.1.236.65:4041/api/v1/applications/application_1512542119073_0229/storage/rdd/23(on yarn 模式会自动跳转至如下页面)
    ![屏幕快照 2017-12-21 上午11.21.08.png](http://upload-images.jianshu.io/upload_images/9004616-0a836f6bdedadf03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



 对于运行完的应用，可通过jobhistory服务查看
此场景下，需要提交应用时打开eventlog记录功能
      打开方法在应用的spark-defaults.conf中添加如下配置spark.eventLog.enabled为true，spark.eventLog.dir为hdfs:///spark-history 。
      其中/spark-history可配置，需要和jobhistory进程的路径配置一致 ，该路径可通过historyserver页面查看。
ip:port/api/v1/....(其中Ip为spark服务的jobhistory进程所在节点ip，默认端口为18080). 可通过如下方式访问：

![屏幕快照 2017-12-21 上午11.32.06.png](http://upload-images.jianshu.io/upload_images/9004616-f18587bb1713c376.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结：
Spark作为计算引擎，对于大数据集群来说，作为客户端向Yarn提交应用来完成数据的分析。所使用的资源一般在yarn控制之下。其应用场景并非作为服务端为其他组件提供服务。其所提供的信息通常是针对app级别，如job，stage，task等信息。一般的信息监控需求均可通过其ui页面查看。对于一些应用的运行情况，可通过restapi获取和分析。

