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

> 需要注意的是，

其中：
- addFragmentFile：
	- 只会添加delete
	- 记录dataFile的大小。
- addUndersizedSegmentFile：
	- 只会添加delete。
	- 如果是`full optimize`阶段，则记录dataFile的大小。
- addTargetSizeReachedFile
	- 如果是`full optimize`阶段，则记录dataFile的大小。并添加delete。

# 构建Task

