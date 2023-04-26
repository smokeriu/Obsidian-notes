一个 TaskManager 中可能会同时并行运行多个 Task，每个 Task 都在单独的线程中运行。在不同的 TaskManager 中运行的 Task 之间进行数据传输要基于网络进行通信。实际上，是 TaskManager 和另一个 TaskManager 之间通过网络进行通信，通信是基于 Netty 创建的标准的 TCP 连接，同一个 TaskManager 内运行的不同 Task 会复用网络连接。

# 数据交换的控制流

![[Pasted image 20230426160451.png]]
这图展示了一个典型的Map-Reduce作业，即两个Task之间如何交互数据（shuffle场景）。其中，有两个并行的任务。有两个 TaskManager，每个 TaskManager 都分别运行一个 map Task 和一个 reduce Task。我们重点观察 M1 和 R2 这两个 Task 之间的数据传输的发起过程。数据传输用粗箭头表示，消息用细箭头表示。首先，M1 产出了一个 ResultPartition(RP1)（箭头1）。当这个 RP 可以被消费是，会告知 JobManager（箭头2）。JobManager 会通知想要接收这个 RP 分区数据的接收者（tasks R1 and R2）当前分区数据已经准备好。如果接受放还没有被调度，这将会触发对应任务的部署（箭头 3a，3b）。接着，接受方会从 RP 中请求数据（箭头 4a，4b）。这将会初始化 Task 之间的数据传输（5a,5b）,数据传输可能是本地的(5a)，也可能是通过 TaskManager 的网络栈进行（5b）。对于一个 RP 什么时候告知 JobManager 当前已经出于可用状态，在这个过程中是有充分的自由度的：例如，如果在 RP1 在告知 JM 之前已经完整地产出了所有的数据（甚至可能写入了本地文件），那么相应的数据传输更类似于 Batch 的批交换；如果 RP1 在第一条记录产出时就告知 JM，那么就是 Streaming 流交换。

> 对于非shuffle场景，flink会尽力优化为operator-chain。

# 字节缓冲区在两个 Task 之间的传输
![[Pasted image 20230426161454.png]]



# 参考
1. [Data exchange between tasks](https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks)
2. [Task 之间的数据传输](https://blog.jrwang.me/2019/flink-source-code-data-exchange/)