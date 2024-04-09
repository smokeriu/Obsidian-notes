在完成[[扫描文件]]后，系统会将扫描的文件加入到Evaluator中：
```Java
for (TableFileScanHelper.FileScanResult fileScanResult : results) {
	evaluator.addFile(fileScanResult.file(), fileScanResult.deleteFiles());
}
```

