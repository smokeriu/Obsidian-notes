Spark可以通过`--files`参数或`spark.files`配置，向应用传递本地文件，包括如下格式：
1. `file:/`或`/path/to/file`，则文件会在`spark-submit`阶段发送给Driver的Http server。Executor通过这个server获取文件。
2. `hdfs:/`、`http:/`等网络地址：由executor自行下载。
3. `local:/`：executor会将其当作机器上的绝对路径处理。

# 坑
## Driver的地址
一般会通过`SparkFiles.get(fileName)`来获取通过`--files`传输的文件，这里有两个注意点：
1. Driver执行这段代码，和Executor执行这段代码，得到的结果是不一样的。
2. Driver得到的地址，实际上是**没有文件的！**。这是因为Driver实际拿到的是`SparkEnv.get.driverTmpDir+fileName`，而driver的临时目录会被清空，我们需要使用工作目录。而对于JVM来说，工作目录可以直接使用`fileName`定位，所以可以通过这个特性，间接获取其绝对路径。

简而言之：
- 当*Executor*要使用文件时，需要在具体的Executor执行`SparkFiles.get(fileName)`来获取其在Executor上的地址。
- 当*Driver*要使用文件时，不能使用`SparkFiles.get(fileName)`来获取地址，可以使用`new File(fileName).getAbsolutePath()`来获取地址。

参考：
- [spark 带文件上集群，获取外部文件，--files 使用说明](https://blog.csdn.net/Code_LT/article/details/132225997)