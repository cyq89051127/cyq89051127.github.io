---
layout: post
title:  "Flink checkpoint流程源码分析"
date:   2019-11-05 16:20:12 +0800
tags:
      - Flink
---


Checkpoint在分布式流处理框架的准确性具有重要意义。Flink实现了基于`Chandy-Lamport`算法的checkpoint机制。在消息可靠性保障，集群升级等场景具有重大意义。Flink中checkpoint的调度管理由Jobmanager测下发checkpoint指令，并由各TaskManager执行，在Flink的框架中，对消息的读取，处理，输出等逻辑都是由Task执行，因此执行checkpoint的核心都是在TaskManager中完成，在各task完成checkpoint之后上报JobManager，并有JobManager进行元信息的写入。本文结合源码对Flink中的checkpoint流程进行简要分析。

#### JobManager中的checkpoint触发

在Flink的Jobmnanager中，在接收到客户端提交的JobGraph后，会生成相关的ExecutionGraph，如果配置了checkpoint操作，则会生成一个CheckpointCoordinatorDeActivator的监听器，该监听器，会在job状态装华为running时，调用`startCheckpointScheduler`方法，通过定时调用`ScheduledTrigger`来定期想各`SourceTask`发送checkpoint请求。

`ScheduledTrigger`是一个runnable实现,run方法调用`triggerCheckpoint(System.currentTimeMillis(), true);`  其逻辑较为复杂，主要进行如下操作：

* 根据配置进行各种checkpoint需要的校验
* 找出需要发送checkpoint消息的task(即tasksToTrigger，由生成JobGraph时生成，由所有不包含输入的顶点组成)放入executions
* 找出需要返回checkpoint的ack反馈信息的task放入ackTasks，并将其作为构造PendingCheckpoint的参数
* 注册checkpoint取消器，如果超过checkpointTImeout的时间内，还没有结束，则取消checkpoint
* 调用`execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions)`方法向各个sourcetask发送checkpoint，主要包括`attemptId, getVertex().getJobId(), checkpointId, timestamp, checkpointOptions, advanceToEndOfEventTime`等信息



> ```java
> public PendingCheckpoint triggerCheckpoint(
>       long timestamp,
>       CheckpointProperties props,
>       @Nullable String externalSavepointLocation,
>       boolean isPeriodic,
>       boolean advanceToEndOfTime) throws CheckpointException {
> ...
>    // make some eager pre-checks
>    synchronized (lock) {
>       ...
>
>    // check if all tasks that we need to trigger are running.
>    // if not, abort the checkpoint
>    Execution[] executions = new Execution[tasksToTrigger.length];
>    for (int i = 0; i < tasksToTrigger.length; i++) {
>       Execution ee = tasksToTrigger[i].getCurrentExecutionAttempt();
>       if (ee == null) {
>       ...
>       } else if (ee.getState() == ExecutionState.RUNNING) {
>          executions[i] = ee;
>       } else {
>         ...
>    }
>    }
>
>    // next, check if all tasks that need to acknowledge the checkpoint are running.
>    // if not, abort the checkpoint
>    Map<ExecutionAttemptID, ExecutionVertex> ackTasks = new HashMap<>(tasksToWaitFor.length);
>
>    for (ExecutionVertex ev : tasksToWaitFor) {
>       Execution ee = ev.getCurrentExecutionAttempt();
>       if (ee != null) {
>       ackTasks.put(ee.getAttemptId(), ev);
>       } else {
>       ...
>       }
>    }
>
> synchronized (triggerLock) {
>
>       final CheckpointStorageLocation checkpointStorageLocation;
>       final long checkpointID;
>
>       try {
>          // this must happen outside the coordinator-wide lock, because it communicates
>          // with external services (in HA mode) and may block for a while.
>          checkpointID = checkpointIdCounter.getAndIncrement();
>
>          checkpointStorageLocation = props.isSavepoint() ?
>                checkpointStorage.initializeLocationForSavepoint(checkpointID, externalSavepointLocation) :
>                checkpointStorage.initializeLocationForCheckpoint(checkpointID);
>       }
>       catch (Throwable t) {
>          ...
>       }
>
>       final PendingCheckpoint checkpoint = new PendingCheckpoint(
>          job,
>          checkpointID,
>          timestamp,
>       ackTasks,
>          props,
>          checkpointStorageLocation,
>          executor);
>             ...
>
>          // send the messages to the tasks that trigger their checkpoint
>          for (Execution execution: executions) {
>             if (props.isSynchronous()) {
>                execution.triggerSynchronousSavepoint(checkpointID, timestamp, checkpointOptions, advanceToEndOfTime);
>             } else {
>                execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions);
>             }
>          }
>    ...
>
> } // end trigger lock
>    }
> ```

