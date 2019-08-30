---
layout: post
title:  "mvn编译异常“was-cached-in-the-local-repository,-resolution-will-not-be-rea"
date:   2018-03-13 11:27:12 +0800
tags:
      - Others
---

### 问题
最近编译livy-release工程，各种异常，加入hdp的relase库之后，出现了找不到jetty-ssslengine,jetty-util,jetty的1.26.1.hwxjar包和pom文件异常。从其他机器的mvn仓库copy一份放入本地仓库后，编译出现上述异常。

### 尝试解决方案：
根据网上搜索的解决方案有如下：

    删除lastupdate”为后缀的临时文件
    删除相关目录
    使用mvn -U命令
    配置相关仓库的 <updatePolicy>always</updatePolicy>

均无法解决问题。

由于找不到（不知道）哪个仓库下有这些jar包。因此也没有办法配置可以找到这些jar包的仓库

### 解决办法：

    将从其他机器copy来的本地仓库的相关jar包目录下文件如pom，sha256的check文件全部删除，只保留jar包文件。
    编译成功
