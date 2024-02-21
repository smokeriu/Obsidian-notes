ClassLoader是用于加载我们编译好的class文件到JVM中。

# 双亲委派
涉及到三类ClassLoader：
- BootstrapClassLoader：负责加载`rt.jar`包中的class文件。 C++写的，所以在java不可见。
- ExtClassLoader：负责加载ext目录的jar包中的class文件。
- AppClassLoader：负责加载classpath下的class文件，即**我们编写**的java文件编译成的class文件。

双亲委派模型是指，在加载class时， **优先**让父classLoader加载class，如果父classLoader加载不到，再自己加载。 这里的“父”， 不是指继承关系，而是指属性指定的“父”ClassLoader。具体而言，AppClassLoader的父加载器是ExtClassLoader，ExtClassLoader的父加载器是BootstrapClassLoader。

而双亲委派主要是保证**应用安全**，简单而言，