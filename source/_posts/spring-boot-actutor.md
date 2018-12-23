---
title: spring boot actuator
copyright: true
date: 2018-08-25 19:02:32
tags:
    - spring boot
    - actuator
categories:
    - spring
    - spring boot
---
## 前言
spring boot的一大特性就是自带的actuator。它是spring-boot框架提供的对应系统的自省和监控的集成功能，可以对系统进行配置查看、相关功能统计等。

## actuator的使用
### 引入依赖
```
<dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 配置文件的配置
- management.port:指定访问监控防范的端口，这个端口应该与逻辑端口分离。如果不想使actuator暴露在http中，可以设置这个端口为7002。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203003653424.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)
- management.address：指定地址，比如只能通过本机监控，可以设置 management.address = 127.0.0.1
启动项目，可以看到actuator启动在了配置的7002端口，并且提供了可以访问其中的一些endPoints。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203003701603.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

### 一些主要的EndPoints
spring-boot提供了一些常用的EndPoints

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203003710947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

其中鉴权为true的表示访问这些endPoints是需要保护的不能随意进行访问的。如果要取消，可以设置关闭鉴权（低版本的spring-boot没有提供鉴权）

```
management.security.enable=false
```

## 官方文档
可以看到这个监控和自省的功能是十分有用的，可以看到bean信息、dump信息、mapping信息和访问链路信息等，所以这个功能在官方文档中也说的很清楚，我们也可以通过实现HealthIndicator接口，编写自己的health接口，也可以增加自己的监控接口。
具体的还可以看一下官方文档 [acautor文档](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready)