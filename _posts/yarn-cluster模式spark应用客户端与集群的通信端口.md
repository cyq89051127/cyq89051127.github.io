
---
layout: post
title:  "yarn-cluster模式spark应用客户端与集群的通信端口"
date:   2018-06-21 13:20:12 +0800
tags:
      - Spark
---

Spark应用在on yarn模式下运行，需要打开集群中的节点的端口以便完成应用的提交和运行。下面针对yarn-cluster模式下提交spark应用需要的集群端口进行测试。


## 非安全集群场景下

### 测试结论：

集群外节点yarn-clsuter模式下提交spark应用，需要连接ResourceManager完成app的提交，同时也需要上传部分文件到hdfs以供container使用。因此至少需要开通ResourceManager，NameNode，DataNode等节点与客户端的通信端口。

非安全模式下需要对客户端开通的几个端口：


服务 | 使用原因 | 参数 | 端口默认值 | 端口协议
---|--- | ---- | --- | -----
ResourceManager | 提交app时与RM通信 |yarn.resourcemanager.address | 8050 | TCP
Namenode | 上传文件至hdfs时与NN通信 |dfs.namenode.rpc-address | 8020  | TCP
DataNode | 上传文件时与DN通信 | dfs.datanode.address| 50010 | TCP




### 测试方法：


#### 准备工作

    * 准备一个三节点hadoop集群（节点ip分别设为ip1,ip2,ip3）和一个集群外节点(ip_external)  
    * 在ip_external创建创建hadoop_conf目录（将集群中$HADOOP_HOME/conf目录下文件copy至hadoop_conf目录）
    * 安装spark客户端（将集群中的$SPARK_HOME下文件copy至集群外ip_external节点），在spark-env.sh中配置HADOOP_CONF_DIR并指向hadoop_conf目录



#### 测试步骤

- 屏蔽集群中节点对客户端的所有端口

通过在集群中各节点执行如下命令将集群中节点通过iptables设置对ip_external的所有端口（用户端口）均屏蔽

    iptables -A INPUT -s ip_external -p tcp --dport 1025:65535  -j DROP

此时以cluster模式提交spark应用，有如下打印，长时间无法提交应用

    Retrying connect to server: ip3:8050. Already tried 1 time(s); maxRetries=45
    ......
    Retrying connect to server: ip3:8050. Already tried 44 time(s); maxRetries=45
    Retrying connect to server: ip3:8050. Already tried 0 time(s); maxRetries=45

异常分析：

    在hdp集群中，8050的ResourdeManager端口（yarn.resourcemanager.address参数控制，默认8050），显然相关打印是由于端口被屏蔽，导致客户端无法和ResourceManager通信导致。

- 打开ResourceManager节点对客户端节点的端口

登录ipv3节点，执行如下命令，打开8050端口与ip_external的通信

    //清理iptables设置
    iptables -F
    //添加iptables设置
    iptables -A INPUT -s ip_external -p tcp --dport 8050  -j ACCEPT
    iptables -A INPUT -s ip_external -p tcp --dport 1025:65535  -j DROP

再次提交spark应用，有如下打印，长时间无法提交应用

    Retrying connect to server: ip2:8020. Already tried 1 time(s); maxRetries=45
    ......
    Retrying connect to server: ip2:8020. Already tried 44 time(s); maxRetries=45
    Retrying connect to server: ip2:8020. Already tried 0 time(s); maxRetries=45

异常分析：

    在hdp集群中，8020的NameNode端口（dfs.namenode.rpc-address参数控制，默认8020），显然相关打印是由于端口被屏蔽，导致客户端无法和NameNode通信导致。

- 打开NameNode节点对客户端节点的端口

登录ip2节点，执行如下命令，打开8020端口与ip_external的通信

    //清理iptables设置
    iptables -F
    //添加iptables设置
    iptables -A INPUT -s ip_external -p tcp --dport 8020  -j ACCEPT
    iptables -A INPUT -s ip_external -p tcp --dport 1025:65535  -j DROP


再次提交spark应用，抛出如下异常，提交失败

    RemoteException(java.io.IOException): File /user/abc/.sparkStaging/application_1521424748238_0198/spark-assembly-1.6.2.2.5.0.0-1245-hadoop2.7.3.2.5.0.0-1245.jar could only be replicated to 0 nodes instead of minReplication (=1).  There are 3 datanode(s) running and 3 node(s) are excluded in this operation.
    
异常分析：

    该异常显示连接datanode失败，无法将文件上传至datanode，引发失败。客户端与datanode通信端口（dfs.datanode.address参数控制，默认50010）

- 打开DataNode节点对客户端节点的端

分别登录ip1,ip2,ip3三个节点，打开50010的端口与客户端的通信。同样是先清理iptable配置，再添加，参考如上操作即可。

此时提交spark应用，成功运行。


## 安全集群场景下


测试结论：

安全集群场景下除了打开以上端口外，还需要关注安全认证的kdc服务端口，以及可能的timelineservice服务的端口。

安全模式下需要对客户端开通的几个端口：

