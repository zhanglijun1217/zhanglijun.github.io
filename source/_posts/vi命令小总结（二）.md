---
title: vi命令小总结（二）
copyright: true
date: 2018-09-30 16:46:17
tags:		
	- vi命令
categories:	
	- linux
---

## 在vi编辑模式下显示行数

在vi编辑模式下可以显示下行数，比如在php调试模式下可以根据相应的行数的代码去打印值调试代码。

方法：在vi模式下输入:set nu即可。也可以直接:line number跳转到对应的行数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120300250641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

## 在vi编辑模式中撤回一个操作

在INSERT模式下如果写了一些操作，然后想撤回这个操作，按一下esc之后输入u即可。如果是想撤回刚才那个撤回操作，可以按了esc之后点击Ctrl + r。