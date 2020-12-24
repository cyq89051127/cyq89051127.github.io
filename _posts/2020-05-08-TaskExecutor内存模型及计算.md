---
layout: post
title:  "Flink-TaskExecutor内存分析"
date:   2020-05-08 17:25:12 +0800
tags:
      - Flink
---

Flink的TaskExecutor/Container进程主要运行工作线程，其内存管理对Flink作业的运行有重要意义。Flink的TaskExecutor进程的内存配置参数较多，理解较为复杂。本文尝试从Flink源码角度来分析一下进程中各内存大小是如何确定的。

首先看一下进程包含使用的内存分类，如下图：

![Flink Executor 内存模型](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCEffcdcf2fe55cc6a96f107cce940e9248/9686)



> 绿色为JVM堆内存，红色为堆外内存。进程各部分内存均是通过配置参数获取。
>
> -Xmx决定了TaskHeapMemory + FrameworkHeapMemory上限
>
> -XX:MaxDirectMemorySize 决定了TaskOffHeapMemory + FrameOffHeapMemory + NetworkMemory的上限

*为方便表述，如下我们分别使用以下变量名表示各部分内存*

|       变量名        |          含义          | Flink参数                                  |
| :-----------------: | :--------------------: | ------------------------------------------ |
|     TotalMemory     | TaskExecutor进程总内存 | taskmanager.memory.process.size            |
|     FlinkMemory     |    Flink管理的内存     | taskmanager.memory.flink.size              |
|   TaskHeapMemory    |    Task Heap Memory    | taskmanager.memory.task.heap.size          |
|  TaskOffHeapMemory  |  Task Off-Heap Memory  | taskmanager.memory.task.off-heap.size      |
| FrameworkHeapMemory | Framework Heap Memory  | taskmanager.memory.framework.heap.size     |
| FrameOffHeapMemory  | Frame Off-Heap Memory  | taskmanager.memory.framework.off-heap.size |
|    NetworkMemory    |     Network Memory     | NA                                         |
|    ManagedMemory    |     Managed Memory     | taskmanager.memory.managed.size            |
|   OverHeadMemory    |  JVM Overhead Memory   | NA                                         |


