Spark可以通过`--files`参数或`spark.files`配置，向应用传递本地文件，包括如下格式：
1. `file:/`或`/path/to/file`，则文件会在`spark-submit`阶段发送给Driver的Http server。Executor通过这个server获取文件。
2. `hdfs:/`、`http:/`等网络地址：由executor自行下载。
3. `local:/`：executor会将其当作ji qi uh de