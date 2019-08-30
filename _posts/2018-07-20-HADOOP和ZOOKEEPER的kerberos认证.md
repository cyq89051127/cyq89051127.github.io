---
layout: post
title:  "HADOOP和ZOOKEEPER的kerberos认证"
date:   2018-07-20 10:14:12 +0800
tags:
      - Zookeeper
---

安全场景下的大数据集群的使用相比非安全的集群具有更高的复杂度。关于kerberos认证的讲解资料较多，本文不在赘述。仅针对HADOOP和ZOOKEEPER的的认证进行简单分析和对比并给出一些kerberos访问的一下Tip。

### 安全HADOOP的使用：

使用过安全hadoop的小伙伴都知道在安全集群中，如果需要对集群（hdfs,yarn,hive,hbase，spark thriftserver）的访问，都需要先进行认证：

    1. 在命令行/测试场景下一般使用kinit命令完成用户的认证，然后“任意”访问集群中的组件
    2. 在线上应用中一般使用UserGroupInformation.loginUserFromKeytab(principalName, keytabFilename)来完成kerberos认证

### 安全HADOOP的访问的实现：

HADOOP的安全认证与狭义的hdfs，yarn实现了分离，独立封装出hadoop-common*.jar给上层各组件使用，应用层调用UserGroupInformation类完成认证

    /*直接成longinContext*/
    login = newLoginContext(HadoopConfiguration.KEYTAB_KERBEROS_CONFIG_NAME,subject, new HadoopConfiguration());
    login.login();
    
    /*在loginContext的init方法中获取登录信息，此时的config为hadoop实现的Config*/
    AppConfigurationEntry[] entries = config.getAppConfigurationEntry(name);
    
    /*此类中定义了keytab，principal，loginModule等信息，此处使用的longinModule为Krb5LoginModule*/
    private static final AppConfigurationEntry KEYTAB_KERBEROS_LOGIN =
      new AppConfigurationEntry(KerberosUtil.getKrb5LoginModuleName(),
                                LoginModuleControlFlag.REQUIRED,
                                KEYTAB_KERBEROS_OPTIONS);



### 安全Zookeeper的使用

我们知道在进行安全Zookeeper访问时，需要进行kerberos认证，而Zookeeper认证时需要传入对应的jaas.conf配置文件。通常该文件的配置如下：

    Client {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/etc/zookeeper/conf/zkcli.keytab"
      storeKey=true
      useTicketCache=false
      principal="zkcli@HADOOP.COM";
    };
该配置的含义是在进行zookeeper认证时，使用的认证类时Krb5longinModule类，使用keytab文件认证，认证时不使用OS缓存的tgt，而是使用配置的keyTab和principal。 配置文件中的Client表示认证的上下文的段（Login Context section），应用层如果需要再不同的服务访问时，使用不同的用户，则可以配置多个不同的Login Context section。 只要实现对该文件的解析逻辑即可。

安全zookeper场景下的认证实现：

    /*此时的Configuration.getConfiguration返回的是sun.security.provider.ConfigFile，通过该类和jaas.conf配置文件获取认证信息*/
    
    Configuration.getConfiguration().getAppConfigurationEntry(clientSection)
    
    /*在初始化zookeeper的封装login类初始化时，调用login方法，使用loginContext类完成用户的kerberos的认证*/
    
    LoginContext loginContext = new LoginContext(loginContextName,callbackHandler);
    loginContext.login();
    LOG.info("successfully logged in.");
    
### 总结： 

#### HADOOP和zookeeper的的认证本质是一致的

在安全认证时，均使用的是kerberos的认证机制，客户端的认证实现都是通过底层jdk的代码LonginContext，Krb5longinModule等模块实现，本质上来讲，HADOOP和zookeeper的认证是一回事。

#### 认证信息的解析方式不同

zookeeper的认证通过解析应用配置的jaas.conf配置文件得到相关的信息完成

而HADOOP的认证模块则是内部定义了部分配置，只需要用户传入principal和keytab即可。

#### HADOOP和zookeeper为何没有使用同一的接口实现

zookeeper为何不使用HADOOP的认证方式的可能原因 ：HADOOP最初的实现没有引入相关的安全机制；    Zookeeper作为底层的分布式协调器，不应当对上层的组件有依赖；作者不想用。

HADOOP为何不使用zookeeper的认证方式的可能原因：jaas.conf的配置相对复杂，作者用起来不爽


#### 各个组件使用的认证方式

组件如Hive,HBase，Spark等组件使用的认证都是hadoop-common的实现

Kafka和zookeeper使用的均为配置jaas.conf的方式进行kerberos认证。大概是kafka本身对hadoop没有依赖关系，仅仅对zk有相关的依赖，为了保持统一风格，因此选择了Zookeeper的认证
    


#### 关于kerberos的一些Tips

KRB5CCNAME环境变量：

    在客户端与kdc进行认真时会生成tgt文件，默认会缓存在/tmp/krb5cc_${uid}的文件中，如果使用不同的终端同样的用户登录机器分别使用kinit命令，则会造成用户冲突，可能引发各种异常。因此可通过设置KRB5CCNAME环境变量的方式进行认证时用户隔离

KRB5_CONFIG环境变量：

    * 在客户端与kdc交互时，使用使用到krb5.conf的配置文件，该配置文件默认存储在/etc目录下。 如果执行kinit等命令时需要使用特定的krb5.conf配置文件，可通过设置KRB5_CONFIG实现
    
    * 同样如果是在java进程中需要使用特定的krb5.conf配置文件，则可通过设置-Djava.security.krb5.conf来完成


操作 | 功能 | 备注
---|---|---
export KRB5CCNAME=/path/${tgtcacheFileName} | 指定私有的tgt缓存文件，与其他tgt文件隔离，当前session有效，不影响其他用户使用
export KRB5_CONFIG=/path/krb5.conf | 执行kinit,klist命令行时，指定特定的krb5.conf配置文件 | 
-Djava.security.krb5.conf=/path/krb5.conf | java命令行中指定特定的krb5.conf配置文件
-Djava.security.auth.login.config=/path/jaas.conf | 指定特定的jaas.conf配置文件 
