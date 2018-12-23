---
title: orika转换bean的字段丢失
copyright: true
date: 2018-08-02 10:33:08
tags:
    - orika
    - bug
categories:
    - bug记录
---

## 背景
- 用orika对象转换工具去转换list的时候，发现只去完整转了list的第一条数据，但是后边的数据都没有将字段全部映射上去。
<!-- more -->
- 描述：
     1.debug时发现的，源数据list是数据都存在的
![源数据](http://pcsb3jzo3.bkt.clouddn.com/0B07F97B-A2AD-428A-80B0-C11E440C9704.png)

    2. 转完之后的list数据，发现userName、realName等字段是丢失的。
    ![转换之后的数据](http://pcsb3jzo3.bkt.clouddn.com/C953E1D8-E10B-4248-B067-4CD5CC819811.png)


## 解决
经过排查发现是因为在转换注册的字段中，有个type字段没有对应的注册上去。这里就造成了orika这个转换工具丢失了list中记录字段的数据转换。
![注册字段](http://pcsb3jzo3.bkt.clouddn.com/429B7186-4B0E-477A-BC63-55057A713FC7.png)

这里去记录一下这次碰到的小bug，其实也是粗心导致的。
