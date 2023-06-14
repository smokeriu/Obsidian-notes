> 基础知识: [[Task 之间的数据传输]]

数据交换从本质上来说就是一个典型的**生产者-消费者**模型，上游算子生产数据到 `ResultPartition` 中，下游算子通过 `InputGate` 消费数据。由于消费的数据可能来源于同一个TaskManager中运行的不同Task，也可能来自于其他节点的Task，所以Flink通过合理的抽象，使核心代码能够复用。

# Task的输入和输出
## Task的输出
输出会写入到`ResultPartition` 中，其实现了ResultPartitionWriter。

```java
// ResultPartition.java
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

> 在Flink1.16中，ResultPartition由Task通过shuffleEnvironment创建。

```java
// Task.java
final ResultPartitionWriter[] resultPartitionWriters =  
	shuffleEnvironment  
		.createResultPartitionWriters(  
			taskShuffleContext, resultPartitionDeploymentDescriptors)  
		.toArray(new ResultPartitionWriter[] {});
```

目前唯一的实现是`NettyShuffleEnvironment`。其内部由ResultPartitionFactory统一负责ResultPartition和ResultPartition的创建

#### ResultPartition
```java
// NettyShuffleEnvironment.java
ResultPartition[] resultPartitions =  
	new ResultPartition[resultPartitionDeploymentDescriptors.size()];

for (int partitionIndex = 0;  
		partitionIndex < resultPartitions.length;  
		partitionIndex++) {  
	resultPartitions[partitionIndex] =  
		resultPartitionFactory.create(  
			ownerContext.getOwnerName(),  
			partitionIndex,  
			resultPartitionDeploymentDescriptors.get(partitionIndex));  
}
```
#### SubResultPartition

```java
// ResultPartitionFactory.java
final ResultPartition partition;

ResultSubpartition[] subpartitions = new ResultSubpartition[num];
if (type == ResultPartitionType.PIPELINED_APPROXIMATE) {  
	subpartitions[i] =  
		new PipelinedApproximateSubpartition(  
			i, configuredNetworkBuffersPerChannel, pipelinedPartition);  
} else {  
	subpartitions[i] =  
		new PipelinedSubpartition(  
			i, configuredNetworkBuffersPerChannel, pipelinedPartition);  
}

partition = new PipelinedResultPartition(...subpartitions);

return partition;
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
1. 决定输出的targetSubPartition，其等价于targetChannel。
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
// 3. 请求buffer，提交序列化后的数据
public void emitRecord(ByteBuffer record, int targetSubpartition) throws IOException {  
	// 尝试写入数据
	BufferBuilder buffer = appendUnicastDataForNewRecord(record, targetSubpartition);  
	// 由于实际可用的内存可能不足以承载单条消息，所以这里使用的while循环
	while (record.hasRemaining()) {  
		// buffer已经写满，但这条记录存在部分数据没有写入，则flush现有buffer
		finishUnicastBufferBuilder(targetSubpartition);
		// 申请新的buffer，并立即flush这条未完成的部分
		buffer = appendUnicastDataForRecordContinuation(record, targetSubpartition);  
	}  
  
	if (buffer.isFull()) {  
		// full buffer, full record  
		finishUnicastBufferBuilder(targetSubpartition);  
	}  
}

// 具体的写入逻辑
private BufferBuilder appendUnicastDataForNewRecord(  
	final ByteBuffer record, final int targetSubpartition) throws IOException {  
	// 申请buffer
	BufferBuilder buffer = unicastBufferBuilders[targetSubpartition];  
  
	if (buffer == null) {
		// 新建buffer
		buffer = requestNewUnicastBufferBuilder(targetSubpartition);  
		// 使buffer与consumer关联
		addToSubpartition(buffer, targetSubpartition, 0, record.remaining());  
	}  
	// 写入数据
	buffer.appendAndCommit(record);  
  
	return buffer;  
}

