有些时候，需要在代码中识别这段代码执行在Driver，还是Executor，从而决定后续的处理逻辑，例如使用[[Spark的files参数]]时。

`SparkSession.getActiveSession`返回值是一个Option，其注视中提到：
> Returns the active SparkSession for the current thread, returned by the builder.  
   **Return None, when calling this function on executors**。

所以在初始化后，通过`SparkSession.getActiveSession.isDefined`就可以判断当前代码片段是在Driver还是Executor了。