#### SourceTask的checkpoint执行流程

在JobMaster发送Checkpoint之后，对于SourceTask来讲，其运行在TaskManager进程中，其对checkpoint消息的接收通过TaskManager的akka服务完成，由`flink-akka.actor.default-dispatcher-*`线程处理，该线程池由TaskManger启动时创建，该处理会调用至Task的`triggerCheckpointBarrier`来完成checkpoint

在triggerCheckpointBarrier内部可以看出，checkpoint逻辑被封装为runnable，通过executeAsyncCallRunnable(Runnable runnable, String callName) 方法完成调用，在方法中创建asyncCallDispatcher，并使用该ExecutorService执行以上runnable方法

ExecutorService 创建如下，由此可知，运行checkpoint逻辑的线程为`Async  calls on Source*`：

> ```java
> executor = Executors.newSingleThreadExecutor(
>       new DispatcherThreadFactory(
>          TASK_THREADS_GROUP,
>          "Async calls on " + taskNameWithSubtask,
>          userCodeClassLoader));
> ```



在该runnable方法中，通过invokable.triggerCheckpoint来实现checkpoint逻辑，其实就是StreamTask.triggerCheckpoint -> StreamTask.performCheckpoint，具体调用堆栈如下：

> ```java
> "Async calls on Source: Custom Source -> Map -> Timestamps/Watermarks (1/1)@10192" daemon prio=5 tid=0x13f nid=NA runnable
>  java.lang.Thread.State: RUNNABLE
>     at org.apache.flink.streaming.runtime.tasks.StreamTask.performCheckpoint(StreamTask.java:784)
>     locked <0x27f3> (a java.lang.Object)
>     at org.apache.flink.streaming.runtime.tasks.StreamTask.triggerCheckpoint(StreamTask.java:683)
>     at org.apache.flink.streaming.runtime.tasks.SourceStreamTask.triggerCheckpoint(SourceStreamTask.java:178)
>     at org.apache.flink.runtime.taskmanager.Task$$RunnableAdapter.call(Executors.java:511)
>     at  java.util.concurrent.FutureTask.run(FutureTask.java:266)
>     at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
>     at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
>     at java.lang.Thread.run(Thread.java:745)
>``` 

其中performCheckpoint的核心流程如下：

1. 调用prepareSnapshotPreBarrier方法
2. 将checkpoint消息发送至下游task
3. 执行checkpoint

> ```java
> synchronized (lock) {
>    if (isRunning) {
>
>       if (checkpointOptions.getCheckpointType().isSynchronous()) {
>          syncSavepointLatch.setCheckpointId(checkpointId);
>
>          if (advanceToEndOfTime) {
>             advanceToEndOfEventTime();
>          }
>       }
>
>       // All of the following steps happen as an atomic step from the perspective of barriers and
>       // records/watermarks/timers/callbacks.
>       // We generally try to emit the checkpoint barrier as soon as possible to not affect downstream
>       // checkpoint alignments
>
>       // Step (1): Prepare the checkpoint, allow operators to do some pre-barrier work.
>       //           The pre-barrier work should be nothing or minimal in the common case.
>       operatorChain.prepareSnapshotPreBarrier(checkpointId);
>
>       // Step (2): Send the checkpoint barrier downstream
>       operatorChain.broadcastCheckpointBarrier(
>             checkpointId,
>             checkpointMetaData.getTimestamp(),
>             checkpointOptions);
>
>       // Step (3): Take the state snapshot. This should be largely asynchronous, to not
>       //           impact progress of the streaming topology
>       checkpointState(checkpointMetaData, checkpointOptions, checkpointMetrics);
>
>       return true;
>    }
> ```

