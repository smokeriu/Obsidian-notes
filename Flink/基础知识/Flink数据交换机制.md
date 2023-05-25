> 基础知识: [[Task 之间的数据传输]]

数据交换从本质上来说就是一个典型的**生产者-消费者**模型，上游算子生产数据到 `ResultPartition` 中，下游算子通过 `InputGate` 消费数据。由于消费的数据可能来源于同一个TaskManager中运行的不同Task，也可能来自于其他节点的Task，所以Flink通过合理的抽象，使核心代码能够复用。

# Task的输入和输出
## Task的输出
输出会写入到`ResultPartition` 中，其实现了ResultPartitionWriter。

```java
public abstract class ResultPartition implements ResultPartitionWriter {
	// 类型，Flink根据是否锁、有界、持久化等维度定义了多种类型
	protected final ResultPartitionType partitionType;
	// 管理TaskManager上的所有ResultPartition
	protected final ResultPartitionManager partitionManager;
	// subPartition的数量，在1.16的版本中，具体的subPartition由子类自行管理
	protected final int numSubpartitions;
	
	private final int numTargetKeyGroups;
}
```


## Task的输入