如下是结合源码（[Flink1.10.0版本的TaskExecutorProcessUtils#processSpecFromConfig方法](https://github.com/apache/flink/blob/release-1.10.0/flink-runtime/src/main/java/org/apache/flink/runtime/clusterframework/TaskExecutorProcessUtils.java)）分析各部分内存的计算方法。其核心逻辑是根据配置好的一个或多个参数推测出其他各部分内存大小。根据指定参数的情况，分为如下三个场景（判断有先后之分，不满足场景一才会继续判断是否满足场景二，不满足场景二才会去判断是否满足场景三）

|  场景  |               场景条件                |
| :----: | :-----------------------------------: |
| 场景一 | 同时配置TaskHeapMemory和ManagedMemory |
| 场景二 |            配置FlinkMemory            |
| 场景三 |            配置TotalMemory            |

场景一：已知TaskHeapMemory和ManagedMemory
---------------------------------------

| 组件内存               | 计算方法                                           | 默认值 |
| ---------------------- | -------------------------------------------------- | ------ |
| TaskHeapMemory         | 通过taskmanager.memory.task.heap.size配置项获取    | NA     |
| ManagedMemory          | 通过taskmanager.memory.managed.size配置获取        | NA     |
| FrameworkHeapMemory    | 通过taskmanager.memory.framework.heap.size配置获取 | 128M   |
| FrameworkOffHeapMemory | 通过taskmanager.memory.framework.off-heap.size获取 | 128M   |
| TaskOffHeapMemory      | 通过taskmanager.memory.task.off-heap.size获取      | 0      |
| MetaspaceMemory        | 通过taskmanager.memory.jvm-metaspace.size获取      | 96M    |

### NetworkMemory的计算：

先计算一个出Network之外的内存之和`totalFlinkExcludeNetworkMemorySize`

```java
totalFlinkExcludeNetworkMemorySize = TaskHeapMemory + TaskOffHeapMemory + ManagedMemory + FrameworkHeapMemory + FrameworkOffHeapMemory 
```

* 如果设置了FlinkMemory (由参数taskmanager.memory.flink.size控制)：

  ```java
  NetWorkMemory = FlinkMemory - totalFlinkExcludeNetworkMemorySize
  ```

* 如果配置了taskmanager.network.numberOfBuffers 并且没有配置taskmanager.memory.network.min,taskmanager.memory.network.max,taskmanager.memory.network.fraction

  ```java
  // 默认是32Kb* 2048
  NetworkMemory = taskmanager.memory.segment-size * taskmanager.network.numberOfBuffers
  ```

* 否则

  ```java
  // fraction 默认0.1
  fraction = taskmanager.memory.network.fraction
  NetworkMemory = totalFlinkExcludeNetworkMemorySize * (fraction/(1-fraction))
  // 最终计算出来的NetWorkMemory的值应当在范围内[taskmanager.memory.network.min,taskmanager.memory.network.max]，否则会被截取
   ```

### OverHeadMemory的计算

* 如果有配置TotalMemory(通过`taskmanager.memory.process.size`配置)

  ```java
  OverHeadMemory = TotalMemory - FlinkMemory - MetaspaceMemory
  ```

* 否则

  ```java
  // fraction默认0.1
  fraction = taskmanager.memory.jvm-overhead.fraction
  OverHeadMemory = （FlinkMemory + MetaspaceMemory） * (fraction/(1-fraction))
  // 最终计算出来的NetWorkMemory的值应当在范围内[taskmanager.memory.jvm-overhead.min,taskmanager.memory.jvm-overhead.max]，否则会被截取
  ```

场景二：已知FlinkMemory
----------------

| 组件内存               | 计算方法                                           | 默认值 |
| ---------------------- | -------------------------------------------------- | ------ |
| FrameworkHeapMemory    | 通过taskmanager.memory.framework.heap.size配置获取 | 128M   |
| FrameworkOffHeapMemory | 通过taskmanager.memory.framework.off-heap.size获取 | 128M   |
| TaskOffHeapMemory      | 通过taskmanager.memory.task.off-heap.size获取      | 0      |
| MetaspaceMemory        | 通过taskmanager.memory.jvm-metaspace.size获取      | 96M    |

### TaskHeapMemory,ManagedMemory,NetworkMemory的计算

#### 如果有配置TaskHeapMemory

* TaskHeapMemory的计算

   通过配置`taskmanager.memory.task.heap.size`获取

* ManagedMemory的计算

   * 如果有配置`taskmanager.memory.managed.size`，从该配置获取

   * 否则
    
     ```java
     //// fraction默认0.4
     fraction = taskmanager.memory.managed.fraction
     NetWorkMemory = FlinkMemory * fraction
     // 最终计算出来的NetWorkMemory的值应当在范围内[0,Long.MAX_VALUE]，否则会被截取
       ```

* NetworkMemory的计算
  
  ```java
  NetworkMemory = FlinkMemory - TaskHeapMemory - TaskOffHeapMemory - FrameworkHeapMemory - FrameworkOffHeapMemory - ManagedMemory
  ```
  

#### 如果没有配置TaskHeapMemory

* Managed Memory的计算：

  * 如果有配置`taskmanager.memory.managed.size`，从该配置获取

  * 否则

    ```java
    //// fraction默认0.4
    fraction = taskmanager.memory.managed.fraction
    ManagedMemory = FlinkMemory * fraction
    // 最终计算出来的NetWorkMemory的值应当在范围内[0,Long.MAX_VALUE]，否则会被截取
    ```

* NetworkMemory的计算：

  * 如果配置了taskmanager.network.numberOfBuffers 并且没有配置taskmanager.memory.network.min,taskmanager.memory.network.max,taskmanager.memory.network.fraction
  
    ```java
    // 默认是32Kb* 2048
    NetworkMemory=taskmanager.memory.segment-size * taskmanager.network.numberOfBuffers 
    ```
  
  * 否则
  
    ```java
    // fraction 默认0.1
    fraction = taskmanager.memory.network.fraction
    NetworkMemory = FlinkMemory * fraction
    // 最终计算出来的NetWorkMemory的值应当在范围内[taskmanager.memory.network.min,taskmanager.memory.network.max]，否则会被截取
    ```
  
* TaskHeapMemory的计算

  ```java
  TaskHeapMemory = FlinkMemory - NetworkMemory - ManagedMemory - TaskOffHeapMemory - FrameworkHeapMemory - FrameworkOffHeapMemory
  ```

#### OverHeadMemory内存计算

- 如果有配置 TotalMemory（通过`taskmanager.memory.process.size`配置）

  ```java
  OverHeadMomory = TotalMemory - FlinkMemory - MetaspaceMemory
  ```

- 否则

  ```java
  // fraction默认0.1
  fraction = taskmanager.memory.jvm-overhead.fraction
  NetWorkMemory = （totalFlinkExcludeNetworkMemorySize + NetWorkMemory） * (fraction/(1-fraction))
  // 最终计算出来的NetWorkMemory的值应当在范围内[taskmanager.memory.jvm-overhead.min,taskmanager.memory.jvm-overhead.max]，否则会被截取
  ```


场景三:配置TotalMemory
---------------------------

| 组件内存               | 计算方法                                           | 默认值 |
| ---------------------- | -------------------------------------------------- | ------ |
| FrameworkHeapMemory    | 通过taskmanager.memory.framework.heap.size配置获取 | 128M   |
| FrameworkOffHeapMemory | 通过taskmanager.memory.framework.off-heap.size获取 | 128M   |
| TaskOffHeapMemory      | 通过taskmanager.memory.task.off-heap.size获取      | 0      |
| MetaspaceMemory        | taskmanager.memory.jvm-metaspace.size              | 96M    |

#### OverHeadMemory内存计算

  ```java
  //fraction默认为0.1
  fraction = taskmanager.memory.jvm-overhead.fraction
  OverHeadMemory = TotalMemory * fraction
  // 最终计算出来的NetWorkMemory的值应当在范围内[taskmanager.memory.jvm-overhead.min,taskmanager.memory.jvm-overhead.max]，否则会被截取
  ```

#### TaskHeapMemory,ManagedMemory,NetworkMemory的计算

##### 如果有配置TaskHeapMemory

* TaskHeapMemory通过配置`taskmanager.memory.task.heap.size`获取

* ManagedMemory内存计算

  * 如果有配置ManagedMemory(通过`taskmanager.memory.managed.size`配置)，从该配置获取

  * 否则
  
    ```java
    //// fraction默认0.4
    fraction = taskmanager.memory.managed.fraction
    ManagedMemory = （TotalMemory - MetaspaceMemory - OverHeadMemory） * fraction
    // 最终计算出来的NetWorkMemory的值应当在范围内[0,Long.MAX_VALUE]，否则会被截取
    ```

* NetworkMemory的计算

   ```java
  NetworkMemory = TotalMemory - MetaspaceMemory - OverHeadMemory - ManagedMemory - TaskHeapMemory - TaskOffHeapMemory - FrameworkOffHeapMemory - FrameworkHeapMemory
   ```


##### 如果没有配置TaskHeapMemory

* ManagedMemory内存计算
   * 如果有配置`taskmanager.memory.managed.size`，从该配置获取

   * 否则
   
     ```java
      //// fraction默认0.4
      fraction = taskmanager.memory.managed.fraction
      Managed Memory = （TotalMemory - MetaspaceMemory - OverHeadMemory） * fraction
      // 最终计算出来的NetWorkMemory的值应当在范围内[0,Long.MAX_VALUE]，否则会被截取
     ```

* NetworkMemory内存计算
  
   * 如果配置了taskmanager.network.numberOfBuffers 并且没有配置taskmanager.memory.network.min,taskmanager.memory.network.max,taskmanager.memory.network.fraction
   
     ```java
     // 默认是32Kb* 2048
     NetworkMemory=taskmanager.memory.segment-size * taskmanager.network.numberOfBuffers 
     ```
   
   * 否则：
   
     ```java
     // fraction 默认0.1
     fraction = taskmanager.memory.network.fraction
     NetworkMemory = (TotalMemory - MetaspaceMemory - OverHeadMemory）* fraction
     // 最终计算出来的NetWorkMemory的值应当在范围内[taskmanager.memory.network.min,taskmanager.memory.network.max]，否则会被截取
     ```

* TaskHeapMemory 内存计算
  
  ```java
  TaskHeapMemory = TotalMemory - MetaSpaceMemory - OverHeadMemory - NetworkMemory - ManagedMemory - TaskOffHeapMemory - FrameworkOffHeapMemory - FrameworkHeapMemory
  ```
  