如上Step(3)中的checkpointingOperation.executeCheckpointing()的主要是调用task的各个operator进行checkpoint，并调用异步接口将各operator的checkpint执行或者获取其checkpoint结果。

> ```java
> public void executeCheckpointing() throws Exception {
>    startSyncPartNano = System.nanoTime();
>
>    try {
>       for (StreamOperator<?> op : allOperators) {
>         // 对各operator进行checkpoint调用，该过程我们成为是同步调用
>          checkpointStreamOperator(op);
>       }
>
>       if (LOG.isDebugEnabled()) {
>          LOG.debug("Finished synchronous checkpoints for checkpoint {} on task {}",
>             checkpointMetaData.getCheckpointId(), owner.getName());
>       }
>
>       startAsyncPartNano = System.nanoTime();
>
>       checkpointMetrics.setSyncDurationMillis((startAsyncPartNano - startSyncPartNano) / 1_000_000);
>
>       // we are transferring ownership over snapshotInProgressList for cleanup to the thread, active on submit
>       AsyncCheckpointRunnable asyncCheckpointRunnable = new AsyncCheckpointRunnable(
>          owner,
>          operatorSnapshotsInProgress,
>          checkpointMetaData,
>          checkpointMetrics,
>          startAsyncPartNano);
>
>       owner.cancelables.registerCloseable(asyncCheckpointRunnable);
>       owner.asyncOperationsThreadPool.execute(asyncCheckpointRunnable);
>      // 在异步调用各future执行获取结果，直接打印完成同步checkpint
>      if (LOG.isDebugEnabled()) {
> 					LOG.debug("{} - finished synchronous part of checkpoint {}. " +
> 							"Alignment duration: {} ms, snapshot duration {} ms",
> 						owner.getName(), checkpointMetaData.getCheckpointId(),
> 						checkpointMetrics.getAlignmentDurationNanos() / 1_000_000,
> 						checkpointMetrics.getSyncDurationMillis());
> 				}
> ```

其实现又分为两个部分，

* 同步checkpoint

  对每个operator进行checkpoint，通过调用各个operator的snapshotState方法生成OperatorSnapshotFutures，并将其放入operatorSnapshotsInProgress。各operator的snapshotState方法的实现是一样的，都是调用AbstractStreamOperator的snapshotState方法

  > ```java
  > public final OperatorSnapshotFutures snapshotState(long checkpointId, long timestamp, CheckpointOptions checkpointOptions,
  >       CheckpointStreamFactory factory) throws Exception {
  >
  >    KeyGroupRange keyGroupRange = null != keyedStateBackend ?
  >          keyedStateBackend.getKeyGroupRange() : KeyGroupRange.EMPTY_KEY_GROUP_RANGE;
  >
  >    OperatorSnapshotFutures snapshotInProgress = new OperatorSnapshotFutures();
  >
  >    StateSnapshotContextSynchronousImpl snapshotContext = new StateSnapshotContextSynchronousImpl(
  >       checkpointId,
  >       timestamp,
  >       factory,
  >       keyGroupRange,
  >       getContainingTask().getCancelables());
  >
  >    try {
  >      //调用到各算子/函数的snapshot方法，
  >       snapshotState(snapshotContext);
  > snapshotInProgress.setKeyedStateRawFuture(snapshotContext.getKeyedStateStreamFuture());
  >       snapshotInProgress.setOperatorStateRawFuture(snapshotContext.getOperatorStateStreamFuture());
  >       if (null != operatorStateBackend) {
  >          snapshotInProgress.setOperatorStateManagedFuture(
  >             operatorStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
  >       }
  >       if (null != keyedStateBackend) {
  >          snapshotInProgress.setKeyedStateManagedFuture(
  >             keyedStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
  >       }
  >    } catch (Exception snapshotException) {
  >       ..
  >    }
  >    return snapshotInProgress;
  >    }
  > ```

AbstractStreamOperator的snapshotState方法返回的是一个`OperatorSnapshotFutures`，其中包含了各个操作的future，如operatorState,keyedState等也是在该函数中进行持久化，查看`operatorStateBackend.snapshot，keyedStateBackend.snapshot`方法即可看出真正的数据写入; snapshotState方法也会调用到各算子/函数的snapshot方法，以FlinkKafkaConsumer为例，其snapshotState方法就是在此处被调用，堆栈如下：

