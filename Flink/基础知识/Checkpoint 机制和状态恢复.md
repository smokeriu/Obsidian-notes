> 代码参考的Flink version: 1.16

# Checkpoint的发起
在`StreamingJobGraphGenerator`中，生成JobGraph后会调用`configureCheckpointing` 方法进行 Checkpoint相关的配置。具体是`configureCheckpointing方法`。
这里会将：状态后端，Checkpoint配置一并配置。生成一个JobCheckpointingSettings。

在 `ExecutionGraphBuilder#buildGraph` 中，如果作业开启了 checkpoint，则会调用 `ExecutionGraph.enableCheckpointing()` 方法。
```java
checkpointCoordinator =  
	new CheckpointCoordinator(  
	jobInformation.getJobId(),  
	chkConfig,  
	operatorCoordinators,  
	checkpointIDCounter,  
	checkpointStore,  
	checkpointStorage,  
	ioExecutor,  
	checkpointsCleaner,  
	new ScheduledExecutorServiceAdapter(checkpointCoordinatorTimer),  
	failureManager,  
	createCheckpointPlanCalculator(  
	chkConfig.isEnableCheckpointsAfterTasksFinish()),  
	new ExecutionAttemptMappingProvider(getAllExecutionVertices()),  
	checkpointStatsTracker);
```
这里会创建 `CheckpointCoordinator` 对象，并注册一个作业状态的监听 `CheckpointCoordinatorDeActivator`
```java
if (checkpointCoordinator.isPeriodicCheckpointingConfigured()) {  
	// the periodic checkpoint scheduler is activated and deactivated as a result of  
	// job status changes (running -> on, all other states -> off)  
	registerJobStatusListener(checkpointCoordinator.createActivatorDeactivator());  
}
```
当状态变更为Running时，才会启动coordinator。
```java
public void jobStatusChanges(JobID jobId, JobStatus newJobStatus, long timestamp) {  
	if (newJobStatus == JobStatus.RUNNING) {  
		// start the checkpoint scheduler  
		coordinator.startCheckpointScheduler();  
	} else {  
		// anything else should stop the trigger for now  
		coordinator.stopCheckpointScheduler();  
	}  
}
```
其中，`startCheckpointScheduler`会调用timer的`scheduleAtFixedRate`来定时触发checkpoint。

> timer由ScheduledExecutorServiceAdapter触发。

# checkpoint的触发

具体的触发逻辑在`CheckpointCoordinator`的`triggerCheckpoint`中。
1. 构建CheckpointTriggerRequest。
	1. 如果有队列已满（1000），且包含用户手动触发的checkpoint，则跳过这次checkpoint。
	2. 如果距离上一次成功时间较短，也不会发送checkpointRequest。
2. 触发startTriggeringCheckpoint方法。

## 具体的触发
1. 生成一个checkpoint计划：`checkpointPlanCalculator.calculateCheckpointPlan()`。
	1. 检查是否所有task都在RUNNING。
	2. 默认会对source打上trigger，用于实际触发checkpoint消息。为operator和sink打上waitFor和commit标签，用于报告checkpoint的完成。
	3. 如果此时有Task已经结束则会复杂一些，不过总体来说规则是一样的。
2. 生成递增的checkpointID。
3. createPendingCheckpoint。这表示一个处于中间状态的 checkpoint，持有checkpoint所需的一些信息。
4. 计算checkpointStorageLocation。
5. triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion
	1. 依次触发所有OperatorCoordinators的Snapshot。
	2. 通知所有的OperatorCoordinator。判断checkopint的执行情况，并保存OperatorState。
		1. 这里的State是主键的State，他包含了主节点上Operator的状态，并持有指向子节点具体任务的状态。
	3. 得到`coordinatorCheckpointsComplete`。
6. 在第五步完成后，获取第3步生成的pendingCheckpoint，用于生成masterState。得到`masterStatesComplete`。
7. 等待`masterStatesComplete`和`coordinatorCheckpointsComplete`。用于等待主节点的checkpint执行完成。在进行从节点的checkpoint。
8. 如果没有异常，则触发triggerCheckpointRequest。通过triggerTasks的触发Barrier。
```java
for (Execution execution : checkpoint.getCheckpointPlan().getTasksToTrigger()) {  
	// 所有设置了trigger的。一般都是source
	if (request.props.isSynchronous()) {  
		acks.add(  
			execution.triggerSynchronousSavepoint(  
				checkpointId, timestamp, checkpointOptions));  
	} else { 
		acks.add(execution.triggerCheckpoint(checkpointId, timestamp, checkpointOptions));  
	}  
}
```
9. 最终经由TaskManagerGateway，发送到具体的Task执行triggerCheckpointBarrier方法。

