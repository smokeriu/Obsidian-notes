主要由`OptimizingPlanner`负责
# 构建Evaluator
在完成[[扫描文件]]后，系统会将扫描的文件加入到Evaluator中：
```Java
for (TableFileScanHelper.FileScanResult fileScanResult : results) {
	evaluator.addFile(fileScanResult.file(), fileScanResult.deleteFiles());
}
```

在`addFile`方法中，会区分文件类型：
```java
boolean added = evaluator().addFile(dataFile, deletes);  
if (added) {  
  if (evaluator().fileShouldRewrite(dataFile, deletes)) {  
    rewriteDataFiles.put(dataFile, deletes);  
  } else if (evaluator().isUndersizedSegmentFile(dataFile)) {  
    undersizedSegmentFiles.put(dataFile, deletes);  
  } else if (evaluator().segmentShouldRewritePos(dataFile, deletes)) {  
    rewritePosDataFiles.put(dataFile, deletes);  
  } else {  
    added = false;  
  }  
}  
if (!added) {  
  reservedDeleteFiles(deletes);  
}
```

> 需要注意的是，`OptimizingPlanner`中的Evaluator使用的是`AbstractPartitionPlan`的子类。

数据会被记录在四个`Map<DataFile, List<ContentFile<?>>>`类型的Map中。其依赖的分类如下：
- rewriteDataFiles
	- 开启了`Full optimize`。
	- 是Fragment类型的文件。
	- delete中，`pos-delete`类型的数量达到了阈值。
- undersizedSegmentFiles：
	- Segment类型的文件。
- rewritePosDataFiles：
	- delete中有两个以上的文件属于`pos-delete`。
	- delete中有任意一个`eq-delete`的文件。
- reservedDeleteFiles：
	- 如果DataFile没被添加到上述3个Map中的任意一个。
	- 则需要将delete添加到reservedDeleteFiles中，避免被移除。

## CommonPartitionEvaluator
另一方面，代码中的`evaluator()`返回`CommonPartitionEvaluator`类型，其添加文件代码如下：
```java
if (isFragmentFile(dataFile)) {  
  return addFragmentFile(dataFile, deletes);  
} else if (isUndersizedSegmentFile(dataFile)) {  
  return addUndersizedSegmentFile(dataFile, deletes);  
} else {  
  return addTargetSizeReachedFile(dataFile, deletes);  
}
```

其中：
- addFragmentFile：
	- 只会添加delete
	- 记录dataFile的大小。
- addUndersizedSegmentFile：
	- 只会添加delete。
	- 如果是`full optimize`阶段，则记录dataFile的大小。
- addTargetSizeReachedFile
	- 如果是`full optimize`阶段，则记录dataFile的大小。并添加delete。

总而言之，`CommonPartitionEvaluator`只会记录delete文件。此时的`Plan`实际包含了两层。
- 第一层是`AbstractPartitionPlan`。
- 第二层则是`CommonPartitionEvaluator`。

# 构建Task

上述构建的Plan，通过Split方法，构建`TaskDescriptor`：
```Java
List<TaskDescriptor> tasks = Lists.newArrayList();  
for (AbstractPartitionPlan partitionPlan : actualPartitionPlans) {  
  tasks.addAll(partitionPlan.splitTasks((int) (actualInputSize /
				   avgThreadCost)));  
}
```

## TaskSplitter
用于生成`SplitTask`，以`TreeNodeTaskSplitter`为例：
1. 将`undersizedSegmentFiles`中的文件加入到rootTree的`RewriteDataFile`中。
2. 将`rewritePosDataFiles`中的文件，加入到`RewritePosDataFile`中。
3. 将`rewriteDataFiles`中的文件，加入到`RewriteDataFile`中。

上述数据会按照一定逻辑处理，分为三部分：
- rewriteDataFiles：
	- 来自上一步中，加入到`rewriteDataFile`中的文件。
- rewritePosDataFiles：
	- 来自上一步中，加入到`rewritePosDataFile`中的文件。
- deleteFiles：
	- 来自`rewriteDataFiles`和`rewritePosDataFiles`中的所有deletes。

TaskSplitter将生成SplitTask，其是`AbstractPartitionPlan`的内部类。
## buildTask
buildTask是SplitTask的方法，用于构建TaskDescriptor。
方法内定义了两个集合：
```java
Set<ContentFile<?>> readOnlyDeleteFiles = Sets.newHashSet();  
Set<ContentFile<?>> rewriteDeleteFiles = Sets.newHashSet();
```

`deleteFiles`会根据其在`reservedDeleteFiles`的情况分别加入到`readOnlyDeleteFiles`和`rewriteDeleteFiles`中：

```java
if (reservedDeleteFiles.contains(deleteFile.path().toString())) {  
  readOnlyDeleteFiles.add(deleteFile);  
} else {  
  rewriteDeleteFiles.add(deleteFile);  
}
```
紧接着构建`RewriteFilesInput`：
```java
new RewriteFilesInput(  
    rewriteDataFiles.toArray(new DataFile[0]),  
    rewritePosDataFiles.toArray(new DataFile[0]),  
    readOnlyDeleteFiles.toArray(new ContentFile[0]),  
    rewriteDeleteFiles.toArray(new ContentFile[0]),  
    tableObject);
```

由此可见RewriteFilesInput的构成：
1. rewrittenDataFiles：
	1. Segment类型的文件。
	2. Fragment的文件。
	3. 或满足其他要求的，例如开启`Full Optimize`。
2. rePosDeletedDataFiles：
	1. 存在`pos-delete`或`eq-delete`的原始文件（非delete文件）。
3. readOnlyDeleteFiles：不应该被处理的delete文件。
4. rewrittenDeleteFiles：应该被处理的delete文件。

