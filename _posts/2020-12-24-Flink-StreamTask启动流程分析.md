---
layout: post
title:  "StreamTask OperatorChain分析"
date:   2020-12-24 10:15:12 +0800
tags:
      - Flink
---

Flink的作业StreamTask是任务执行的核心，其执行的本质即为各个operator的执行，而operator之间又有前后依赖关系，各operator构成一条链条（Chain），各operator顺序执行，完成业务逻辑的执行。如下我们以WordCount为例分析其作业执行(WordCount)源码可参考:

[WordCount.scala](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/wordcount/WordCount.scala)



其业务代码逻辑如下：

```scala
// 读取数据源
val text = env.fromElements(WordCountData.WORDS: _*)
// 业务逻辑
 val counts: DataStream[(String, Int)] = text
      // split up the lines in pairs (2-tuples) containing: (word,1)
      .flatMap(_.toLowerCase.split("\\W+"))
      .filter(_.nonEmpty)
      .map((_, 1))
      // group by the tuple field "0" and sum up tuple field "1"
      .keyBy(0)
      .sum(1)
// 打印结果
counts.print()
```

在Flink的运行时，以上代码逻辑会被解析为两个streamTask（根据设置的并行度每个task可能有多个subTask）：

> // 读取数据并进行业务操作  （SourceTask）
>
> Source: Collection Source -> Flat Map -> Filter -> Map (1/1)
>
> // 汇总分析并输出（SinkTask）
>
> aggregation -> Sink: Print to Std. Out (1/1)

如下以此逻辑为例，分析operatorChain的生成和task的执行。



OperatorChain的的生成过程的核心逻辑是：

> 依次将生成operatorWrapper（对operator的封装）,并设置operatorWrapper的先后关系。



#### 读取数据并进行业务操作Task（SourceTask）

SourceTask生成operatorChain的流程如下图所示：



![OperatorChain生成步骤](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE8500f1f996f38d3b15ddd89cd37468fc/9704)



> 从图片中可以看出由headOperator（其实就是Source）开始，先生存该operator的output，然后生成其opeator，并封装为operatorWrapper
>
> 尤其output的生成是递归调用，output中会包含下一个operator，因此operatorchain的生成是一个递归调用。



task的执行过程（数据如何在operator间传递）

作业执行堆栈如下图所示：

![线程执行堆栈](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE4265303c56126bf5c2e5db221a9bfac4/9707)

从堆栈来看task的执行依赖的是operator的执行逻辑，每个operator在调用完函数完成处理后，会调用其output往下一个operator发送当前operator的处理结果，在最后一个operator处理后，会交给其output（对应的writer）完成结果输出或者发送给后续的task。如下：

![task执行](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/FC82B4A91F56463EAB67D438D0075E9E/9709)

#### 汇总分析和输出Task（SinkTask）

SinkTask生成operatorChain的流程如下图所示：

![Task的operatorChain生成过程](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE266a345f38d1fdcf227fcb2fa5a50e64/9713)



SinkTask的执行任务线程堆栈如下：

![Task运行时堆栈](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE585157a84f7cb3b6af36e8a1d25c4219/9715)



同SourceTask的执行逻辑一样，每个operator在调用完函数完成处理后，会调用其output往下一个operator发送当前operator的处理结果，在最后一个operator处理后，会交给其output（对应的writer）完成结果输出或者发送给后续的task。

![SinkTask的执行](http://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE11a64b21abe1d2036052e44cd717e440/9712)





以上大致介绍了一个flink作业的API层的代码开发开发对应至物理执行Task（OperatorChain），以及Task的执行逻辑。如上所述，作业即分为两个Task（以上我们不准确的成为SourceTask，SinkTask），数据在两个Task级别的处理逻辑/流程如上分析，然后数据在SourceTask处理后，如何传递至SinkTask我们将在后续文章中进行分析。