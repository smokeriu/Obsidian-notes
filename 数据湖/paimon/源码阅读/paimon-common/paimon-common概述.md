用于提供一些顶层API和通用方法。主体上分为
- annotation：提供一些常用的注释。
- codegen：代码生成的相关工具或抽象接口定义。
- catalog：目前仅提供了catalog的上下文。
- data：数据类型的定义，包括：
	- columnar：列存类型。
	- serializer：序列化器。
	- 一些额外的高级数据类型，如BinaryMap等。基础数据类型则使用的是Java的基本类型，如Double，Long。
- format：
	- 