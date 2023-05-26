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
	// subPartition的数量， 一般为下游reduce的数量，
	// 在1.16的版本中，具体的subPartition由子类自行管理
	protected final int numSubpartitions;
	// 发生shuffle时，需要根据下游的redcuer数量来按照key进行分析，
	// 从而计算出每条记录发往那个channel
	// 在目前实现中，其总是等于最大并行度。
	private final int numTargetKeyGroups;
	// 用于提供bufferPool
	private final SupplierWithException<BufferPool, IOException> bufferPoolFactory;
}
```

> 在flink1.16下，SubPartition的创建由ResultPartitionFactory统一负责。

```java
if (type == ResultPartitionType.PIPELINED_APPROXIMATE) {  
	subpartitions[i] =  
		new PipelinedApproximateSubpartition(  
			i, configuredNetworkBuffersPerChannel, pipelinedPartition);  
} else {  
	subpartitions[i] =  
		new PipelinedSubpartition(  
			i, configuredNetworkBuffersPerChannel, pipelinedPartition);  
}
```

### 数据的写出
数据写出由`RecordWriter`负责，其是`ResultPartition`的一层封装。
```java
public abstract class RecordWriter<T extends IOReadableWritable> implements AvailabilityProvider {
	protected final ResultPartitionWriter targetPartition;
	// channels数量，等于subPartition数量
	protected final int numberOfChannels;
	// 序列化产生的数据
	protected final DataOutputSerializer serializer;
}
```

> 目前实现中，子类包含了`BroadcastRecordWriter`和`ChannelSelectorRecordWriter`，这里我们主要讨论后者。

主要流程为：
1. 决定输出的targetSubPartition。其等价于targetChannel。
2. 序列化将要输出的数据。
3. 向ResultPartition请求buffer，执行数据的写入。
4. 向ResultPartition请求Consumer，将数据向下流。

> 不同的ResultPartition实现有区别，这里以BufferWritingResultPartition举例。

其代码为：
```java
// ChannelSelectorRecordWriter.java
// 1. 决定输出的targetSubPartition
public void emit(T record) throws IOException {  
	emit(record, channelSelector.selectChannel(record));  
}
// RecordWriter.java
// 2. 序列化数据
protected void emit(T record, int targetSubpartition) throws IOException {    
	targetPartition.emitRecord(serializeRecord(serializer, record), targetSubpartition);
}

// BufferWritingResultPartition.java
// 3. 提交序列化后的数据
public void emitRecord(ByteBuffer record, int targetSubpartition) throws IOException {  
	// 申请buffer
	BufferBuilder buffer = appendUnicastDataForNewRecord(record, targetSubpartition);  
  
	while (record.hasRemaining()) {  
		// full buffer, partial record  
		finishUnicastBufferBuilder(targetSubpartition);  
		buffer = appendUnicastDataForRecordContinuation(record, targetSubpartition);  
	}  
  
	if (buffer.isFull()) {  
		// full buffer, full record  
		finishUnicastBufferBuilder(targetSubpartition);  
	}  
  
// partial buffer, full record  
}
```

## Task的输入