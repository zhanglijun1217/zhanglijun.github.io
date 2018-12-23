---
title: ClassNotFoundException和NoClassDefFoundError
copyright: true
date: 2018-12-06 01:07:01
tags:
	- Java平台
	- 解释执行、编译执行
categories:
	- Java基础
---

## 背景

极客时间上《Java核心技术36讲》第二讲中提到了一个问题：ClassNotFoundException和NoClassDefFoundError有什么区别？看到这个问题的时候，第一时间想到的就是一个是受检的异常，而另一个是一个Error，但是其实在真正的项目开发中这两个错误都遇到过，都是关于类或者文件jar包找不到的错误，这里去总结下其中的不同。

<!-- more -->

## 两者的区别

### ClassNotFoundException

可以看到，ClassNotFound是一个继承自ReflectiveOperationException的受检异常。JDK中的解释是：当应用程序视图使用以下方法通过字符串加载类时，（加载的方法）
（1）Class类中的forName()方法
（2）ClassLoader类中的findSystemClass方法
（3）ClassLoader类中的loadClass方法。
但是没有找到具体指定名称的类的定义。
这里可以知道当应用程序运行的过程中尝试使用类加载器去加载Class文件的时候，如果没有在classpath中查找到执行的类，那么就会抛出ClassNotFoundException。一般情况下，当我们使用Class.forName()或者ClassLoader.loadClass()以及使用ClassLoader.findSystemClass()在运行时加载类的时候，如果类没有找到，那么就会导致JVM抛出ClassNotFoundException。最常见的，我们都写过jdbc连接数据库的代码，我们都会用到Class.forName()去加载JDBC的驱动，如果我们没有将驱动放到应用下的classPath，那么会导致抛出异常ClassNotFoundException。

```java
 public static void main(String[] args) {

        try {

            Class.forName("aaaaaa");

        } catch (ClassNotFoundException e) {
            System.out.println("发生了ClassNotFoundException");
        }
    }
```
还有根据类加载器的可见性机制，子类加载器可以看到父类加载器加载的类，而反之则不行。所以当一个类已经被Application类加载器加载过了，然后如果想要使用Extension类加载器加载这个类，将会抛出java.lang.ClassNotFoundException异常。
### NoClassDefFoundError
看名字这个是一个Error异常，看源码可以看到它是一个继承自LinkageError的Error异常，看官网中的解释是要找的类在编译时期还可以找到，但是在运行java应用的时候找不到了，这比较经常出现在静态块的初始化过程中。JDK官方的解释：
当虚拟机或ClassLoader实例在类的定义中加载（作为通用方法调用的一部分或者作为new 表达式创建的新实例的一部分），但是无法找到该类的定义时，抛出此异常。当前执行的类被编译时，所搜索的类定义存在，但无法再找到该定义。
从官网的解释中我们可以看到如果编译了一个类B，在类A中调用，编译完成之后，你又删除B的class文件，运行A的时候那么就会出现这个错误。

## 总结
这里总结一下ClassNotFoundException和NoClassDefFoundError的区别：
（1）一个是Exception，受检的异常；而一个是Error。
（2）ClassNotFoundException是在动态加载Class的时候调用Class.forName等方法抛出的异常；而NoClassDefFoundError是在编译通过后执行过程中Class找不到导出的错误。
（3）ClassNotFoundException是发生在加载内存阶段，加载时从classpath中找不到需要的class就会出现ClassNotFoundException，出现这种错误可能是调用上述的三个方法加载类，也有可能是已经被一个类加载器加载过的类又被另一个类加载器加载；而NoClassDefFoundError是链接阶段从内存找不到需要的class才会出现，比如maven项目有的时候打包问题会引起这个error报错，这个时候要把仓库中的包删掉重新去拉一下包。