---
layout: post
title:  "Flink-on-Yarn-Per-job分析"
date:   2020-05-07 17:25:12 +0800
tags:
      - Flink
---

### JobManager启动分析

#### JobManager/AM进程启动命令

```shell
/usr/jdk64/jdk1.8.0_77/bin/java -Xms1448m -Xmx1448m -Dlog.file=/data/hadoop/yarn/log/**application_1581392414078_0311**/container_e67_1581392414078_0311_01_000001/jobmanager.log -Dlog4j.configuration=file:log4j.properties org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint
```

#### JobManager启动类

> org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint

#### 服务初始化

JobManager启动过程中，首先完成一些`基础服务`的初始化工作，如：

* RpcService ： 基于akka的rpc服务

* HighAvailabilityServices ：服务的高可用实现

* BlobServer ： 监听分发请求，管理blob

* HeartbeatServices:用于和其他进程发送和接收心跳

* MetricQueryService：Metric信息服务

随后JobManager会启动创建运行作业相关的服务，源码中大量采用工厂模式完成服务的包装、创建、启动，显得非常复杂难懂。JobManager进程启动时会创建和启动相关服务，如下：

| 服务接口（实现类）                                           | 工厂类接口（工厂类）                                         | 功能                                                      | 备注 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- | ---- |
| DispatcherResourceManagerComponent                           | DefaultDispatcherResourceManagerComponentFactory             | 包装和启动Dispatcher，ResourceManager，WebMonitorEndpoint |      |
| WebMonitorEndpoint                                           | RestEndpointFactory                                          | 基于netty的restserver服务                                 |      |
| DispatcherRunnerLeaderElectionLifecycleManager               | DispatcherRunnerFactory(DefaultDispatcherRunnerFactory)      | 封装dispacther的行为                                      |      |
| DispatcherGatewayService                                     | DispatcherGatewayServiceFactory(DefaultDispatcherGatewayServiceFactory) | 对dispatcher的包装，创建并启动Dispatcher(MiniDispatcher)  |      |
| Dispatcher(MiniDispatcher)                                   | DispatcherFactory(JobDispatcherFactory)                      | job提交、取消、停止、及状态查询                           |      |
| DispatcherLeaderProcessFactory(JobDispatcherLeaderProcessFactory) | DispatcherLeaderProcessFactoryFactory（JobDispatcherLeaderProcessFactoryFactory） | 用于生成DispatcherLeaderProcess                           |      |
| DispatcherLeaderProcess(JobDispatcherLeaderProcess)          | DispatcherLeaderProcessFactory(JobDispatcherLeaderProcessFactory) | 对DispatcherGatewayService的包装                          |      |
| JobManagerRunner(JobManagerRunnerImpl)                       | JobManagerRunnerFactory（DefaultJobManagerRunnerFactory）    | 包装和启动jobMaster                                       |      |
| JobMasterService(JobMaster)                                  | JobMasterServiceFactory(DefaultJobMasterServiceFactory)      | 执行job                                                   |      |
| ResourceManager(YarnResourceManager)                         | ResourceManagerFactory(YarnResourceManagerFactory)           | 封装资源管理，包含与Yarn交互的逻辑及slot管理              |      |

### TaskManager启动分析

#### TaskManager/Container进程启动命令

```                                                                                     shell
/usr/jdk64/jdk1.8.0_77/bin/java -Xmx251658235 -Xms251658235 -XX:MaxDirectMemorySize=211392922 -XX:MaxMetaspaceSize=100663296 -Dlog.file=/data/hadoop/yarn/log/**application_1581392414078_0311**/container_e67_1581392414078_0311_01_000002/taskmanager.log -Dlog4j.configuration=file:./log4j.properties org.apache.flink.yarn.YarnTaskExecutorRunner -Dtaskmanager.memory.framework.offheap.size=134217728b -D taskmanager.memory.network.max=77175194b -D taskmanager.memory.network.min=77175194b -D taskmanager.memory.framework.heap.size=134217728b -D taskmanager.memory.managed.size=308700779b -D taskmanager.cpu.cores=1.0 -Dtaskmanager.memory.task.heap.size=117440507b -D taskmanager.memory.task.off-heap.size=0b --configDir . -Dweb.port=0 -Djobmanager.rpc.address=yj03 -Dweb.tmpdir=/tmp/flink-web-a54f230b-7d4a-464f-963b-85a88a6cdf11 -Djobmanager.rpc.port=34905 -Drest.address=yj03
```

#### TaskManager启动类

> org.apache.flink.yarn.YarnTaskExecutorRunner

TaskManager中的核心服务为`TaskExecutor`，用户执行task。其中主要包含如下服务

* HighAvailabilityServices ： 服务的高可用实现

* RpcService 基于akka的rpc服务

* BlobCacheService ： 与BlobCache交互永久或临时缓存Blob

* HeartbeatServices ：用于和其他进程发送和接收心跳

* MetricQueryService：Metric信息服务

* TaskManagerService：用户封装TaskExecutor中如下服务

  * `IOManager`：管理TaskManager本地临时目录
  * `ShuffleEnvironment`: 用于管理shuffle服务，仅有基于netty的实现。
  * `KvStateService` ： 和TaskExecutor绑定的KVStateService
  * `BroadcastVariableManager`  用于管理广播变量
  * `JobManagerTable` ：管理JobManagerConnection
  * `JobLeaderService`: 用于监控JobLeader
  * `TaskExecutorLocalStateStoresManager`:TaskManager级别的本地存储和checkpoint存储
