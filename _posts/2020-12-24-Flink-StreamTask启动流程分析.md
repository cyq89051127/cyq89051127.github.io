---
layout: post
title:  "Flink-StreamTask启动流程分析"
date:   2020-12-24 10:17:12 +0800
tags:
      - Flink
---

StreamTask是流作业的任务基类，通常一个流作业的task启动由该方法的invoke函数为入口，本文基于Flink1.11.0该类生命流程进行分析。



#### StreamTask的构造

StreamTask的的初始化构造方法主要对一些参数进行设置，如configuration,stateBackend,timeService等

```java
protected StreamTask(
			Environment environment,
			@Nullable TimerService timerService,
			Thread.UncaughtExceptionHandler uncaughtExceptionHandler,
			StreamTaskActionExecutor actionExecutor,
			TaskMailbox mailbox) throws Exception {

		super(environment);

		this.configuration = new StreamConfig(getTaskConfiguration());
		this.recordWriter = createRecordWriterDelegate(configuration, environment);
		this.actionExecutor = Preconditions.checkNotNull(actionExecutor);
    // 创建处理器，用于异步执行各种请求，同时将processInput方法的执行放入待执行的任务队列
		this.mailboxProcessor = new MailboxProcessor(this::processInput, mailbox, actionExecutor);
		this.mailboxProcessor.initMetric(environment.getMetricGroup());
		this.mainMailboxExecutor = mailboxProcessor.getMainMailboxExecutor();
		this.asyncExceptionHandler = new StreamTaskAsyncExceptionHandler(environment);
		this.asyncOperationsThreadPool = Executors.newCachedThreadPool(
			new ExecutorThreadFactory("AsyncOperations", uncaughtExceptionHandler));
		// 创建stateBackend
		this.stateBackend = createStateBackend();

		this.subtaskCheckpointCoordinator = new SubtaskCheckpointCoordinatorImpl(
			stateBackend.createCheckpointStorage(getEnvironment().getJobID()),
			getName(),
			actionExecutor,
			getCancelables(),
			getAsyncOperationsThreadPool(),
			getEnvironment(),
			this,
			configuration.isUnalignedCheckpointsEnabled(),
			this::prepareInputSnapshot);

		// if the clock is not already set, then assign a default TimeServiceProvider
		if (timerService == null) {
			ThreadFactory timerThreadFactory = new DispatcherThreadFactory(TRIGGER_THREAD_GROUP, "Time Trigger for " + getName());
			this.timerService = new SystemProcessingTimeService(this::handleTimerException, timerThreadFactory);
		} else {
			this.timerService = timerService;
		}

		this.channelIOExecutor = Executors.newSingleThreadExecutor(new ExecutorThreadFactory("channel-state-unspilling"));
	}
```





该方法主要有如下流程：

```
*  -- invoke()
*        |
*        +----> Create basic utils (config, etc) and load the chain of operators
*        +----> operators.setup()  
*        +----> task specific init()
*        +----> initialize-operator-states()
*        +----> open-operators()
*        +----> run()
*        +----> close-operators()
*        +----> dispose-operators()
*        +----> common cleanup
*        +----> task specific cleanup()
```

总结起来task的运行主要分为三个主要部分：

1. StreamTask初始化 ---- beforeInvoke
2. 运行业务逻辑  ------ runMailboxLoop
3. 关闭/资源清理   ----- afterInvoke

### StreamTask初始化

​    在beforeInvoke方法中，主要调用如下步骤：

#### 生成operatorChain

​		Flink的task运行本质是执行业务逻辑（业务处理代码/处理函数），Flink将业务处理函数进行抽象为operator，通过operatorChain将业务代码串起来执行，完成业务逻辑的处理。后续笔者将针对operatorchain的生成单独分析。

#### 调用具体task的init方法

   init方法在StreamTask中是抽象方法，由具体的task进行覆写实现，通常该方法中会生成inputStreamPorcessor，完成数据的处理。 如OneInputStreamTask中的init如下：

```java
public void init() throws Exception {
		StreamConfig configuration = getConfiguration();
		int numberOfInputs = configuration.getNumberOfInputs();

		if (numberOfInputs > 0) {
			CheckpointedInputGate inputGate = createCheckpointedInputGate();
			DataOutput<IN> output = createDataOutput();
			StreamTaskInput<IN> input = createTaskInput(inputGate, output);
			inputProcessor = new StreamOneInputProcessor<>(
				input,
				output,
				operatorChain);
		}
		headOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, this.inputWatermarkGauge);
		// wrap watermark gauge since registered metrics must be unique
		getEnvironment().getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, this.inputWatermarkGauge::getValue);
	}
```



#### operator的初始化和open

依次调用operatorChain中所有operator的初始化和open方法：

```java
protected void initializeStateAndOpenOperators(StreamTaskStateInitializer streamTaskStateInitializer) throws Exception {
		for (StreamOperatorWrapper<?, ?> operatorWrapper : getAllOperators(true)) {
			StreamOperator<?> operator = operatorWrapper.getStreamOperator();
			operator.initializeState(streamTaskStateInitializer);
			operator.open();
		}
	}
```



### 运行业务逻辑

任务的处理逻辑主要有processInput方法来处理，其核心是调用的inputPorcessor的processInput方法来完成。

```java
protected void processInput(MailboxDefaultAction.Controller controller) throws Exception {
		InputStatus status = inputProcessor.processInput();
		if (status == InputStatus.MORE_AVAILABLE && recordWriter.isAvailable()) {
			return;
		}
		if (status == InputStatus.END_OF_INPUT) {
			controller.allActionsCompleted();
			return;
		}
		CompletableFuture<?> jointFuture = getInputOutputJointFuture(status);
		MailboxDefaultAction.Suspension suspendedDefaultAction = controller.suspendDefaultAction();
		jointFuture.thenRun(suspendedDefaultAction::resume);
	}
```



### 关闭/资源清理

```
// close the head operators
operatorChain.closeOperators
// make sure no new timers can come
FutureUtils.forward(timerService.quiesce(), timersFinishedFuture);
// let mailbox execution reject all new letters from this point
mailboxProcessor.prepareClose
// processes the remaining mails; no new mails can be enqueued
mailboxProcessor.drain
// make sure all timers finish
timersFinishedFuture.get();
// make sure all buffered data is flushed
operatorChain.flushOutputs
// dispose the operators
disposeAllOperators  -> foreach (operator.dispose  --> stateHandler.dispose)
```