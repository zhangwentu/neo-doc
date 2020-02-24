## JVM是什么

`JVM`是`Java Virtual Machine`(Java虚拟机)的缩写，`JVM`是一种用于计算设备的**规范**，它是一个**虚构**的计算机，是通过在实际的计算机上仿真模拟各种计算机功能来实现的。

**JVM本质上也是一个程序，屏蔽了与具体操作系统平台相关的信息**，使`Java`程序只需生成在`Java`虚拟机上**一次编译，多次运行，具有跨平台性**。`JVM`在执行字节码时，实际上最终还是把**字节码**解释成具体平台上的**机器指令**执行。

`Java`虚拟机包括一套**字节码指令集**、一组**寄存器**、一个**栈**、一个**垃圾回收堆**和一个**存储方法区**。



### JDK、JRE和JVM

Java语言的三驾马车就是jdk、jre和Jvm

- `JDK`(Java Development Kit) 是 `Java` 语言的软件开发工具包（`SDK`）,是整个Java的核心，包括了Java运行环境JRE*（Java Runtime Envirnment）*，和一些Java `programming tools`  （下载安装后可以发现Java目录下有个jre和其他工具）
- `JRE`（Java Runtime Environment）`Java` 运行时环境，`JRE` 是物理存在的，主要由`Java API` 、 `JVM` 组成还包括一些**其他支持文件**，提供了用于执行 `java` 应用程序**最低**要求的环境。
- `JVM`是一种用于计算设备的**规范**，它是一个**虚构**的计算机的软件实现，简单的说，`JVM`是运行`byte code`字节码程序的一个容器。

**Jdk 和 JRE 是真实存在的，而 JVM 是一个抽象的概念，并不真实存在。**

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybply1gbvq0j2qtdj314t0u0tbo.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybply1gbvq0j2qtdj314t0u0tbo.jpg" width=""/>
    </a>
</p>

### JVM字节码

`JVM`使用Java字节码（即.class文件）的方式，作为`Java` **用户语言** 和 **机器语言** 之间的中间语言。实现一个**通用的**、 **机器无关** 的执行平台。

## JVM功能

- 加载代码
- 验证代码
- 执行代码
- 提供运行环境

## JVM生命周期

**启动**：任何一个拥有`main`方法的`class`都可以作为`JVM`实例运行的起点。

**运行**：`main`函数为起点，程序中的其他线程均有它启动，包括`daemon`守护线程和`non-daemon`普通线程。`daemon`是`JVM`自己使用的线程比如`GC`线程，`main`方法的初始线程是`non-daemon`。

**消亡**：所有线程终止时，`JVM`实例结束生命。

## JVM运行过程

代码执行过程如下：

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbvs0ghdkbj30v40pwah4.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbvs0ghdkbj30v40pwah4.jpg" width=""/>
    </a>
</p>

### 1. 类加载器（Class Loader）

**类加载器** 负责加载程序中的类型（类和接口），并赋予唯一的名字予以标识。

#### 加载器初始化顺序
`JDK` 默认提供的三种 `ClassLoader ` 加载顺序如下：

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbvs2k2ky1j30lg0f643v.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbvs2k2ky1j30lg0f643v.jpg" width=""/>
    </a>
</p>
- Bootstrap Classloader 是在Java虚拟机启动后初始化的。
- Bootstrap Classloader 负责加载 ExtClassLoader，并且将 ExtClassLoader的父加载器设置为 Bootstrap Classloader
- Bootstrap Classloader 加载完 ExtClassLoader 后，就会加载 AppClassLoader，并且将 AppClassLoader 的父加载器指定为 ExtClassLoader。

#### 类加载器的作用

|            Class Loader | 实现 | 负责加载                                                     |
| ----------------------: | :--- | :----------------------------------------------------------- |
|        Bootstrap Loader | C++  | `%JAVA_HOME%/jre/lib`, `%JAVA_HOME%/jre/classes`以及-Xbootclasspath参数指定的路径以及中的类 |
|   Extension ClassLoader | Java | `%JAVA_HOME%/jre/lib/ext`，路径下的所有`classes`目录以及`java.ext.dirs`系统变量指定的路径中类库 |
| Application ClassLoader | Java | `Classpath`所指定的位置的类或者是`jar`文档，它也是`Java`程序默认的类加载器 |

#### 双亲委托机制

`Java`中`ClassLoader`的加载采用了**双亲委托机制**，采用**双亲委托机制**加载类的时候采用如下的几个步骤：

1. 当前`ClassLoader`首先从自己已经加载的类中查询是否此类已经加载，如果已经加载则直接返回原来已经加载的类。
2. 当前`ClassLoader`的缓存中没有找到被加载的类的时候，委托父类加载器去加载，父类加载器采用同样的策略，首先查看自己的缓存，然后委托父类的父类去加载，一直到`Bootstrap ClassLoader`。
3. 当所有的父类加载器都没有加载的时候，再由当前的类加载器加载，并将其放入它自己的缓存中，以便下次有加载请求的时候直接返回。

> **小结** ：双亲委托机制的核心思想分为两个步骤。其一，自底向上检查类是否已经加载；其二，自顶向下尝试加载类。



### 2. 执行引擎（Execution Engine）