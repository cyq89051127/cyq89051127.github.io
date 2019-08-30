---
layout: post
title:  "linux服务器的物理CPU，CPU核数，逻辑CPU及Hadoop的Vcore"
date:   2018-10-30 12:19:12 +0800
tags:
      - Others
---
## Linux服务器的核数的概念

    物理CPU： 服务器上真实存在的CPU，可以看到
    CPU的核 (core)： 一个CPU上包含多少核(core)，真实存在但不能直接看到

* 总核数 = 物理CPU个数 X 每颗物理CPU的核数 

* 总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X 超线程数 

      在没有开启超线程时，总核数 = 总逻辑CPU个数，如果开启超线程，则总核数  < 总逻辑CPU数

## 在linux服务器节点上，可直接通过如下命令查看相关配置：

* 查看物理CPU个数
        
      cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
* 查看每个物理CPU中core的个数(即核数)

      cat /proc/cpuinfo| grep "cpu cores"| uniq
* 查看逻辑CPU的个数
        
      cat /proc/cpuinfo| grep "processor"| wc -l


## Vcore的概念及使用
 在HADOOP平台中，涉及资源管理/分配的的服务会对CPU进行资源划分和管理（分配，回收）以Yarn服务为例：

### 如果单节点可供yarn管理的资源有128G内存，16个逻辑CPU（也就是16core）：

在申请/分配/启动container时，如果单个container申请资源4G，1core，则单节点只能分配 max(128/4,16/1)=16个container 引起半数（128-16*4）的内存浪费


yarn引入了Vcore的概念作为对cpu core的逻辑封装，将节点的"设置"为32 Vcore（由于这里不是真正的Core，因此使用Vcore这个名字，通常Vcore的个数赢设置为cpu core总数的1-5倍），在申请/分配/启动container时，使用Vcore代替core，单container申请资源为4，1Vcore， 则单节点可以分配max(128/4,32/1)=32个container，则不造成资源的浪费

节点资源 | 管理/分配资源的单位 | container申请资源 | 可启动container数目 | container使用资源 | 无法真正管理的资源的资源
---|--- | ---| ---| -----|----|
128G，16 Core | 内存，core | 4G，1Core |  min(128/4,16/1)=16 | 64 G，16Core | 64G，0Core |
128G，32V Core | 内存，Vcore | 4G，1VCore |  min(128/4,32/1)=32 | 128 G，32 Vcore | 0G，0Vcore |

### Vcore的个数如何设置

从以上分析可知Vcore设置的大小将直接影响单个NodeManager节点上启动的container个数，而container启动之后执行的业务逻辑对cpu资源的消耗是不定的，集群资源的管理者无法感知。

#### 假定节点内存资源充足

* 如果container运行业务是cpu密集型的逻辑
    
      在设置较多的Vcore意味着启动较多的container进程，将导致节点CPU使用率飚高，计算性能将受到影响。
    此场景下应当设置较少的Vcore数

* 如果container运行的业务是CPU不敏感型的业务逻辑

      设置较少的Vcore意味着启动较少的container进程，虽然运行起来没有问题，但是节点的资源使用率过低，无法充分利用物理资源
    
    此场景下可以设置较多的Vcore数
#### 如何判断业务是否是CPU密集型的执行逻辑

对于具体的应用来说，业务是否是CPU密集型的业务的开发人员应当是清楚的。但作为集群的管理者，集群中运行的应用成千上万，花样百出，也就无法直接判断应用的CPU消耗情况，那我们应该如何设置Vcore数呢？

* 一般大数据集群在规划部署时：

    需要确认集群资源以及在Yarn服务中配置单节点的Vcore数目，通常此时Vcore数目设置为物理节点的逻辑CPU总数的2倍

* 在集群运行过程中：
    
观察节点的CPU使用情况以及Vcore消耗情况：    
 
      节点CPU使用情况：可以登录后台使用top命令查看或者登录集群页面的监控查找
      节点Vcore的消耗情况：登录Yarn服务原生页面，点击左侧Nodes标签页，可查看各节点的资源消耗情况

   ![nodes.jpg](https://upload-images.jianshu.io/upload_images/9004616-dd3a4a48952aea0c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    如果Vcore使用率较高的情况下，节点的CPU使用率很低，则可以提升节点的Vcore数
    如果Vcore使用率较低的情况下，节点的CPU使用率很高，则需要降低节点的Vcore数


###### PS： 对于生产集群来说： Vcore数的调整对于集群来说是个高危的操作，不能单纯的根据一个节点的情况，或者一次的观察情况就修改节点的Vcore数目，需要对集群资源管理机制的深入理解，以及广泛的长期的观察的基础上做出判断