## Barrier
1. 调用triggerCheckpointAsync。这是CheckpointableTask接口所定义的方法，用于sourceTask触发barrier。
	1. 同接口还定义了triggerCheckpointOnBarrier，用于一般的operator处理收到barrier后的操作。
2. 初始化一个新的`InputsCheckpoint`。subtaskCheckpointCoordinator.initInputsCheckpoint
3. 触发checkpointState方法。这里就涉及到Barrier了

### checkpointState

> Source 和 operator的处理逻辑均在这里。

1. 让下游准备处理Barrier。
	1. operatorChain.prepareSnapshotPreBarrier。
2. 生成checkpointBarrier。
	1. 意思是无论Source还是operator都会在这里生成自己的Barrier，用于向下游传递。
3. 向下游发送checkpointBarrier。
	1. operatorChain.broadcastEvent。
4. 注册对齐定时器，用于将对齐的barrier转换为未对齐的barrier。
5. 如果需要缓存in-flight数据（即非对齐）
	1. 向外部缓存数据：channelStateWriter.finishOutput。
6. 执行快照：takeSnapshotSync。
	1. 调用operatorChain.snapshotState。
	2. 具体的快照逻辑则取决于StreamOperator的snapshotState方法，并加入Future。
	3. 向operatorCoordinator发送消息，报告这个task的checkpoint已经完成。
7. 完成快照，并向JobMaster的coordinator报告情况。
	1. 具体的快照完成被放在了异步的AsyncCheckpointRunnable中。

> 注意，这里在第6步和第7步均发送了报告完成的消息，但目的地不一样。后者才是真正的完成checkpoint后发送的消息。前者则是用于在checkpoint期间对数据进行阻塞，从而保证一致性。

### Barrier的处理

### createInputProcessor
在初始化Multiple/One/TwoInputStreamTask时，会生成CheckpointBarrierHandler。上述3中InputStreamTask覆盖了所有非Source的Task。其内部是CheckpointedInputGate，会在收到Barrier消息后调用processBarrier方法。
> CheckpointedInputGate是对InputGate的进一步封装，使用的是组合的方式。

### processBarrier
根据是否是精确一次，使用的BarrierHandle会有所不同。
- SingleCheckpointBarrierHandler：精确一次。
	- 提供了了对齐和不对齐两种形式。
		- 对齐：收到一个Barrier时，阻塞这路channel，等待其他Input的Barrier。
			- 在一些老版本中，对齐会将数据收集在缓存中，而非阻塞这一路。
		- 不对齐：收到一个Barrier时，将InputBuffer和OutputBuffer中的数据一并缓存。还会缓存上游其他路算子barrier前的数据。
- CheckpointBarrierTracker：至少一次。
> 这里的Channel，指的是其接受的parition个数，如果上一步是多线程的，则即使不是shuffle，也可能有多路channel。

processBarrier用于协调具体如何应对Barrier，具体的操作则由triggerCheckpointOnBarrier方法实现。


## 执行checkpoint
在CheckpointableTask的triggerCheckpointOnBarrier时，即表示一个Operator收到了Barrier，便会执行一系列checkpoint的操作。
具体的实现放在了subtaskCheckpointCoordinator.checkpointState中，具体的操作和[[Checkpoint 机制和状态恢复#checkpointState]]一致。

# 总结

1. checkpoint由`ExecutionGraphBuilder`创建的`CheckpointCoordinator`负责。
2. 触发checkpoint时，会先生成`checkpointPlan`，此时会先触发master和coordinator的checkpoint，并等待完成。
3. 其他的checkpoint基本都由`CheckpointableTask`接口定义，其中source会调用`triggerCheckpointAsync`方法，而operator会调用`triggerCheckpointOnBarrier`方法。但最终具体的逻辑会回到`subtaskCheckpointCoordinator.checkpointState`中。其逻辑是一致的，包含了几部分：
	1. 生成`barrier`。
	2. 触发`checkpoint`和`snapshot`。
	3. 通知`coordinator`已经完成快照。


## 几个误区
1. 基于1.16的代码，可以看到Barrier由Source产生并往下传递。但每个Operator在收到Barrier后会生成自己的新的Barrier再向下传递，而非直接使用source的Barrier。
2. Barrier并不是单纯的随着数据量自己向下流动，而是通过`streamOutput.broadcastEvent`方法向下传递。


# 参考
参考： [flink-source-code-checkpoint](https://blog.jrwang.me/2019/flink-source-code-checkpoint/)
