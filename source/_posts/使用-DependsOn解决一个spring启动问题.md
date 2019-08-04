---
title: 使用@DependsOn解决一个spring启动问题
copyright: true
date: 2019-08-04 18:18:42
tags:
	- DependsOn注解
	- spring启动
categories:
	- spring
---

## 前言

最近遇到了一个启动失败的问题，原因是在bean初始化完成之后的钩子方法中使用获取容器中bean的工具类，(对应工具类之前的一篇博客 [获取springbean]([https://zhanglijun1217.github.io/blog/2018/08/12/spring%E4%B8%AD%E6%A0%B9%E6%8D%AEApplication%E8%8E%B7%E5%8F%96BEAN%E7%9A%84%E5%B7%A5%E5%85%B7%E7%B1%BB/](https://zhanglijun1217.github.io/blog/2018/08/12/spring中根据Application获取BEAN的工具类/))）。

## 分析

这里具体的场景是我想实现一个bean在钩子方法中往一个策略map中注册自己作为一个策略使用，但是在启动的时候报错:

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/DepnedsOn%E5%90%AF%E5%8A%A8%E6%8A%A5%E9%94%99.png)

第33行代码如下：

```java
 public static <T> T getBean(@NotNull Class<T> tClass) {
        return context.getBean(tClass);
    }
```

可以看到可能为空的是context，这个是通过在项目中启动时注入到ApplicationContextUtil中的静态变量context，很明显是在当前这个bean启动的时候，其钩子方法去调用这个变量还没实现context的注入。

```java
   @Override
    public void afterPropertiesSet() throws Exception {
       // 策略工厂中注册 自身 的代码
    }
```

## 解决

这里主要是一个场景，其实在bean启动的时候是依赖ApplicationContextUtil这个bean的，但是因为getBean方法都static方法，在平常业务代码中调用都是容器启动完毕的时候，所以没有问题，但是这里是想实现在bean初始化时自动通过钩子往一个map工厂中注册bean实例，且该bean没有显示的@Resource依赖ApplicationContextUtil，所以在注册的时候applicationContextutil这个bean还没初始化好，这里在这些具体策略的类上加了@DependsOn("applicationContextUtil")

```java
@Service
@Slf4j
@DependsOn(value = "applicationContextUtil")
public class AStrategy extends AbstractStrategy {
}
```

这表示这个bean的初始化是依赖 applicationContextUtil 这个bean初始化完成之后(也就是静态变量上下文被注入)才去初始化的，这样启动就不会报NPE了。