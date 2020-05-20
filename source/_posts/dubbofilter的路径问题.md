---
title: dubbofilter的路径问题
copyright: true
date: 2020-05-20 00:17:24
tags:
	- dubbo filter
categories:
	- dubbo
---

## 自定义dubbofilter

在使用dubbo框架的时候可以使用filter去实现一些拦截功能和调整拦截顺序，在每次调用的过程中，Filter的拦截都会被执行。当然除了Dubbo默认的filter，用户也可以自定义dubbo filter来实现对应的功能。这里记录一个遇到的spi文件路径问题。



## 问题现象

在测试自定义一个dubbo filter之后，发现并没有生效。

对应的filter代码：

```java
@Activate(group = Constants.PROVIDER, order = Integer.MIN_VALUE)
public class HelloFilter implements Filter {

    /**
     * @param invoker
     * @param invocation
     * @return
     * @throws RpcException
     */
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        System.out.println("filter调用了");
        return invoker.invoke(invocation);
    }
}

```

对应的spi文件内容：

```
dubboLoggerFilter=com.xxx.demo.xxx.filter.HelloFilter
```

经过排查之后发现代码写的没啥问题，但dubbo并没有加载这个自定义filter的spi。隐约感觉是路径问题。

## 问题解决

因为是对应一个已生效的filter去设置的，所以当时看到已有项目中的文件是这样的：

![image-20200520204641035](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200520204641035.png)

所以在建立spi文件的文件目录时直接new了一个目录名字叫做META-INF.dubbo。但其实看官方blog中的介绍是要在maven资源文件下建立如下的结果的spi文件：

![image-20200520204834979](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200520204834979.png)

这里恍然大悟才发现自己的目录路径建立错误了。

这里其实是idea展示的一个坑，比如我在现在改对的基础上去建立META-INF.dubbo目录，其实是和正确目录展示是一样的：

![image-20200520205113150](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200520205113150.png)

但是在你提交git文件的时候，其实是能明显看到对应的差别的：

![image-20200520205152514](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20200520205152514.png)

这里记录下踩到的这个坑。