服务 | 使用原因 | 参数 | 端口默认值 | 端口协议
---|--- | ---- | --- | ---
kdc | 与集群交互前需要在kdc进行认证用户信息 |kdc_ports| 88 | UDP
ResourceManager | 提交app时与RM通信 |yarn.resourcemanager.address | 8050 | TCP
TimeLineService | 安全模式下，需要客户端获取TImeLineService服务的token |yarn.timeline-service.webapp.address  | 8188 | TCP
Namenode | 上传文件至hdfs时与NN通信 |dfs.namenode.rpc-address | 8020  | TCP
DataNode | 上传文件时与DN通信 | dfs.datanode.address| 50010 | TCP

### 测试方法：


#### 准备工作

    * 准备一个三节点安全hadoop集群（节点ip分别设为ip4,ip5,ip6）和一个集群外节点(ip_external) ，hadoop集群的kdc节点的ip为ip_kdc
    * 在ip_external节点创建hadoop_conf_security目录（将集群中$HADOOP_HOME/conf目录下文件copy至hadoop_conf目录）
    * 安装spark客户端（将集群中的$SPARK_HOME下文件copy至集群外ip_external节点），在spark-env.sh中配置HADOOP_CONF_DIR并指向hadoop_conf_security目录
    * 将集群的/etc/security/keytab下要用于提交应用的keytabf文件copy至ip_external节点相关目录下，和/etc/krb5.con文件copy至ip_external节点的/etc目录下

测试步骤

根据非安全集群下的测试情况，首先打开所需rm,nn,dn相关端口

- 先不对kdc几点做处理，提交应用，应用提交成功

- 屏蔽kdc节点的所有端口

    登录登录kerberos节点，关闭所有端口与ip_external节点的通信，使用如下命令（由于kdc服务默认使用的是88端口，直接关闭1：65536）
    
        iptables -A INPUT -s ip_external -p tcp --dport 1:65535  -j DROP
    
    提交spark应用，应用正常运行。这与预期不一致。
    
    分析： 在安全集群中所有与集群中安全的服务通信的进程（客户端）都需要完成与kdc认证之后才能访问集群中的服务，在关闭端口与客户端通信之后，居然可以完成认证而且还可以提交运行应用，比较诡异。

    再次登录kdc节点,使用netstat -anp | grep 88 查看到如下信息：
    
        tcp        0      0 0.0.0.0:88              0.0.0.0:*               LISTEN      20414/krb5kdc
        
        udp        0      0 0.0.0.0:88              0.0.0.0:*                           20414/krb5kdc 
    说明此服务同时支持tcp和udp协议，猜测客户端进行kerberos认证是通过udp协议完成

-  屏蔽kdc节点的udp协议的88端口

        iptables -A INPUT -s ip_external -p tcp --dport 88  -j DROP
    再次提交spark应用，客户端卡主，无法完成应用的提交，说明需要打开kdc服务使用端口的udp服务，客户端才能提交应用

- 打开kdc节点的udp协议的88端口

    再次提交spark应用，应用提交失败出现如下异常： 
    
        18/06/21 16:26:11 INFO impl.TimelineClientImpl: Exception caught by TimelineClientConnectionRetry, will try 30 more time(s).
        Message: connect timed out
        18/06/21 16:27:12 INFO impl.TimelineClientImpl: Exception caught by TimelineClientConnectionRetry, will try 29 more time(s).
        ...
    
    该异常显示timelineClient和timeLineService服务通信时异常，引发失败，通信端口（yarn.timeline-service.webapp.address参数控制，默认8188）

-  打开timelineservice 服务节点的8188端口
    由于本集群的timelineService服务和RM服务在同一节点，登录该节点：

        // 清理原有的iptables设置
        iptables -F
        //添加iptables设置
        iptables -A INPUT -s ip_external -p tcp --dport 8188  -j ACCEPT
        iptables -A INPUT -s ip_external -p tcp --dport 1019  -j ACCEPT
        iptables -A INPUT -s ip_external -p tcp --dport 8050  -j ACCEPT
        iptables -A INPUT -s ip_external -p tcp --dport 1:65535  -j DROP
    此时提交应用，运行正常。




### 安全和非安全环境下对timelineservice服务的端口访问要求不同的原因

查看yarnclientImpl 源码发现在submitApplication的函数中，有如下逻辑：

    //根据集群安全性和timeLineService情况判断是否需要为am添加timeLine服务的token
    if (isSecurityEnabled() && timelineServiceEnabled) {
      addTimelineDelegationToken(appContext.getAMContainerSpec());
    }
    //该函数中会调用到
    timelineDelegationToken = getTimelineDelegationToken();
    
继续跟踪代码可以发现在安全模式下，与其他服务（hdfs，hive，hbase）的默认的访问机制一样，客户端会先获取timelineservicetoken并存入hdfs目录， 以供applicationMaster运行时使用。

由于只有在安全模式下，才可能使用此逻辑，因此在安全和非全的集群中，使用cluster模式提交应用，需要关注一下集群的timelineService服务。
