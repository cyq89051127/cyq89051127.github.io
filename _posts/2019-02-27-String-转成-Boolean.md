---
layout: post
title:  "String-转成-Boolean"
date:   2019-02-27 11:17:26 +0800
tags:
      - Others
---

* SDK中String => Boolean的转换

    Scala中把String类型转化为Boolean类型可以调用toBoolean方法即可

    “true”.toBoolean
 
 * JDK中String => Boolean的转换
 
    在Java中就不太明显，容易掉坑了，Java中Boolean有getBoolean和parseBoolean两个方法：
    
    API | 功能 | 示例 | 
    ---|--- | ----
    ParseBoolean | 直接将将字符串转为为Boolean，字符串为true时，返回true，否则返回false | Boolean.parseBoolean("true")
    getBoolean   | 在系统中查找字符串对应的value，找到且未true时，返回true，否则返回false | 设置系统变量 -Dxyz=true 然后再代码中调用 Boolean.getBoolean("xyz")

     如上API要看明白再使用（查看下源码实现其实很容易明白了），否则可能无法得到预期结果。笔者就曾使用getBoolean方法错误，耗费了一定精力才找到问题所在。
* 第三方JAVA API的String => Boolean

    以上是利用JDK/SDK提供的转换能力，如果引入第三方的框架时，则有更强大的转换能力了，
    
    如commons-lang中的StringUtils可以将y,yes,on,true,t等字符串转化为字符串true的能力，将n,no,off,false,f等转换为false的能力。
    
    同时该API提供了int与Boolean互转的方法，

    