// 关联buffer、consumer与subPartition
private void addToSubpartition(...){
	BufferConsumer bufferConsumer = buffer.createBufferConsumerFromBeginning();
	
	int desirableBufferSize =
		subpartitions[targetSubpartition].add(
			bufferConsumer, partialRecordLength);
}
```
在完成写出后，根据设置，可能立即flush数据，也可能通过如OutputFlusher等其他线程触发flush。flush由subPartition负责，当数据flush后，则下游便可见。
```java
// PipelinedSubpartition.java
public void flush() {  
	final boolean notifyDataAvailable;  
	// buffer为下游的comsumer
	synchronized (buffers) {  
		if (buffers.isEmpty() || flushRequested) {  
			return;  
	}  
	// if there is more then 1 buffer, we already notified the reader  
	// (at the latest when adding the second buffer)  
		boolean isDataAvailableInUnfinishedBuffer =  
			buffers.size() == 1 && buffers.peek().getBufferConsumer().isDataAvailable();  
			
	notifyDataAvailable = !isBlocked && isDataAvailableInUnfinishedBuffer;  
	flushRequested = buffers.size() > 1 || isDataAvailableInUnfinishedBuffer;  
	}  
	if (notifyDataAvailable) {  
		// 通知下游
		notifyDataAvailable();  
	}  
}

private void notifyDataAvailable() {  
	final PipelinedSubpartitionView readView = this.readView;  
	if (readView != null) {  
		readView.notifyDataAvailable();  
	}  
}
```

实际的数据读取由`ResultSubpartitionView`负责。subPartition会通知与自身关联的`ResultSubpartitionView`数据已经可读。


## 输入和输出的关联
输入会通过与subPartition关联，并构建`ResultSubpartitionView`：
```java
// 本地情况
// LocalInputChannel.java
// 将自身传递给subpartitionView，用作数据可读的通知。
ResultSubpartitionView subpartitionView =  
	partitionManager.createSubpartitionView(  
		partitionId, consumedSubpartitionIndex, this);

// 分布式情况
// CreditBasedSequenceNumberingViewReader.java
// 将自身传递给subpartitionView，用作数据可读的通知。
this.subpartitionView =  
	partitionProvider.createSubpartitionView(  
		resultPartitionId, subPartitionIndex, this);

// ResultPartitionManager.java
subpartitionView =  
	partition.createSubpartitionView(subpartitionIndex, availabilityListener);

// BufferWritingResultPartition.java
ResultSubpartitionView readView = subpartition.createReadView(availabilityListener);
```

当输出数据可读时，即会通过`ResultSubpartitionView`让consumer直到数据可读，此时，数据仍然存放在`subPartition`的`BufferConsumer`中。
LocalInputChannel.java和CreditBasedSequenceNumberingViewReader都实现了BufferAvailabilityListener，用于及时直到上游数据可读。


## Task的输入
`notifyDataAvailable`会通知`InputChannel`，即数据可读。目前主要是两种实现：
- localInputChannel：本地数据交互。
- CreditBasedSequenceNumberingViewReader：涉及shuffle时的数据交互。

### Local
```java
// InputChannel.java
// 本地buffer采取这种方式，通知inputGate
protected void notifyChannelNonEmpty() {  
	inputGate.notifyChannelNonEmpty(this);  
}
void notifyChannelNonEmpty(InputChannel channel) {  
	queueChannel(checkNotNull(channel), null, false);  
}
// 将channel加入到队列中
private boolean queueChannelUnsafe(InputChannel channel, boolean priority) {
	// ...
	inputChannelsWithData.add(channel, priority, alreadyEnqueued);
}
// 读取数据
private Optional<InputWithData<InputChannel, BufferAndAvailability>> waitAndGetNextData(...){
	return Optional.of(  
		new InputWithData<>(  
			inputChannel,  
			bufferAndAvailability,  
			!inputChannelsWithData.isEmpty(),  
			morePriorityEvents));
}
```
通过上述处理，将channel添加到gate的缓存中。

### Network
```java
// CreditBasedSequenceNumberingViewReader.java
// 分布式采取这种方式
public void notifyDataAvailable() {  
	requestQueue.notifyReaderNonEmpty(this);  
}

