---
layout: post
title:  "让JVM进程退出正确姿势"
date:   2018-12-29 16:17:12 +0800
tags:
      - Others
---

System.exit的在系统开发中常备开发人员调用以用于退出应用。然而一个不合理的调用位置可能导致JVM无法正常退出。

### JVM无法正常退出问题

今天小编就遇到一个这样的问题：问题的原因是这样的：应用在运行时，main线程抛出异常，退出，但其他线程并未退出，导致JVM进程依旧存活，但无法提供服务。查看代码发现

    ....
    service.startAsync().awaitTerminated();
    system.exit(1)
    ....
通常awaitTerminated/awaitTermination等方法用于主线程来监控整个系统运行的状态，如果发现异常，则进程会自动退出。但是如上的逻辑中即调用了waitTerminated方法，又调用了System.exit方法，进程却还是没有退出成功，这就尴尬了。。。

### JVM无法退出的分析

于是查看代码发现在awaitTerminated方法中当检测到某些异常（通常时其他线程或者服务在遇到一些致命问题时，设置的标志位）之后，直接抛出Exception异常，导致System.exit(1)没有被调用到，再加上服务中有部分线程并非daemon级别，因此进程无法退出。

### 退出JVM的相关测试

可参考如下代码：

    object TermitatedTest {
      val t2 = new ThreadExample
      private def throwException = throw new Exception("Exit in Main")
      def main(args: Array[String]): Unit = {
    //    t2.setDaemon(true) // 设置线程为daemon级别的线程
        t2.start()
        Thread.sleep(1000)
    //    throwException    // 主线程中抛出异常：将引发主线程后续的代码不会执行，即使System.exit（1）
        println("Trying to print")
        System.exit(1)
      }
    }
    class ThreadExample extends Thread{
      override def run(){
        println("Thread is running");
        Thread.sleep(1000000)
        println("Tread runned")
      }
    }


* 在throwException行注释掉之后：
 
        无法t2线程是否为daemon级别，主线程运行完后，将直接退出

* 在throwException行没有注释时:
 
        throwException后的代码不会执行
        t2设置为damon级别，主线程抛出异常后，进程立刻退出
        t2设置为非daemon级别，主线程抛出异常后，t2线程依然运行，JVM不会退出，直到t2运行完毕，JVM退出



### JVM进程退出的建议：

对于一个单线程（除JVM必要的系统线程外只有一个线程）的简单应用来说，JVM会随着线程的运行结束，运行异常等情况退出JVM进程。但是对于一个多线程多服务的进程来说，一个线程的退出（即使这个线程是main线程）通常不会导致进程退出。但对于一个这样的系统服务JVM进程该如何退出呢


1 ：直接调用System.exit方法完成JVM的退出

2 ：启动服务/线程时，非核心线程应当线程设置为daemon，并将服务/线程的清理放入JVM的shutdownhook中，在JVMcatch到只有系统线程和daemon线程后，会调用注册过的hook函数，完成服务/线程的清理，进而退出JVM进程

3: 在一个大型的应用中：main函数中使用awaitterminatation机制来观察等待等个应用的运行，在需要终止的时候，结束等待或抛出异常。

4: 在使用第三方框架需要将其启动的线程设置为daemon级别。如使用akka时设置akka.daemonic=on,则akka启动的线程均为daemon级别的线程。


PS: 附一条查看JVM进程中damon线程的方法 ： 
 
     jstack ${pid} | grep "tid" | grep "os_prio" | grep -v grep | grep daemon
