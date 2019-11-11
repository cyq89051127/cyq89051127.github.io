---
layout: post
title:  "Flink Async-IO 源码分析"
date:   2019-11-11 16:45:12 +0800
tags:
      - Flink
---

### Async IO的设计

Flink 基于事件的消息驱动流处理引擎，对于每条消息都会触发一次全流程的处理，因此在与外部存储系统交互时，对于每条消息都需要一次外部请求，对于性能的损耗较大，严重制约了flink的吞吐量。 Flink 1.2中引入了Async IO(异步IO)来加快flink与外部系统的交互性能，提升吞吐量。[[FLIP-12: Asynchronous I/O Design and Implementation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65870673)]。 其设计的核心是对原有的每条处理后的消息发送至下游operator的执行流程进行改进。其核心实现是引入了一个AsyncWaitOperator,在其processElement/processWatermark方法中完成对消息的处理。其执行流程是：

1. 将每条消息封装成一个`StreamRecordQueueEntry`(其实现了`ResultFuture`)，放入`StreamElementQueue`中
2. 消息与外部系统交互的逻辑放入AsynInvoke方法中，将交互执行结果放入`StreamRecordQueueEntry`中
3. 启动一个emitter线程，从`StreamElementQueue`中读取已经完成的`StreamRecordQueueEntry`，将其结果发送至下游operator算子



### 顺序/无序的消息模式

在异步处理消息阶段，由于网络延迟，服务器响应等因素可能导致先发出的请求返回比后发出的请求更晚的情况，如果要严格做到消息发送至下游是有序的，则可能需要更多的存储空间，也会引发更高的消息处理时延，而不同的业务场景对于消息的顺序有不一样的要求（如在基于Eventtime的消息统计时watermark前的消息必须保证在watermark后发送至下游operator），因此在实现AsyncWaitOperator时，同时支持有序（Order）和无序（Unorder）的消息处理场景。



#### Ordered/有序 

 有序处理指的是消息流入operator的顺序与经过处理后流入下一级operator的顺序一致。

Flink基于`OrderedStreamElementQueue`实现了有序消息处理。 在顺序消息处理的场景中，先到的消息先发出。因此对于ProcessingTime和EventTime模式下的实现是一致的。其实现较为简单，使用简单的java的队列即可。如下：

* 消息的处理：

   > 1. 接收到的消息封装成`StreamElementQueueEntry`
   > 2. 通过ArrayQueue的addLast放入`ArrayDeque<StreamElementQueueEntry<?>>`

* 消息的发送

  
  
  > 1. 每条消息在处理完之后，其onCompleteHandler方法会调用检查位于队列头部的`StreamElementQueueEntry`是否已经完成，如果完成则会调用headIsComplete的signalAll方法
  >
  > ```java
  > private void onCompleteHandler(StreamElementQueueEntry<?> streamElementQueueEntry) throws InterruptedException {
  >    lock.lockInterruptibly();
  > 
  >    try {
  >       if (!queue.isEmpty() && queue.peek().isDone()) {
  >          LOG.debug("Signal ordered stream element queue has completed head element.");
  >          headIsCompleted.signalAll();
  >       }
  >    } finally {
  >       lock.unlock();
  >    }
  > }
  > ```
  >
  > 2. 通过emmiter线程循环从`ArrayDeque`中循环读取消息处理结果并发送至下游operator
  >
  > ```java
  > public AsyncResult peekBlockingly() throws InterruptedException {
  > lock.lockInterruptibly();
  > try {
  >    while (queue.isEmpty() || !queue.peek().isDone()) {
  >      
  >       headIsCompleted.await();
  >    }
  > 		...
  >    return queue.peek();
  > } finally {
  >    lock.unlock();
  > }
  > }
  > ```

####Unordered 

无序处理指的是消息流入operator的顺序与经过处理后流入下一级operator的顺序无必然关联。

* 在processingTime模式下：应用对消息的顺序不敏感，因此可以实现严格意义的无序处理。
* 在EventTime时间模式下：应用对消息顺序敏感，消息的顺序对应用的统计结果影响较大，应用定期生成watermark并在task/operator间流动，在两个watermark之间的消息其消息无序不会对应用结果产生负面影响，如果一个watermark前后的消息发送到下游时，与接收到消息的顺序不一致，那么很有可能导致统计结果异常。因此该模式下的无序处理主要是指watermark之间的消息处理是无序的，而同一watermark两侧的消息必须遵循watermark前的消息早于watermark发送至下游，而watermark后的消息晚于watermark发送至下游。

Flink基于`UnorderedStreamElementQueue`实现了无序消息处理，由于在该queue中实现了两种不同时间模式下的无序处理，其实现较Order模式更为复杂。查看源码发现其实现也比较精妙，主要数据结构如下：

> ```java
> /** Queue of uncompleted stream element queue entries segmented by watermarks. */
> private final ArrayDeque<Set<StreamElementQueueEntry<?>>> uncompletedQueue;
> 
> /** Queue of completed stream element queue entries. */
> private final ArrayDeque<StreamElementQueueEntry<?>> completedQueue;
> 
> /** First (chronologically oldest) uncompleted set of stream element queue entries. */
> private Set<StreamElementQueueEntry<?>> firstSet;
> 
> // Last (chronologically youngest) uncompleted set of stream element queue entries. New
> // stream element queue entries are inserted into this set.
> // 在类初始化方法中，将lastSet = firstSet
> private Set<StreamElementQueueEntry<?>> lastSet;
> ```



其核心逻辑如下：

