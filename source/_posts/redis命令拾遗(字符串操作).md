---
title: redis命令拾遗(字符串操作)
copyright: true
date: 2019-05-19 01:09:46
tags:
	- redis命令
categories:		
	- redis
---

## 前言

前一段时间一直在忙，拉下了一些知识的学习，现在努力追赶修补中。= =

当然也有一些新的知识的学习，但其实更多的是关于一些知识的拾遗。之前在工作当中发现对redis命令掌握的还不是很完善，所以想花比较少的碎片时间去写一下redis常用命令的拾遗。

## redis命令

对这些命令的拾遗记录是在网站：[http://redisdoc.com](http://redisdoc.com/)上进行学习的，很简单明了，推荐给大家进行学习拾遗。

这里只是把日常会被忽略或者遗忘的点进行一下梳理，并不是每个知识点的一个总结。

### 字符串操作

####set命令

set可以通过一系列参数进行修改：

- ex seconds ：将键的过期时间设置为seconds秒，具体的命令是set key value EX seconds。等同于执行setex key second value。
- px milliseconds：和ex一样，只不过单位是毫秒，具体的命令是 set key value PX milliseconds。等同于执行psetex key milliseconds value。
- nx / xx： set key value nx等价于 setnx key value；set key value xx是当键存在才设置值，没有setxx这个吗命令，这两个设置值失败的时候，set命令会返回nil，而直接使用setnx命令，则返回的是0和1。

#### setex命令

setex命令效果等价于执行下边两个命令：

```c
set key value
expire key seconds
```

但是不同的是，setex是一个原子的操作，它是在同一时间完成设置值和过期时间的操作，经常用在存储缓存时候。

setex设置成功时候 返回ok。

同样psetex只是单位是毫秒而已。

#### get命令

get命令不用多说，但是注意get命令只是用在字符串操作，如果key对应的值不是字符串类型，那么返回一个错误。

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190519-020316.png)



#### getset命令

此命令的作用是：将key设置为value，并且返回key在被设置之前的值。如果key之前不存在，则返回nil。当键key存在但不是字符串时，会报错。

#### strlen命令

返回字符串key的长度，当key不是字符串时，返回一个错误。如果key不存在，返回0。

#### append命令

append命令：如果已经存在key并且它的值是一个字符串，append命令将value追加到key对应值的末尾。如果key不存在，append命令会像执行set key value一样将值设置为对应的key的值。

append命令的返回值是值字符串的长度。

注意append的时间复杂度是平摊o(1)

#### setrange key offset value

指从偏移量offset开始，用value参数覆写value值。这个命令会确保字符串足够长以便于设置value到对应的偏移量。比如字符串只有5个字符长，但设置的offset是10，那么会在原来字符串值到偏移量之间设置零字节("\x00")进行填充。

这个命令的返回值是被修改之后字符串值的长度

#### getrange key start end

这个命令指的是返回键key对应的字符串值的指定部分，字符串的截取范围由start end两个参数决定(包括start和end在内)。start和end支持负数偏移量，-1代表最后一个字符，-2代表倒数第二个字符。但是注意只能按照字符串顺序获取，不能倒序获取**(比如 getrange key -1 -3)**

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/getrange%E5%91%BD%E4%BB%A4.png)

#### incr key

incr虽然是自增的含义命令，但其实是一个属于字符串的操作，redis并未提供一个专用的整数类型，所以键key存储的值在执行incr命令的时候会被翻译解释为十进制64位有符号整数。

如果incr操作的key值对应不存在，那么先会初始化为0，然后再执行incr命令。

如果key值不能被解释为数字，那么会返回一个错误。

#### incrby key increment

和incr一样的含义，只不过有递增量为increment。同样的递减是有对应的decr key和decrby key decrement。

#### incrbyfloat key increment

这个就是针对浮点数的增加计算。注意incrbyfloat命令计算的结果最多只保留小数点后面17位。

#### mset key value [key value ...]

同时为多个键设置值，这个命令是一个原子操作，所有给定键会在同一时间内被设置，并且具有set的特性，会覆盖key对应原来的值。如果仅是在不存在的情况下设置值，可以用msetnx，msetnx也是一个原子操作，如果多个key中有一个key没有设置上，那么所有的key都不会设置对应的值。