---
title: vi命令小总结
copyright: true
date: 2018-09-09 22:04:52
tags:
	- vi命令
categories:
	- linux
---

## 前言

这篇去写一些最近在工作中get到的关于vi/vim命令的点，简单去记录下。

## 技能点

### 在文件中快速删除一行

在vi编辑文件的时候，发现有时要删除很多的文件内容，这个时候去一点点删除很慢，这里get到了一个快速删除一行的技能。在打开的文件中所要删除的行连续按两次d就可以快速删除一行，然后在用:wq保存即可。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203005303936.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

### 解决没有正确关闭vi打开的文件

有的时候我们在查看一个文件之后，直接ctrl+z去退出了文件，当我们再用vim命令去编辑文件的时候，这时候会发现报一个没有正确关闭这个文件的冲突错误，并且你不去解决就会一直存在。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203005248626.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

这里get到了去解决的方法，其实也是怪自己一看到一堆英文就不想看下去，这里其实已经写得很清楚了，这里提供了两种情况：

（1）有可能另一个人也在编辑这个文件，它提醒你要注意两个人同时编辑不同的地方。

（2）上一次编辑的session还在，这时候提供了解决办法：可以用recover或者vim -r 文件名去修复这个changes。

可以看到当你没正确退出时，还保持着edit session，这时候会生成两个临时文件，我这里采用的是直接删除这两个文件即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203005235670.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

运行 rm -rf .fileName.* 命令之后，再次打开之后就不会有冲突。 