1. 消息的处理

   > 1. 接收到的消息封装成`StreamElementQueueEntry`
   >
   > 2. 通过调用addEntry方法将`StreamElementQueueEntry`放入对应的queue中
   >
   >    ```java
   >    private <T> void addEntry(StreamElementQueueEntry<T> streamElementQueueEntry) {
   >          assert(lock.isHeldByCurrentThread());
   >      
   >          if (streamElementQueueEntry.isWatermark()) {
   >            //只有EventTime模式下接收到watermark类型的消息才会走入此分支
   >             lastSet = new HashSet<>(capacity);
   >    
   >             if (firstSet.isEmpty()) {
   >               // 只有在所有的queue中所有消息均发送至下游operator或者第一条消息就是watermark消息才会走进此分支
   >                firstSet.add(streamElementQueueEntry);
   >             } else {
   >               // 每次进入此分支，会生成一个只包含watermark消息的entry放入uncompleteQueue,同时生成一个lasteSet并放入uncomplteQueue,用于存放后续接收到的消息的entry
   >                Set<StreamElementQueueEntry<?>> watermarkSet = new HashSet<>(1);
   >                watermarkSet.add(streamElementQueueEntry);
   >                uncompletedQueue.offer(watermarkSet);
   >             }
   >             uncompletedQueue.offer(lastSet);
   >          } else {
   >             lastSet.add(streamElementQueueEntry);
   >          }
   >    			...
   >          numberEntries++;
   >       }
   >    }
   >    ```

2. 消息的发送

   > 1. 消息处理完毕后，其onCompleteHandler方法会试图将该消息的entry放入completedQueue，同时会遍历所有可以放入completeQueue的消息
   >
   >    ```java
   >    public void onCompleteHandler(StreamElementQueueEntry<?> streamElementQueueEntry) throws InterruptedException {
   >       lock.lockInterruptibly();
   >       try {
   >         // 从firstSet中移除该entry,如果该entry不在firsetSeq中，则跳过
   >          if (firstSet.remove(streamElementQueueEntry)) {
   >            //将该entry放入completeQueue中
   >             completedQueue.offer(streamElementQueueEntry);
   >            //rstSet为空，且firset != lastSet 说明此时后续至少还有一些set中可能包含已经处理完的消息待放入completeQueue中
   >             while (firstSet.isEmpty() && firstSet != lastSet) {
   >                firstSet = uncompletedQueue.poll();
   >                Iterator<StreamElementQueueEntry<?>> it = firstSet.iterator();
   >                while (it.hasNext()) {
   >                   StreamElementQueueEntry<?> bufferEntry = it.next();
   >                   if (bufferEntry.isDone()) {
   >                      completedQueue.offer(bufferEntry);
   >                      it.remove();
   >                   }
   >                }
   >             }
   >             LOG.debug("Signal unordered stream element queue has completed entries.");
   >             hasCompletedEntries.signalAll();
   >          }
   >       } finally {
   >          lock.unlock();
   >       }
   >    }
   >    ```
   >
   > 2. 通过emmiter线程循环从`completeQueue`中循环读取消息处理结果并发送至下游operator
   >
   >    ```java
   >    //每次顺序从completedQueue取出消息发送至下游
   >    public AsyncResult poll() throws InterruptedException {
   >       lock.lockInterruptibly();
   >       try {
   >          while (completedQueue.isEmpty()) {
   >             hasCompletedEntries.await();
   >          }
   >          numberEntries--;
   >          notFull.signalAll();
   >    			...
   >          return completedQueue.poll();
   >       } finally {
   >          lock.unlock();
   >       }
   >    }
   >    ```

##### 基于ProcessingTime的Unorder模式

该模式下，不存在watermark类型的消息，因此所有消息的entry都是放入lastSeq(此场景下lastSet和firstSet是同一个),且此时incompleteQueue并没有被使用到；在消息entry的onCompleteHandler方法中，直接将该消息的entry放入completeQueue中，通过emmitter线程发送至下游operator，因此该场景下实现的是完全无序的处理模式。

#####基于EventTime的Unorder模式

该模式下，实现较为复杂。如图：

![Unordered EventTime mode](https://note.youdao.com/yws/public/resource/309860f8d6d1ca28097175b7c5701261/xmlnote/WEBRESOURCE959dd8f774ff827139e1a7e6d22f7fac/9677)

* 消息的接收
  > 消息的接收与存放步骤如下，保证了watermark前后的消息分别放入不同的set中
  > 1. 流入的消息分别为E1,E2,E3,W1,E4,E5,W2,E6,W3,E7
  > 2. 当operator接收到E1,E2,E3时，分别生成相关的streamElementQueueEntry，FE1,FE2,FE3存入lastSet，并放入uncompleteQueue中
  > 3. 接收到watermark消息W1后，生成只包含一个entry的queue并放入uncompleteQueue中，同时生成lastSet，放入uncompleteQueue中
  > 4. 后续接收到消息如E4,E5方别生成FE4,FE5放入lastSet中
  > 5. 再次接收到watermark消息，则重复3,4

* 消息的发送

  > 1. firstSet指向的是uncompleteQueue中的head，当有消息处理完执行`onCompleteHandler`方法时，会在firstSet中移除此entry，并将其放入completeQueue中（此步骤说明在同一个set中，消息的接收和发送是无序的）
  > 2. 如果此时firstSet为空，且firstSet != lastSet 则说明此时uncompleteQueue中还存在其他Set，将firstSet设置为uncompleteQueue的下一个元素Set，根据消息接收和放入uncompleteQueue的逻辑，此时的Set应为只包含一个watermark的entry的Set，由于watermark的entry不需要和外部系统交互，直接执行完毕返回的，则此时可以将watermark的消息直接放入completeQueue中同时遍历下一个Set，取出器已经执行完毕的entry，并放入completeQueue中（此逻辑保证了watermark前后两侧的消息在发送至下游operator时，依旧分布在watermark前后两侧）


