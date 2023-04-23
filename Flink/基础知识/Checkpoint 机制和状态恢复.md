# Checkpoint的发起
在`StreamingJobGraphGenerator`中，生成JobGraph后会调用`configureCheckpointing` 方法进行 Checkpoint相关的配置。具体是`configureCheckpointing方法`。
这里会将：状态后端，Checkpoint配置一并配置。生成一个JobCheckpointingSettings。

在 `ExecutionGraphBuilder#buildGraph` 中，如果作业开启了 checkpoint，则会调用 `ExecutionGraph.enableCheckpointing()` 方法, 这里会创建 `CheckpointCoordinator` 对象，并注册一个作业状态的监听 `CheckpointCoordinatorDeActivator`