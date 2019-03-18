---
title: dubbo基础（二）——一个简单调用demo
copyright: true
date: 2019-03-18 22:51:21
tags:
	- dubbo基础
categories:
	- dubbo
---

## get start

在上一篇中介绍了dubbo诞生的背景和框架的特性：[dubbo概念和基本概念](https://zhanglijun1217.github.io/blog/2019/03/06/dubbo%E5%9F%BA%E7%A1%80%EF%BC%88%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94%E6%A6%82%E5%BF%B5%E5%8F%8A%E5%9F%BA%E6%9C%AC%E6%A1%86%E6%9E%B6/)，这里就来一个dubbo的简单使用小体验。

### dubbo注册中心安装

dubbo中的官方文档的快速启动使用的是multicast广播注册中心暴露服务地址，这里选择的是使用zk作为注册中心，因为zk是很多公司作为dubbo注册中心，并且zk也是dubbo官方文档中推荐使用的注册中心。

本次是使用的mac上安装的zk，安装步骤：[mac下安装zk](https://www.jianshu.com/p/5491d16e6abd)

### dubbo-demo建立

此次是采用maven构建整个项目。整个项目中的module结构如下：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190318-230924.png)

其中：

- common-interface是抽离出的公用接口，其实这里就是将provider中的接口和consumer中的接口抽离出来
- order-service-consumer是此次设置的dubbo-consumer，模拟的是一个订单服务，里面有一个初始化订单的方法。
- user-service-provider是此次设置的dubbo服务提供者，模拟的是一个用户服务，在初始化订单时肯定要查询用户服务的接口。

#### maven依赖

此次在项目中的依赖用到了dubbo的依赖，因为使用的是zk注册中心，所以这里要引入zk的客户端依赖，这里要注意的是**dubbo2.6版本之后是要引入zk的curator客户端**。

dubbo依赖：

```java
<!-- https://mvnrepository.com/artifact/com.alibaba/dubbo  这个dubbo版本是集成了spring的 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.6.2</version>
</dependency>
```

zk client依赖

```java
<!-- curator-framework -->
<!-- dubbo2.6 以上版本要引入curator 的 zk客户端 -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.12.0</version>
</dependency>
```

#### provider

由dubbo的架构图可知

![dubbo整体结构](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/dubbo-architecture.jpg)

provider提供的服务要先注册在注册中心上，这里就要去配置provider相关的配置。

可以看到provider要提供的接口服务：

```java
public interface UserService {

    List<UserAddress> getUserAddressList(String userId);
}
```

这里可以给一个简单实现：

```java
public class UserServiceImpl implements UserService {

    @Override
    public List<UserAddress> getUserAddressList(String userId) {

        System.out.println("调用到了消费者");
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

这里的关键是provider的xml配置：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
         http://dubbo.apache.org/schema/dubbo
         http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="dubbo-demo"  />

    <!-- 注册的地址 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181" />

    <!-- 使用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />

    <!-- 生命要暴露的服务接口 -->
    <dubbo:service interface="service.user.UserService" ref="userService" />

    <!-- 实现服务 -->
    <bean id="userService" class="service.impl.UserServiceImpl" />

</beans>
```

其中的配置：

- application标签，提供了服务提供方应用信息，用于计算依赖关系和dubbo-admin中的界面信息
- registry标签，这个标签标明了注册中心和注册中心的地址，这里可以支持多种协议和多种写法，具体见官方文档。
- protocol标签，这里是暴露在20880端口的服务，这里的端口可以修改。
- service标签，这里是声明要暴露的用户服务接口，ref为spring环境中的bean的id。
- bean标签，这里是spring的操作，将实现作为spring bean管理。

这里可以简单的测试一下服务是否注册成功：

```java
public class TestProvider {

    public static void main(String[] args) throws Exception {
        // 加载spring配置
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("provider.xml");
        // 启动spring环境
        applicationContext.start();

        // 使程序hang住  不退出
        System.in.read();
    }
}
```

启动这个测试程序后可以用telnet测试一下服务是否在本地的zk上注册上去

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/dubbo%E6%9C%AC%E5%9C%B0telnet.png)

zk也可以看到多了一个dubbo节点：
![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/zk-dubbo.png)



#### consumer

同样consumer也要先实现order服务的接口

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Resource
    private UserService userService;

    /**
     * 生成订单过程：
     *  调用远程接口 查询用户信息
     *  将用户信息去生成订单
     * @return
     */
    @Override
    public boolean initOrder() {
        List<UserAddress> userAddressList = userService.getUserAddressList("1");
        if (null != userAddressList && userAddressList.size() > 0) {
            System.out.println("调用远程接口完成");

            Optional.of(userAddressList).ifPresent(System.out::println);
        }


        return true;
    }
}
```

配置对应的consumer.xml文件

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
         http://dubbo.apache.org/schema/dubbo
         http://dubbo.apache.org/schema/dubbo/dubbo.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <dubbo:application name="dubbo-demo" />

    <dubbo:registry address="zookeeper://127.0.0.1:2181" />

    <!-- 引用远程接口 -->
    <dubbo:reference id="userService" interface="service.user.UserService" />

    <!-- 启动包扫描 -->
    <context:component-scan base-package="service.impl" />

</beans>
```

这里多了一个dubbo的reference标签，这里就是要引用刚才暴露的接口。

在实现里看到了将实现类作为spring bean管理，这里在xml中开启了包扫描。

这里去写一个简单的test程序去测试写的consumer程序：

```java
public class TestConsumer {

    @SneakyThrows
    public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("consumer.xml");

        // 启动spring环境
        applicationContext.start();

        // 拿到orderService这个bean
        OrderService bean = applicationContext.getBean(OrderService.class);

        // 调用 测试是否调用了远程接口
        bean.initOrder();

        System.in.read();
    }
}
```

同时这里也启动这个test类，看控制台中的输出观察是否调用了远程接口。

可以看到在TestConsumer控制台输出

```
调用远程接口完成
[UserAddress(addressId=1, addressNo=123, addressStr=庆丰大街, userName=小张, userId=1), UserAddress(addressId=2, addressNo=456, addressStr=西湖, userName=小王, userId=1)]
```

在TestProvider控制台输出

```
调用到了消费者
```

可以看到调用成功。

## github

此次的代码已上传至github: [github](https://github.com/zhanglijun1217/dubbo-demo)

