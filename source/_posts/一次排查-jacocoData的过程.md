---
title: 一次排查$jacocoData的过程
copyright: true
date: 2019-03-10 21:58:02
tags:
	- 反射
	- 字节码
categories:
	- bug记录
---

## 起因

最近在开发过程中，遇到了一个奇怪的现象，在测试环境去利用反射拿一个类的字段时，发现拿到的field数组中多了一个奇怪的变量：$jacocoData，是一个static的boolean数组：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190310-222010%402x.png)

<!-- more -->

很明显jacoco这种统计代码覆盖率不是我定义在一个业务含义的类中，这时考虑到可能是测试环境中对代码覆盖率在编译时对字节码进行了修改，于是去测试环境的机器上看这个jar包。

## 疑惑点

在机器上对jar包解压（解压命令jar -xvf xxxxxxx.jar），并且在对应的目录下找到对应的calss文件，注意这里解压之后的要看的字节码文件都在BOOT-INF目录的lib下：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190310-223514%402x.png)

将反编译的字节码复制了下来，却发现对应的字节码中并没有这个变量。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190310-224134%402x.png)

这里就不是网上说的很多编译时修改字节码来实现测试覆盖率的功能。

## 解决

后来问了部署jacoco服务的框架组人员，发现是用了java Agent在修改运行时字节码实现的，拉了dump发现确实在运行时进行字节码修改的，而在jacoco官方的github也曾经有过这个问题的issue：

[](https://stackoverflow.com/questions/16981831/invalid-column-name-jacocodata-using-ant-jacoco-and-junit)

关于java Agent技术可以参考博客:[](https://www.cnblogs.com/aspirant/p/8796974.html)，这里提到了asm技术和agent探针参数。

所以在反射取字段时候遇到这个坑比较难排查，记录一下。这里的解决办法参考了博客:[](https://www.cnblogs.com/aspirant/p/8796974.html)，即使用了是否为复合字段的field方法解决。