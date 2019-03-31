---
title: dubbo基础（五）——dubbo接口的特性设置
copyright: true
date: 2019-03-31 16:53:39
tags:
	- dubbo接口属性
categories:
	- dubbo
---

## dubbo的一些配置

之前的文章中写了dubbo的初步使用和dubbo和springboot的使用整合，这里来总结下dubbo框架暴露接口常用的配置项。

<!-- more -->

### 启动时检查

dubbo提供了在服务启动时的一些检查机制，这个机制包括consumer端对服务提供者的检查、dubbo对注册中心的检查。可以看下官方文档中dubbo:reference标签中关于check属性的配置。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/check%E8%AF%B4%E6%98%8E.png)

可以看到是默认在启动时检查服务提供者是否存在，不可用时会抛出异常，组织spring初始化。

我们这里可以将之前demo中的userService服务的消费者上配置check=true(默认就是true)，来看看在不启动服务提供者时的消费者控制台的报错：
![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/check%E9%94%99%E8%AF%AF.png)

是经典的No Provider错误。这里官方文档也对这个配置做了说明：

如果在配置文件中设置了`dubbo.reference.check=false`，则是强制改变所有reference的check值，就算配置中部分接口有声明，也会被覆盖。

同时，也可以在配置文件中设置`dubbo.consumer.check=false`，这个是设置所有的reference的check属性**默认值**，当配置文件中有对单个reference的check的显式设置，会被覆盖掉 

这里也可以去看下对注册中心不存在时的错误：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/register%E8%BF%9E%E4%B8%8D%E4%B8%8A.png)

如果我们在reference标签和register标签中显示配置check="false"，这些错误只有在调用时才会报错。

### 超时

在dubbo的service和reference标签中都有timeout设置，service标签中timeout是1000ms，代表远程服务调用时间；而reference标签中的timeout设置默认值是继承自dubbo:consumer标签中的默认值，也是1000ms。在前几篇dubbo配置中有提到，timeout的配置遵守一个规则：

- 更精准的优先。方法级别配置会覆盖接口级别的配置，接口级别的配置会覆盖全局的provider、consumer配置。
- 如果精准级别相同，消费方优先。当配置的颗粒度是相同的，如果消费方设置了超时属性，会覆盖服务提供方超时属性的配置。

比如这里设置服务提供者的超时是5000ms：

```java
<dubbo:service interface="service.user.UserService" ref="userService" timeout="5000" 
               version="0.0.1" stub="service.user.UserServiceStub">
</dubbo:service>
```

而在消费者配置超时是2000ms：

```java
<dubbo:reference id="userService" interface="service.user.UserService" version="0.0.1" timeout="2000"/>
```

在服务提供者加入一个3000ms的sleep，这里按照配置的优先级会是消费者的2000ms生效。这里会报dubbo接口调用超时的错误：(错误信息中接口调用花费了2004ms，超时生效的是2000ms配置，报错是waiting server-side response timeout)

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E6%8E%A5%E5%8F%A3%E8%B6%85%E6%97%B6.png)

### 重试次数

在dubbo:service和dubbo:reference标签中可以设置retries属性来设置接口调用失败时的重试次数，这里重试此时指的是第一次调用之后的重试次数，并且这里的重试会遵守负载均衡的策略。

service标签中的重试次数默认是2，而reference标签的默认重试次数继承consumer标签的默认次数也是2。这里去简单模拟下有三个服务提给者，调用失败的场景是依靠上边的超时设置，重试次数也尊属消费者优先的规则。

在不同的端口暴露userService服务，每个服务会打印第几个服务，并且有对应的睡眠时间模拟调用：

1. 端口是20880，睡眠时间是5000ms

```java
<dubbo:protocol name="dubbo" port="20880" />

<dubbo:service interface="service.user.UserService" ref="userService"
               version="0.0.1" stub="service.user.UserServiceStub">
</dubbo:service>
```

2. 端口是20881，睡眠时间是5000ms

```java
<dubbo:protocol name="dubbo" port="20881" />

<dubbo:service interface="service.user.UserService" ref="userService"
               version="0.0.1" stub="service.user.UserServiceStub">
</dubbo:service>
```

3. 端口是20882，睡眠时间是2000ms

```java
<dubbo:protocol name="dubbo" port="20882" />

<dubbo:service interface="service.user.UserService" ref="userService"
               version="0.0.1" stub="service.user.UserServiceStub">
</dubbo:service>
```

而在消费者端配置超时时间是3000ms，所以只有端口是20882提供的服务可以不会因为超时调用失败。

消费端的配置：

```java
<dubbo:reference id="userService" interface="service.user.UserService" version="0.0.1" timeout="3000"/>
```

服务启动之后，发起调用可以看到consumer端的日志：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E9%87%8D%E8%AF%95%E6%AC%A1%E6%95%B0.png)

对20880端口暴露的服务调用超时之后，因为默认的负载均衡策略是加权随机调用，这里重试调用了第三个服务接口调用成功。

这里要知道重试应该配置在幂等的接口上，比如查询、更新等，因为会进行重试进行请求。

### 多版本功能

同一个服务要升级时可能出现服务不稳定的情况，可以使用版本号进行过度，不同的版本号之间是隔离的。

在service标签上配置的version属性就是服务提供者的版本；而reference标签上的version则代表引用服务的版本。

我们也在上边使用了0.0.1版本，而和spring boot整合的dubbo接口版本是boot-1.0.0。

如果配置version="*" 如下，则表示的是随机引用版本。

`<dubbo:reference id="barService" interface="com.foo.BarService" version="*" />`

### 负载均衡

dubbo框架对集群环境下提供了多种负载均衡机制：

1. **基于权重的随机负载均衡 Random LoadBalance (dubbo的默认的负载均衡策略)。**
   ![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E9%9A%8F%E6%9C%BA%E7%AD%96%E7%95%A5)

2. **基于权重的轮询负载均衡机制**。

   ![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E8%BD%AE%E8%AF%A2)

3. **最小活跃数负载均衡机制。**
   ![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E6%9C%80%E5%B0%8F%E6%B4%BB%E8%B7%83%E6%95%B0)

4. **一致性hash负载均衡(注意这里是一致性hash，因为集群中的服务节点可以增加、减少)。**

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E4%B8%80%E8%87%B4%E6%80%A7hash)