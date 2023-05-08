一个 TaskManager 中可能会同时并行运行多个 Task，每个 Task 都在单独的线程中运行。在不同的 TaskManager 中运行的 Task 之间进行数据传输要基于网络进行通信。实际上，是 TaskManager 和另一个 TaskManager 之间通过网络进行通信，通信是基于 Netty 创建的标准的 TCP 连接，同一个 TaskManager 内运行的不同 Task 会复用网络连接。

# 数据交换的控制流

![[assets/Pasted image 20230426163910.png]]
这图展示了一个典型的Map-Reduce作业，即两个Task之间如何交互数据（shuffle场景）。其中，有两个并行的任务。有两个 TaskManager，每个 TaskManager 都分别运行一个 map Task 和一个 reduce Task。我们重点观察 M1 和 R2 这两个 Task 之间的数据传输的发起过程。数据传输用粗箭头表示，消息用细箭头表示。首先，M1 产出了一个 ResultPartition(RP1)（箭头1）。当这个 RP 可以被消费是，会告知 JobManager（箭头2）。JobManager 会通知想要接收这个 RP 分区数据的接收者（tasks R1 and R2）当前分区数据已经准备好。如果接受放还没有被调度，这将会触发对应任务的部署（箭头 3a，3b）。接着，接受方会从 RP 中请求数据（箭头 4a，4b）。这将会初始化 Task 之间的数据传输（5a,5b）,数据传输可能是本地的(5a)，也可能是通过 TaskManager 的网络栈进行（5b）。对于一个 RP 什么时候告知 JobManager 当前已经出于可用状态，在这个过程中是有充分的自由度的：例如，如果在 RP1 在告知 JM 之前已经完整地产出了所有的数据（甚至可能写入了本地文件），那么相应的数据传输更类似于 Batch 的批交换；如果 RP1 在第一条记录产出时就告知 JM，那么就是 Streaming 流交换。

> 对于非shuffle场景，flink会尽力优化为operator-chain。

# 字节缓冲区在两个 Task 之间的传输
![[assets/Pasted image 20230426163945.png]]
> 该图展示了一条记录从source到sink的全生命周期流程。

1. 最初，MapDriver 生成数据记录（通过 Collector 收集）并传递给 RecordWriter 对
	1. RecordWriter 包含一组序列化器，每个消费数据的 Task 分别对应一个。
	2. ChannelSelector 会选择一个或多个序列化器处理记录。例如，如果记录需要被广播，那么就会被交给每一个序列化器进行处理；如果记录是按照 hash 进行分区的，ChannelSelector 会计算记录的哈希值，然后选择对应的序列化器。
2. 序列化器会将记录序列化为二进制数据，并将其存放在固定大小的 buffer 中（一条记录可能需要跨越多个 buffer）。这些 buffer 被交给 BufferWriter 处理，写入到 ResultPartition（RP）中。
	1. RP 由多个子分区（ResultSubpartitions - RSs）构成，每一个子分区都只收集特定消费者需要的数据。在上图中，需要被第二个 reducer （在 TaskManager 2 中）消费的记录被放在 RS2 中。由于第一个 Buffer 已经生成，RS2 就变成可被消费的状态了（注意，这个行为实现了一个 streaming shuffle），接着它通知 JobManager。
3. JobManager查找RS2的消费者，然后通知 TaskManager 2 一个数据块已经可以访问了。通知TM2的消息会被发送到InputChannel，该inputchannel被认为是接收这个buffer的，接着通知RS2可以初始化一个网络传输了。然后，RS2通过TM1的网络栈请求该buffer，然后双方基于 Netty 准备进行数据传输。
	1. 网络连接是在TaskManager（而非特定的task）之间长时间存在的。
4. 一旦 Buffer 被 TM2 接收，它同样会经过一个类似的结构，起始于 InputChannel，进入 InputGate（它包含多个IC），最终进入一个反序列化器（RecordDeserializer），它会从 buffer 中将记录还原成指定类型的对象，然后将其传递给接收数据的 Task。

# 基本概念

## JobVertex

> 多个operator会被chain到一起形成JobVertex

`JobGraph` 是对 `StreamGraph` 进一步进行优化后得到的逻辑图，它尽量把可以 chain 到一起 operator 合并为一个 `JobVertex`。

## IntermediateDataset

> 表示一个Jobvertex的输出结果。

`JobGraph` 中对中间结果的抽象， `IntermediateDataset` 表示一个 `JobVertex` 的**输出结果**。

`JobVertex` 的输入是 `JobEdge`，而 `JobEdge` 可以看作是 `IntermediateDataset` 的消费者（上一个vertex产生的结果）。一个 `JobVertex` 也可能产生多个 `IntermediateDataset`。

> 需要说明的一点是，目前一个 `IntermediateDataset` 实际上只会有一个 `JobEdge` 作为消费者，也就是说，一个 `JobVertex` 的下游有多少 `JobVertex` 需要依赖当前节点的数据，那么当前节点就有对应数量的 `IntermediateDataset`。

其逻辑如图所示：
![[assets/IntermediateDataset.png]]
> IntermediateDataset经由edge传递

## IntermediateResult 和 IntermediateResultpartition

> 在ExecutionGraph对输出结果的抽象，IntermediateResultpartition是根据并行的子任务对IntermediateResult拆分后的结果。

`JobGraph` 被进一步转换成可以被调度的并行化版本的执行图，即 `ExecutionGraph`。在 `ExecutionGraph` 中
- 和 `JobVertex` 对应的节点是 `ExecutionJobVertex`。
- 和 `IntermediateDataset` 对应的则是 `IntermediataResult`。

由于一个节点在实际运行时可能有多个并行子任务同时运行，所以 `ExecutionJobVertex` 按照并行度的设置被拆分为多个 `ExecutionVertex`，每一个表示一个并行的子任务。同样的，一个 `IntermediataResult` 也会被拆分为多个 `IntermediateResultPartition`，`IntermediateResultPartition` 对应 `ExecutionVertex` 的输出结果。

一个 `IntermediateDataset` 只有一个消费者，那么一个 `IntermediataResult` 也只会有**一个**消费者。但是到了 `IntermediateResultPartition` 这里，由于节点被拆分成了并行化的节点，所以一个 `IntermediateResultPartition` 可能会有**多个** `ExecutionEdge` 作为消费者。

下图展示了某一个条边上的情况：
![[assets/IntermediateResult.png]]

## ResultPartition 和 ResultSubpartition
`ExecutionGraph` 还是 JobManager 中用于描述作业拓扑的一种逻辑上的数据结构，其中表示并行子任务的 `ExecutionVertex` 会被调度到 TaskManager 中执行，一个 `Task` 对应一个 `ExecutionVertex`。

`ExecutionVertex` 的输出结果 `IntermediateResultPartition` 相对应的则是 `ResultPartition`。`IntermediateResultPartition` 可能会有多个 `ExecutionEdge` 作为消费者，那么在 Task 这里，`ResultPartition` 就会被拆分为多个 `ResultSubpartition`，下游每一个需要从当前 `ResultPartition` 消费数据的 Task 都会有一个专属的 `ResultSubpartition`。


# 参考
1. [Data exchange between tasks](https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks)
2. [Task 之间的数据传输](https://blog.jrwang.me/2019/flink-source-code-data-exchange/)