>```java
>"Async calls on Source: Custom Source -> Map -> Timestamps/Watermarks (1/1)@8852" daemon prio=5 tid=0x62 nid=NA runnable
>   java.lang.Thread.State: RUNNABLE
>    at org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumerBase.snapshotState(FlinkKafkaConsumerBase.java:897)
>    at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.trySnapshotFunctionState(StreamingFunctionUtils.java:118)
>    at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.snapshotFunctionState(StreamingFunctionUtils.java:99)
>    at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.snapshotState(AbstractUdfStreamOperator.java:90)
>    at org.apache.flink.streaming.api.operators.AbstractStreamOperator.snapshotState(AbstractStreamOperator.java:399)
>    at org.apache.flink.streaming.runtime.tasks.StreamTask`$`CheckpointingOperation.checkpointStreamOperator(StreamTask.java:1264)
>    ​at org.apache.flink.streaming.runtime.tasks.StreamTask`$`CheckpointingOperation.executeCheckpointing(StreamTask.java:1198)
>    ​at org.apache.flink.streaming.runtime.tasks.StreamTask.checkpointState(StreamTask.java:869)
>    ​at org.apache.flink.streaming.runtime.tasks.StreamTask.performCheckpoint(StreamTask.java:774)
>    locked <0x233d> (a java.lang.Object)
>    at org.apache.flink.streaming.runtime.tasks.StreamTask.triggerCheckpoint(StreamTask.java:683)
>    at org.apache.flink.streaming.runtime.tasks.SourceStreamTask.triggerCheckpoint(SourceStreamTask.java:178)
>    at org.apache.flink.runtime.taskmanager.Task$1.run(Task.java:1155)
>    at java.util.concurrent.Executors$$RunnableAdapter.call(Executors.java:511)
>    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
>    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
>    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
>    at java.lang.Thread.run(Thread.java:745)
>```

* 异步checkpoint

  同步checkpoint完成之后，会将具体的信息写入异步实现，由名为AsyncOperations-thread-* 的线程来执行

  > ```java
  > // we are transferring ownership over snapshotInProgressList for cleanup to the thread, active on submit
  > AsyncCheckpointRunnable asyncCheckpointRunnable = new AsyncCheckpointRunnable(
  >    owner,
  >    operatorSnapshotsInProgress, //
  >    checkpointMetaData,
  >    checkpointMetrics,
  >    startAsyncPartNano);
  >
  > owner.cancelables.registerCloseable(asyncCheckpointRunnable);
  > owner.asyncOperationsThreadPool.execute(asyncCheckpointRunnable);
  > ```

  查看AsyncCheckpointRunnable的run方法可以看出，其

  * 调用`new OperatorSnapshotFinalizer(snapshotInProgress)`方法等待各future执行完成。
  * 调用`reportCompletedSnapshotStates`方法完成checkpoint状态上报JobManager

  > ```java
  > public OperatorSnapshotFinalizer(
  >    @Nonnull OperatorSnapshotFutures snapshotFutures) throws ExecutionException, InterruptedException {
  >
  >    SnapshotResult<KeyedStateHandle> keyedManaged =
  >       FutureUtils.runIfNotDoneAndGet(snapshotFutures.getKeyedStateManagedFuture());
  >
  >    SnapshotResult<KeyedStateHandle> keyedRaw =
  >       FutureUtils.runIfNotDoneAndGet(snapshotFutures.getKeyedStateRawFuture());
  >
  >    SnapshotResult<OperatorStateHandle> operatorManaged =
  >       FutureUtils.runIfNotDoneAndGet(snapshotFutures.getOperatorStateManagedFuture());
  >
  >    SnapshotResult<OperatorStateHandle> operatorRaw =
  >       FutureUtils.runIfNotDoneAndGet(snapshotFutures.getOperatorStateRawFuture());
  >
  >    jobManagerOwnedState = new OperatorSubtaskState(
  >       operatorManaged.getJobManagerOwnedSnapshot(),
  >       operatorRaw.getJobManagerOwnedSnapshot(),
  >       keyedManaged.getJobManagerOwnedSnapshot(),
  >       keyedRaw.getJobManagerOwnedSnapshot()
  >    );
  >
  >    taskLocalState = new OperatorSubtaskState(
  >       operatorManaged.getTaskLocalSnapshot(),
  >       operatorRaw.getTaskLocalSnapshot(),
  >       keyedManaged.getTaskLocalSnapshot(),
  >       keyedRaw.getTaskLocalSnapshot()
  >    );
  > }
  > ```

