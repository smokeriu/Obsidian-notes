# Flink的Checkpoint机制
Flink会在Source中生成一个Barrier的特殊消息事件，其会随着消息流入到后续的算子中。每个算子在接收到Barrier后会触发checkpoint。
具体可参考[Checkpoint 机制和状态恢复](Checkpoint%20机制和状态恢复.md)