void notifyReaderNonEmpty(final NetworkSequenceViewReader reader) {   
	ctx.executor().execute(() -> 
		ctx.pipeline().fireUserEventTriggered(reader));  
}

// 经过netty中转，将reader加入到可读的队列中。
public void userEventTriggered(ChannelHandlerContext ctx, Object msg) throws Exception {
	if (msg instanceof NetworkSequenceViewReader) {
		enqueueAvailableReader((NetworkSequenceViewReader) msg);
	}
}

private void enqueueAvailableReader(final NetworkSequenceViewReader reader) throws Exception {
	registerAvailableReader(reader);
}

private void registerAvailableReader(NetworkSequenceViewReader reader) {  
	availableReaders.add(reader);  
	reader.setRegisteredAsAvailable(true);  
}
```
相比于本地，网络还涉及从实际的机器上获取数据：
```java
NetworkSequenceViewReader reader = pollAvailableReader();
next = reader.getNextBuffer();

public BufferAndAvailability getNextBuffer() throws IOException {  
	BufferAndBacklog next = subpartitionView.getNextBuffer();  
  
	final Buffer.DataType nextDataType = getNextDataType(next);  
	return new BufferAndAvailability(  
		next.buffer(), nextDataType, next.buffersInBacklog(), next.getSequenceNumber());  
	} else {  
		return null;  
	}
}

BufferResponse msg =  
	new BufferResponse(  
		next.buffer(),  
		next.getSequenceNumber(),  
		reader.getReceiverId(),  
		next.buffersInBacklog());

channel.writeAndFlush(msg).addListener(writeListener);

```
这里便将数据信息向网络写出，其他节点接收到对应的信息，并获取inputChannel。
```java
// CreditBasedPartitionRequestClientHandler.java
if (msgClazz == NettyMessage.BufferResponse.class) {
	NettyMessage.BufferResponse bufferOrEvent = (NettyMessage.BufferResponse) msg;
	RemoteInputChannel inputChannel = inputChannels.get(bufferOrEvent.receiverId);
	decodeBufferOrEvent(inputChannel, bufferOrEvent);
}

// 处理数据
private void decodeBufferOrEvent(  
RemoteInputChannel inputChannel, NettyMessage.BufferResponse bufferOrEvent)  
throws Throwable {  
	if (bufferOrEvent.getBuffer() != null) {  
		inputChannel.onBuffer(  
			bufferOrEvent.getBuffer(), bufferOrEvent.sequenceNumber, bufferOrEvent.backlog);  
	}
}

// RemoteInputChannel.java
public void onBuffer(Buffer buffer, int sequenceNumber, int backlog) throws IOException {
	SequenceBuffer sequenceBuffer = new SequenceBuffer(buffer, sequenceNumber);
	receivedBuffers.add(sequenceBuffer);
}
```
通过上述处理，其他机器上的数据便缓存到该节点，用作后续读取需要。

## 读取数据

会通过调用getNextBuffer方法来持续的获取数据Buffer。

```java
abstract Optional<BufferAndAvailability> getNextBuffer()  
	throws IOException, InterruptedException;
```

不同的InputChannel实现会有所不同，但本质就是把缓存的buffer提取出来。调用getNextBuffer的是`InputGate`的`waitAndGetNextData`方法。

> 不同`InputGate`的实现会有所区别

```java
// SingleInputGate.java
private Optional<InputWithData<InputChannel, BufferAndAvailability>> waitAndGetNextData(  
	boolean blocking) throws IOException, InterruptedException {
	
	final InputChannel inputChannel = inputChannelOpt.get();
	// getNextBuffer
	Optional<BufferAndAvailability> bufferAndAvailabilityOpt =  
		inputChannel.getNextBuffer();
		
	final BufferAndAvailability bufferAndAvailability = bufferAndAvailabilityOpt.get();

	return Optional.of(  
		new InputWithData<>(  
			inputChannel,  
			bufferAndAvailability,  
			!inputChannelsWithData.isEmpty(),  
			morePriorityEvents));
}