#### 非SourceTask的checkpoint执行流程

对于SourceTask，其checkpint的执行由jobmaster发送的checkpoint消息触发，而对于非SourceTask，则是依靠上游task的checkpointBarrier消息触发。由以上分析可知，在SourceTask进行checkpoint时，会向下游发送CheckpointBarrier消息，而下游的task正是拿到该消息后，进行checkpoint操作。

对于非SourceTask，其一直通过循环从上游读取消息，当接收一条消息后，会对消息类型进行判断，如果是CheckpointBarrier类型的消息则会进一步判断是需要对齐或是进行checkpoint。该逻辑在 `CheckpointInputGate#pollNext()`方法中进行：

> ```java
> public Optional<BufferOrEvent> pollNext() throws Exception {
>    while (true) {
>       ...
>       BufferOrEvent bufferOrEvent = next.get();
>      // 如果消息对应的channel已经被block，则将该消息缓存至bufferStorage
>       if (barrierHandler.isBlocked(offsetChannelIndex(bufferOrEvent.getChannelIndex()))) {
>          // if the channel is blocked, we just store the BufferOrEvent
>          bufferStorage.add(bufferOrEvent);
>          if (bufferStorage.isFull()) {
>             barrierHandler.checkpointSizeLimitExceeded(bufferStorage.getMaxBufferedBytes());
>             bufferStorage.rollOver();
>          }
>       }
>      // 如果该消息是一般消息，则返回继续下一步处理
>       else if (bufferOrEvent.isBuffer()) {
>          return next;
>       }
>      //如果该消息是CheckpointBarrier类型，则处理该消息
>       else if (bufferOrEvent.getEvent().getClass() == CheckpointBarrier.class) {
>          CheckpointBarrier checkpointBarrier = (CheckpointBarrier) bufferOrEvent.getEvent();
>          if (!endOfInputGate) {
>             // process barriers only if there is a chance of the checkpoint completing
>             if (barrierHandler.processBarrier(checkpointBarrier, offsetChannelIndex(bufferOrEvent.getChannelIndex()), bufferStorage.getPendingBytes())) {
>                bufferStorage.rollOver();
>             }
>          }
>       }
>      //如果该消息是取消CheckpointBarrier类型，则处理该消息
>       else if (bufferOrEvent.getEvent().getClass() == CancelCheckpointMarker.class) {
>          if (barrierHandler.processCancellationBarrier((CancelCheckpointMarker) bufferOrEvent.getEvent())) {
>             bufferStorage.rollOver();
>          }
>       }
>       else {
>          if (bufferOrEvent.getEvent().getClass() == EndOfPartitionEvent.class) {
>             if (barrierHandler.processEndOfPartition()) {
>                bufferStorage.rollOver();
>             }
>          }
>          return next;
>       }
>    }
> }
> ```

* Barrier对齐

  ​	Barier对齐本质是表现在对上游task发送的checkpoint消息的等待和对齐。是Task级别的barrier对齐。当然如果Task几倍的对齐的，则operator级别自然是对齐的。

  从`CheckpointBarrierAligner#processBarrier`中可以看出barrier对齐的逻辑

  1. 如果是只有一个输入channel，如果 barrieId > currentCheckpointId 则直接调用notifyCheckpoint方法触发checkpoint，并返回false

  2. 如果有多个inputChannel

     1. 如果是首次接收到checkpointBarrier，
        1. 如果barrierId > currentCheckpointId，则开启一个新的对齐操作，将currentCheckpointId设置为barrierId并将该消息的channel block，numBarriersReceived++
          2. 其他返回false

     2. 如果不是首次接收到barrier，则判断
         1. 如果barrierId == currentCheckpointId，则将该消息的channel block，并numBarriersReceived++
         2. 如果barrierId > currentCheckpointId，则将之前的checkpint对齐，进行新一次checkpoint
         3. 其他则返回false

  3.   当 numBarriersReceived + numClosedChannels == totalNumberOfInputChannels时，说明已经完成对齐，则直接将blockedChannels均设置为非block状态，numBarriersReceived设置为0，并调用notifyCheckpoint触发checkpoint

