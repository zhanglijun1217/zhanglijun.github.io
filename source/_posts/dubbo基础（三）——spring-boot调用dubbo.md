---
title: dubbo基础（三）——spring boot调用dubbo
copyright: true
date: 2019-03-20 22:11:35
tags:
	- dubbo基础
	- spring boot
categories:
	- dubbo
---

## dubbo集成spring boot

spring boot肯定是现在用的做多的开发框架，而dubbo框架是最流行的rpc框架之一，整合springboot和dubbo的使用很有必要。本篇博客还是根据上一篇中的dubbo简单demo的简单示例来整合spring boot。(上一篇传送门：[dubbo-demo](https://zhanglijun1217.github.io/blog/2019/03/18/dubbo%E5%9F%BA%E7%A1%80%EF%BC%88%E4%BA%8C%EF%BC%89%E2%80%94%E2%80%94%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E8%B0%83%E7%94%A8demo/))

<!-- more -->

### 依赖

因为是springboot项目，dubbo官方也提供了dubbo的starter。

```java
<!-- dubbo starter -->
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.2.0</version>
</dependency>
```

这里注意下spring boot的版本和dubbo-starter的版本映射关系，这里使用的是spring boot 2.1.3版本，用的是0.2版本的dubbo依赖，而1.x版本的spring boot框架则对应0.1.x版本的dubbo starter。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/dubbo-starter%E7%89%88%E6%9C%AC.png)

这里引入了starter之后也引入了之前在上一篇中的zk客户端依赖。

这里也遇到了idea新建maven module之后的一些坑，一直没办法加载对应的类，这里提示下可以尝试查看idea的maven配置，是不是把新加入的module勾选了ignore：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/maven-ignore.png)

### boot-provider

这里也是去先构造对应的服务提供者，提供一个用户地址的简单查询服务。但是spring boot多采用注解驱动和避免了很多繁琐的xml配置，所以这里我们去将dubbo的全局配置配置在application.properties文件中，而关于服务的暴露也是用注解暴露。

#### dubbo应用配置

```java
# 应用方信息
dubbo.application.name=boot-dubbo-demo
# 注册中心地址
dubbo.registry.address=zookeeper://127.0.0.1:2181
# 协议名称
dubbo.protocol.name=dubbo
# 协议端口
dubbo.protocol.port=20800
```

可以看到这里其实就是对应的之前普通spring项目中使用dubbo的provider.xml的标签配置。

**这里要注意是在启动类上要加入@EnableDubbo注解开启spring boot对dubbo的支持。**

#### 服务的暴露

这里是用的@Service注解暴露服务，其实也是对应着dubbo-provider.xml中的dubbo:service标签，这里要注意是不要引入是spring的@Service注解。可以看到这个service注解中也有dubbo:service中对应的属性，比如这里写入的version版本信息。

```java
import com.alibaba.dubbo.config.annotation.Service;
import javabean.UserAddress;
import org.springframework.stereotype.Component;
import service.user.UserService;

import java.util.ArrayList;
import java.util.List;

/**
 * @author 夸克
 * @date 2019/3/18 00:15
 */
@Component
@Service(version = "boot-1.0.0")
public class UserServiceImpl implements UserService {

    @Override
    public List<UserAddress> getUserAddressList(String userId) {

        System.out.println(Thread.currentThread().getName() + " 调用到了消费者");
        final UserAddress userAddress1 = new UserAddress()
                .setUserId(1L)
                .setAddressId(1L)
                .setAddressNo("123")
                .setAddressStr("庆丰大街")
                .setUserName("小张");

        final UserAddress userAddress2  = new UserAddress()
                .setUserId(1L)
                .setAddressId(2L)
                .setAddressNo("456")
                .setAddressStr("西湖")
                .setUserName("小王");

        return new ArrayList<UserAddress>(){{
            add(userAddress1);
            add(userAddress2);
        }};
    }
}
```

启动provider项目，就可以在dubbo-admin上看到注册到注册中心的服务。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/boot-provider.png)

### boot-consumer

消费者端要配置大体和服务提供者端是一样的，也是用@Refernce注解来代替对应的dubbo:refernce标签。这里也要在启动类上去加入@EnableDubbo注解。

#### dubbo的配置

```java
# 应用方信息
dubbo.application.name=boot-dubbo-demo
# 注册中心地址
dubbo.registry.address=zookeeper://127.0.0.1:2181

# 启动端口
server.port=8081
```

这里的端口是8081是因为provider和consumer是两个spring bootmodule 都是启动类去启动的，这里测试在一台电脑上要是不同的端口。

#### 引用暴露的服务

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Reference(version = "boot-1.0.0", timeout = 50000)
    private UserService userService;

    /**
     * 生成订单过程：
     *  调用远程接口 查询用户信息
     *  将用户信息去生成订单
     * @return
     */
    @Override
    public List<UserAddress> initOrder() {
        List<UserAddress> userAddressList = userService.getUserAddressList("1");
        if (null != userAddressList && userAddressList.size() > 0) {
            System.out.println("调用远程接口完成");

            Optional.of(userAddressList).ifPresent(System.out::println);
        }

        return userAddressList;
    }
}
```

可以看到@Reference注解中也可对应dubbo:reference标签的属性，这里设置的超时时间和对应的版本。

#### 简单controller测试

这里去写了一个简单的controller去测试spring-boot使用dubbo这个框架：

```java
@RestController
public class TestController {

    @Resource
    private OrderService orderService;

    @GetMapping(path = "/initOrder")
    public List<UserAddress> initOrder(@RequestParam("userId") Integer userId) {
        return orderService.initOrder();
    }
}
```

## github地址

https://github.com/zhanglijun1217/dubbo-demo