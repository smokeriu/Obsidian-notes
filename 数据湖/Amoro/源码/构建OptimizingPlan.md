# 构建Evaluator
在完成[[扫描文件]]后，系统会将扫描的文件加入到Evaluator中：
```Java
for (TableFileScanHelper.FileScanResult fileScanResult : results) {
	evaluator.addFile(fileScanResult.file(), fileScanResult.deleteFiles());
}
```

在`addFile`方法中，会区分文件类型：
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

