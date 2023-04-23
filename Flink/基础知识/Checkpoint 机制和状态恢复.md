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