对于非SourceTask的checkpoint的执行与SourceTask的执行过程是一致的，都是通过StreamTask的performCheckpoint方法完成，可参考以上分析。以FlinkKafkaProducer为例，其operator的snapshotState方法会调用到FlinkKafkaProducer的precommit，堆栈如下：

> ```java
>  "Window(TumblingProcessingTimeWindows(120000), ProcessingTimeTrigger, ScalaReduceFunction, PassThroughWindowFunction) -> Map -> Sink: Unnamed (1/1)@9180" prio=5 tid=0x7c nid=NA runnable
>   java.lang.Thread.State: RUNNABLE
> 	  at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer.preCommit(FlinkKafkaProducer.java:892)
> 	  at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer.preCommit(FlinkKafkaProducer.java:98)
> 	  at org.apache.flink.streaming.api.functions.sink.TwoPhaseCommitSinkFunction.snapshotState(TwoPhaseCommitSinkFunction.java:311)
> 	  at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer.snapshotState(FlinkKafkaProducer.java:973)
> 	  at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.trySnapshotFunctionState(StreamingFunctionUtils.java:118)
> 	  at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.snapshotFunctionState(StreamingFunctionUtils.java:99)
> 	  at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.snapshotState(AbstractUdfStreamOperator.java:90)
> 	  at org.apache.flink.streaming.api.operators.AbstractStreamOperator.snapshotState(AbstractStreamOperator.java:399)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask$$CheckpointingOperation.checkpointStreamOperator(StreamTask.java:1264)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask$CheckpointingOperation.executeCheckpointing(StreamTask.java:1198)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask.checkpointState(StreamTask.java:869)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask.performCheckpoint(StreamTask.java:774)
> 	  - locked <0x24d7> (a java.lang.Object)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask.triggerCheckpointOnBarrier(StreamTask.java:705)
> 	  at org.apache.flink.streaming.runtime.io.CheckpointBarrierHandler.notifyCheckpoint(CheckpointBarrierHandler.java:88)
> 	  at org.apache.flink.streaming.runtime.io.CheckpointBarrierAligner.processBarrier(CheckpointBarrierAligner.java:113)
> 	  at org.apache.flink.streaming.runtime.io.CheckpointedInputGate.pollNext(CheckpointedInputGate.java:155)
> 	  at org.apache.flink.streaming.runtime.io.StreamTaskNetworkInput.pollNextNullable(StreamTaskNetworkInput.java:102)
> 	  at org.apache.flink.streaming.runtime.io.StreamTaskNetworkInput.pollNextNullable(StreamTaskNetworkInput.java:47)
> 	  at org.apache.flink.streaming.runtime.io.StreamOneInputProcessor.processInput(StreamOneInputProcessor.java:135)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask.performDefaultAction(StreamTask.java:276)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask.run(StreamTask.java:298)
> 	  at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:403)
> 	  at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:705)
> 	  at org.apache.flink.runtime.taskmanager.Task.run(Task.java:530)
> 	  at java.lang.Thread.run(Thread.java:745)
> ```

#### JobManager收到task的checkpoint反馈的处理

在Task完成checkpoint之后，会向JobManager发送`AcknowledgeCheckpoint`消息，该消息在JobManager侧依然通过CheckpointCoordinator处理，主要逻辑如下：

* 对返回消息，作业状态进行校验

*  调用checkpoint.acknowledgeTask方法对一些状态进行清理，并根据不同情况返回不同的状态，并根据不同的返回状态进行响应处理。如果所有需要反馈ack信息的task都已经反馈，则执行`completePendingCheckpoint(checkpoint

* 在`completePendingCheckpoint`中对一些元信息进行保存处理并向需要执行commit请求的task发送`notifyCheckpointComplete`

  > ```java
  > for (ExecutionVertex ev : tasksToCommitTo) {
  >    Execution ee = ev.getCurrentExecutionAttempt();
  >    if (ee != null) {
  >       ee.notifyCheckpointComplete(checkpointId, timestamp);
  >    }
  > }
  > ```


