---
title: java平台与java是解释执行的正确性
copyright: true
date: 2018-11-30 00:35:23
tags:
	- Java平台
	- 解释执行、编译执行
categories:
	- Java基础
---

## 背景

最近看了一点点极客时间上的《Java核心技术36讲》，打算把一些自己感兴趣或者不知道的点总结到博客中，方便对一些知识有一些整理和拾遗。

<!-- more -->

## Java平台性的理解

java本身是一种面向对象的语言，有两个特征，一是“write once, run anywhere”，能够非常容易的获取跨平台的能力；另外就是垃圾手机机制，Java通过垃圾收集器(Garbage Collector)回收分配内存。

我们会接触到JRE和JDK。其中JRE是Java Runtime Environment，JDK是Java Development Kit。JRE是java运行环境，包含了JVM和java类库，以及一些模块等。而JDK可以看做是JRE一个超集，提供了更多工具，比如编译器、各种诊断工具、安全工具等。

对于Java平台的理解，可以从两个方面去谈一下。第一个方面是Java语言的特性，包括泛型、集合类、java8中的lambda特性、基础类库、IO/NIO、网络、并发、安全等，这个系统化的去总结一下，肯定能对java平台有更深的理解。另一方面可以去理解JVM中的一些概念和机制，比如Java的类加载机制，JDK中的Class-Loader，例如Bootstrap、Application和Extension Class-loader；类加载的过程：加载、验证、链接和初始化；自定义Class-Loader等；还有java的内存模型，堆、栈、方法区等内存在程序运行时的分配；垃圾回收的基本原理，常见的垃圾回收器，如SerialGC、Parallel GC、CMS、G1等，对于适用于什么样的工作负载也要去有深入的理解。

对于java是解释执行的理解，这个说法是不太准确的。我们写的java代码，首先是通过javac编译成为字节码（byteCode），然后在运行时，通过JVM内嵌的解释器将字节码转换成为最终的机器码。但是存在常见的JIT(Just-In-Time)编译器，也就是说的动态（及时）编译器，JIT能够在运行时将热点代码编译成机器码，这种情况部分热点代码就是属于编译执行，而不是解释执行了。这里说的Java中的编译不同于C中的编译，javac的编译，是把Java代码生成.class字节码，而不是机器直接运行的机器码，Java通过字节码和内嵌虚拟机的这种机制，屏蔽了操作系统和硬件的细节，促使了跨平台性。

在运行时，JVM会通过类加载器（Class-loader）加载字节码，解释或者编译运行。比如JDK8版本中，实际上是采用的解释和编译混合的模式，即混合模型（-Xmixed）。在运行在server模式的JVM，会进行上万次调用以收集足够的信息进行高效的编译，而client模式这个门限是1500次。Oracle HotSpot JVM内置了两个不同的JIT compliers，C1对应之前说的Client模式，使用于对于启动速度敏感的应用，比如普通Java应用；C2对应server模式，这种JIT编译优化适用于长时间运行的服务器端应用设计的。默认是采用所谓的分层编译。另外，在JDK9中引入了AOT特性，所谓AOT（Ahead-of-Time Compilation）就是直接将字节码编译成机器码，避免了JIT预热的一些开销，jdk9中也加入了新的jaotc工具，可以运用jaotc命令使某个类或者模块编译成为AOT库。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120300003212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

JVM启动时，可以指定不同的参数对运行模式进行选择。比如指定"-Xint"，就是告诉JVM只进行解释执行，不对代码进行编译，这种模式相当于直接抛弃了JIT带来的性能优化。与其相对应的，还有一个“-Xcomp”参数，这是告诉JVM关闭解释器，不再进行解释执行。那么这种是不是就会很高效呢？其实不然，这种模式会导致JVM的启动变得很慢。