// 再通过getNextBufferOrEvent封装一层
private Optional<BufferOrEvent> getNextBufferOrEvent(boolean blocking)  
	throws IOException, InterruptedException {

	// waitAndGetNextData
	Optional<InputWithData<InputChannel, BufferAndAvailability>> next =  
		waitAndGetNextData(blocking);

	InputWithData<InputChannel, BufferAndAvailability> inputWithData = next.get();  
	final BufferOrEvent bufferOrEvent =  
		transformToBufferOrEvent(  
			inputWithData.data.buffer(),  
			inputWithData.moreAvailable,  
			inputWithData.input,  
			inputWithData.morePriorityEvents);

	return Optional.of(bufferOrEvent);
}
```

实际对外提供数据的是`InputGate`的`getNext`或`pollNext`方法，二者的主要区别在于是否阻塞。

而实际读取数据的抽象是`AbstractRecordReader`。其在`getNextRecord`方法中读取数据：
```java
protected boolean getNextRecord(T target) throws IOException, InterruptedException{

	while(true){

		if(currentRecordDeserializer != null){
			// 实际获取数据的地方， 数据其实会在里面被存放仅target中。
			DeserializationResult result = currentRecordDeserializer.getNextRecord(target);
			
		}

		final BufferOrEvent bufferOrEvent =  
			inputGate.getNext().orElseThrow(IllegalStateException::new);


		if (bufferOrEvent.isBuffer()) {  
			// 将buffer设置到currentRecordDeserializer中。
			currentRecordDeserializer = recordDeserializers.get(bufferOrEvent.getChannelInfo());  
			currentRecordDeserializer.setNextBuffer(...);  
			partialData.put(currentRecordDeserializer, Boolean.TRUE);  
		}
	}

}
```

```java
// SpillingAdaptiveSpanningRecordDeserializer.java
private DeserializationResult readNextRecord(T target) throws IOException {  
	if (nonSpanningWrapper.hasCompleteLength()) {  
		// 读取数据
		return readNonSpanningRecord(target);  
	} else if (nonSpanningWrapper.hasRemaining()) {  
		nonSpanningWrapper.transferTo(spanningWrapper.lengthBuffer);  
		return PARTIAL_RECORD;  
	} else if (spanningWrapper.hasFullRecord()) {  
		// 读取数据
		target.read(spanningWrapper.getInputView());  
		spanningWrapper.transferLeftOverTo(nonSpanningWrapper);  
		return nonSpanningWrapper.hasRemaining()  
			? INTERMEDIATE_RECORD_FROM_BUFFER  
			: LAST_RECORD_FROM_BUFFER;  
	} else {  
		return PARTIAL_RECORD;  
	}  
}

private DeserializationResult readNonSpanningRecord(T target) throws IOException {  
	int recordLen = nonSpanningWrapper.readInt();  
	if (nonSpanningWrapper.canReadRecord(recordLen)) {  
		// 读取数据
		return nonSpanningWrapper.readInto(target);  
	} else {  
		spanningWrapper.transferFrom(nonSpanningWrapper, recordLen);  
		return PARTIAL_RECORD;  
	}  
}

```

实际数据如何读取有target控制，但总体而言，是通过DataInputView获取数据的。而实际数据通过buffer交给了NonSpanningWrapper或SpanningWrapper

```java
public void setNextBuffer(Buffer buffer) throws IOException {  
	currentBuffer = buffer;  
	// 获取数据的内存地址，并交给wrapper
	int offset = buffer.getMemorySegmentOffset();  
	MemorySegment segment = buffer.getMemorySegment();  
	int numBytes = buffer.getSize();  
  
// check if some spanning record deserialization is pending  
	if (spanningWrapper.getNumGatheredBytes() > 0) {  
		spanningWrapper.addNextChunkFromMemorySegment(segment, offset, numBytes);  
	} else {  
		nonSpanningWrapper.initializeFromMemorySegment(segment, offset, numBytes + offset);  
	}  
}
```