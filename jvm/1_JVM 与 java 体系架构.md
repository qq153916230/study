> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1PJ411n7xZ/?p=5

# 1. JVM官方规范下载与参考书目

java语言和虚拟机官方规范：https://docs.oracle.com/javase/specs/index.html

JVM官方规范翻译书：《java虚拟机规范》，可以作为查阅

JVM学习推荐书：《深入理解Java虚拟机》作者：周志明

# 2. JVM 整体架构

![](D:\mine\study\JVM\pic\1.png)

详细图：

![](D:\mine\study\JVM\pic\3.jpg)

# 3. java 代码执行流程

![](D:\mine\study\JVM\pic\4.jpg)

![](D:\mine\study\JVM\pic\5.png)

![](D:\mine\study\JVM\pic\6.jpg)

# 4. JVM 的架构模型

Java 编译器输入的指令流基本上是一种基于**栈的指令集架构**，另外一种指令集架构则是基于**寄存器的指令集架构**。

具体来说，这两种架构之间的区别：

- **基于栈式架构的特点**
  - 设计和实现更简单，适用于资源受限的系统
  - 避开了寄存器的分配难题：使用零地址指令方式分配
  - 指令流中的指令大部分是零地址指令，其执行过程依赖于操作栈，指令集更小，编译器容易实现
  - 不需要硬件支持，可移植性更好，更好实现跨平台

- **基于寄存器架构的特点**
  - 典型的应用是 x86 的二进制指令集：比如传统的 PC 以及 Android 的 Davlik 虚拟机
  - **指令集架构则完全依赖硬件，可移植性差**
  - **性能优秀和执行更高效**
  - 花费更少的指令去完成一项操作
  - 在大部分情况下，基于寄存器架构的指令集往往都以一地址指令、二地址指令和三地址指令为主，而基于栈式架构的指令集却是以零地址指令为主

![](D:\mine\study\JVM\pic\7.jpg)

总结：

**由于跨平台性的设计，Java 的指令都是根据栈来设计的**。不同平台 CPU 架构不同，所以不能设计为基于寄存器的。优点是跨平台，指令集小，编译器容易实现，缺点是性能下降，实现同样的功能需要更多的指令。

栈：**跨平台性、指令集小、指令多;执行性能比寄存器差**

# 5. JVM 的生命周期

- **虚拟机的启动**

  Java 虚拟机的启动是通过引导类加载器(bootstrap class loader)创建个初始类(initial class)来完成的，这个类是由虚拟机的具体实现指定的

- **虚拟机的执行**

  - 一个运行中的 Java 虚拟机有着一个清晰的任务：执行Java程序。
  - 程序开始执行时他才运行，程序结束时他就停止。
  - **执行一个所谓的 Java 程序的时候，真真正正在执行的是一个叫做 Java 虚拟机的进程。**

- **虚拟机的退出**，有如下的几种情况：
  - 程序正常执行结束
  - 程序在执行过程中遇到了异常或错误而异常终止
  - 由于操作系统出现错误而导致 Java 虚拟机进程终止
  - 某线程调用 Runtime 类或 system 类的 exit 方法，或 Runtime 类的 halt 方法，并且 Java 安全管理器也允许这次 exit 或 halt 操作。
  - 除此之外，JNI ( Java Native Interface)规范描述了用 JNI Invocation API 来加载或卸载 Java 虚拟机时，Java 虚拟机的退出情况。



