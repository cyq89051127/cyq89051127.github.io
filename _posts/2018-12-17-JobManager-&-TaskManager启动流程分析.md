### JobManager启动流程

* 核心启动类和方法 

      启动类 ：org.apache.flink.runtime.jobmanager.JobManager
      核心启动方法 ： startJobManagerActors

* ha模式：


模式| 场景
---|---
非HA模式| 测试，非生产模式使用
基于zk的ha模式 | 通用的ha模式
自定义HA模式 |  定制自己的ha实现

* 创建线程池：


线程池名称| 线程数 | 线程工作 | 工作性质
---|---|--- | ---
jobmanager-future | Hardware.getNumberCPUCores | executionGraph清理，采样task 栈追踪，metrics信息收集，task调度，job调度结果异常的handle，job失败后的restart | 异步工作
jobmanager-io | Hardware.getNumberCPUCores | ha模式下zk节点的checkpoint信息的handle，清理 checkpoint数据,定期checkpoint |涉及文件数据的存储，清理 


* 启动webService： 用于网页监控信息

* 启动JobManager&archive ： 

        创建并启动blobserver线程：监听并请启动线程处理请求，创建目录结构以缓存blob（Binary large object）或临时缓存blob
        
        FlinkScheduer ： 在不同的taskmanager/slot见调度task；使用了jobmanager-future线程池
        
        BlobLibraryCacheManager : 为一个应用下载library（通常是jar包）
        
        启动Jobmanager（作为actor）
        启动archive （作为actor）
    
* 启动JobManager ProcessReaper： 监控jobmangger运行情况，异常时退出进程

    如果是local部署模式，此时会将taskmanager在JobManager进程内启动，并启动taskManagerReaper来监控taskManager运行
    
* 启动ResouceManager ： 资源管理器，不同的模式下有不同的实现，负责启动taskManager，Yarn模式下的container等


### TaskManager启动
启动类：org.apache.flink.runtime.taskmanager.TaskManager
核心启动方法 ： selectNetworkInterfaceAndRunTaskManager

TaskManager启动流程较为简单，核心逻辑可参考 主要是启动TaskManager，TaskmanagerReaper，启动后直接向JobManager注册自己，注册完成后，进行部分模块的初始化（参考下节associateWithJobManager的逻辑）.


### TaskManager和JobManager的注册流程的交互

* Taskmanager向JobManager发送消息：
        
        如下消息是连续发送，并设置超时时间，如果规定时间内，消息不能发出，则退出TaskManager
        RegisterTaskManager(
                resourceID,
                location,
                resources,
                numberOfSlots)
* JobManager 收到消息后：

     如果之前未注册的taskManager，则收集信息并返回
        
         AcknowledgeRegistration(instanceID, blobServer.getPort)
    如果是已注册过的taskManager，则返回
    
        AlreadyRegistered(
            instanceID,
            blobServer.getPort))
    如果有异常，则返回
        
         RefuseRegistration(e)
*   TaskManager接收到JobManager的注册反馈信息后

    1） 反馈消息：AcknowledgeRegistration:
    
        调用associateWithJobManager方法，主要有如下行为：
            * 将状态设置为已连接    
            * BlobCacheService ： 从JobManager的BlobServer中获取数据
            * libraryCacheManager ： 从JobManager中获取jar包文件
            * FileCache ： 其中有个线程池flink-file-cache，负责为task创建目录，获取文件以及task运行结束后的清理
            * 监控JobManager(通过watch actorRef实现)的运行情况
        
    2） 反馈信息 ： AlreadyRegistered
    
        如果当前状态是已连接：
            则忽略该消息
        如果状态还是非连接状态：
            则调用associateWithJobManager
    
    3） 反馈信息 ：RefuseRegistration
    
            如果之前没有注册成功过：
                则等一定时间，继续注册
        之前已经注册成功过：
                忽略该消息
                
