---
title: orika线程死循环记录
copyright: true
date: 2020-05-24 23:15:08
tags:
	- orika
	- map死循环
categories:
	- bug记录
---

## 问题现象

在测试环境看到机器cpu报警，且cpu是突然升起来并且一直稳定跑满在百分之90左右。观察流量和接口的qps，并没有突然增加或者有突刺。

## 问题排查

上机器top -H -p pid + jstack观察之后发现很多http线程卡在orika的一个weakHashMap的get方法中：

![image-20200524232028707](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200524232028707.png)

![image-20200524232121915](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200524232121915.png)

很明显，这里是触发了经常看到HashMap一类分析文章中的map链表成环并且死循环的问题，然后就去查看了orika中的这个类，代码维护了一个全局的weakHashMap：

```java
// 定义了一个WeakHashMap
	private static volatile WeakHashMap<java.lang.reflect.Type, Integer> knownTypes = new WeakHashMap<java.lang.reflect.Type, Integer>();

```

这里有点不明白的是使用的是java1.8，记得代码中是更改了扩容时候链表的迁移方式，避免了成环操作，然后就去看了WeakHashMap的操作，发现原来WeakHashMap并没有去树化和改变迁移链表的方式，还是可能出现成环，然后在get的时候死循环导致cpu异常。



其实在WeakHashMap的注释中也看到了对应不同步的说法：

```java
 * <p> Like most collection classes, this class is not synchronized.
 * A synchronized <tt>WeakHashMap</tt> may be constructed using the
 * {@link Collections#synchronizedMap Collections.synchronizedMap}
 * method.
```



## 问题解决

这里尝试去找了网上的文章和orika的issue，发现有遇到同样问题的文章和issue:

[参考blog]([https://gsmtoday.github.io/2019/09/03/BeanUtils%E4%B8%ADHashMap%E8%A7%A6%E5%8F%91%E6%AD%BB%E5%BE%AA%E7%8E%AF/](https://gsmtoday.github.io/2019/09/03/BeanUtils中HashMap触发死循环/))

[官方issue](https://github.com/orika-mapper/orika/issues/56)

这里都提示了在高版本解决了这个问题。我这里使用的是orika1.5.1版本的包，升级为最新的1.5.4之后，再看这个weakHashMap加上了同步方法：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200524234147636.png)