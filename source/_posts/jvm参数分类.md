---
title: JVM参数分类
copyright: true
tags:
	- JVM参数分类
categories:
	- Java基础
	- JVM
---

## JVM参数的分类

常用的JVM参数可以大致的分为三类，下边简单的将JVM的参数做一个分类，作为一个JVM参数的简单总结。

<!--more-->

### JVM标准参数

JVM的标准参数是指的在各个JDK版本中都比较稳定的，不会变动的参数，一般是针对jdk全局的参数。

比如

- -help 
- -server -client
- -version -showversion
- -cp -classpath

这种参数称之为标准参数。

### JVM非标准化参数

JVM非标准化参数指的是在各个jdk版本中会有略微变化的参数，这里可以分为 -X参数 和 -XX参数

#### -X参数

比如-X参数有：

- -Xint : 解释执行
- -Xcomp ：第一次使用就编译成本地代码（java不是严格的解释性或者编译性执行的语言）
- -Xmixed :  上面两种混合模式，有编译器自己去优化选择。

可以看到当前电脑上jdk的version中可以显示这个参数的值：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mix-mode.png)

这里是mix模式，我们可以通过命令`java -Xint -version`来将mode调整为int模式：
![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/interrupt-mode.png)

#### -XX参数

-XX参数使我们平常使用最多的JVM参数，主要用于调优和Debug。

常见的-XX参数有

1. Boolean类型

格式：`-XX:[+/-]<name>`  表示启动或者禁用了name参数

比如：`-XX:+UseConcMarkSweepGC `表示启用了CMS垃圾回收器

​	    `-XX:+UseG1GC` 表示启用了G1垃圾回收器

2. 非Boolean(Key-Value类型)

格式：`-XX:<name>=<value>` 表示name参数的值是value

比如：`-XX:MaxGCPauseMills=500`表示GC最大的停顿时间是500

​	    `-XX:GCTimeRatio=19`



-XX参数常见的参数还有一种缩写模式（虽然是-X开头，但是其实是-XX参数）

比如：

- -Xms  等价于`-XX:InitialHeapSize`  表示初始化堆大小
- -Xmx  等价于`-XX:MaxHeapSize`  表示最大堆的大小
- -Xss  等价于 `-XX:ThreadStackSize`  线程堆栈的大小

可以使用 jinfo -falg <参数名称> 命令行工具去查看当前java进程的JVM参数。

比如：

`jinfo -flag MaxHeapSize pid` 

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/maxHeapSize.png)

之后会介绍命令行工具去查看对应的JVM参数。