---
title: dubbo基础（四）——dubbo的配置加载
copyright: true
date: 2019-03-20 23:06:06
tags:
	- dubbo配置
categories:
	- dubbo
---

## dubbo的配置

在之前的文章中配置了spring boot和dubbo框架的使用（传送门：[springboot使用dubbo框架](https://zhanglijun1217.github.io/blog/2019/03/20/dubbo%E5%9F%BA%E7%A1%80%EF%BC%88%E4%B8%89%EF%BC%89%E2%80%94%E2%80%94spring-boot%E8%B0%83%E7%94%A8dubbo/)），看到了把dubbo相关的配置配置在了配置文件中。这里官方文档中也去讲解了对应的dubbo配置的加载。

<!-- more -->

### dubbo的配置加载流程

首先要知道dubbo的配置是在**应用启动阶段**，并且这里的配置包括**应用配置、注册中心配置、服务配置等。**

#### dubbo的配置来源

- Jvm System Properties，-D参数
- Externalized Coniguration， 外部化配置，这里在文档中提到的有zk、apollo
- ServiceConfig、ReferenceConfig等编程接口采集的配置
- 项目本地的配置文件 dubbo.properties

除了外部化配置，dubbo的配置读取在总体上遵循了以下几个原则：

- dubbo支持了多层级的配置，并且按预定优先级自动实现配置间的覆盖，最终所有配置汇总到数据总线URL后，驱动后续的服务暴露、引用等流程。
- ApplicationConfig、ServiceConfig、ReferenceConfig也可以理解为配置来源的一种，是直接面向用户编程的配置采集方式
- 配置格式以Properties为主，在配置上支持path-based的命名规范。

#### dubbo配置的覆盖策略

dubbo的配置覆盖策略如下图：
![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/dubbo%E9%85%8D%E7%BD%AE%E8%A6%86%E7%9B%96%E7%AD%96%E7%95%A5.jpg)

这里看到dubbo预先设置的配置覆盖加载顺序是jvm设置的启动参数 —> 外部配置 —> spring的xml或者api配置 —> 项目中的本地文件。

这里以上一篇中的spring boot和dubbo整合的demo来测试这个覆盖顺序。这里外部的配置先不做演示，只去比较启动参数、spring配置、本地dubbo.properties配置三个覆盖顺序。

我们以dubbo.protocol.port这个配置作为示例：

1. jvm启动参数

首先配置启动时的vm参数：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/vm%E5%8F%82%E6%95%B0.png)

启动之后，可以看到此时服务在dubbo-admin上显示的端口为我们在启动参数中设置的20881。(**配置文件中配置的是20880，这里优先读取的是启动参数中的配置**)

![image-20190321230626086](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/vm%E5%8F%82%E6%95%B0%E7%9A%84%E6%98%BE%E7%A4%BA.png)

2. spring配置

这里因为使用的是spring boot，所以在application.properties中配置即可。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/properties%E9%85%8D%E7%BD%AE.png)

这里启动provider项目，可以看到之前的接口下线，然后暴露服务的端口变为了20880。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190321-231314.png)

3. 本地的dubbo.properties配置

在本地建立一个dubbo.properties文件，这里写成暴露服务的端口是20882。(**这里要主要将spring配置中配置注释掉。**)

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/dubbo.properties.png)

重新启动provider，可以看到对应的dubbo接口暴露为了20882。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/20882.png)

#### 以xml配置说明不同粒度的覆盖和优先级

由下图可以知道这些配置标签分为provider侧、consumer侧、应用共享的配置(这里包括application、注册中心、monitor的配置)、还有一些子配置(方法、参数级别的配置)不同的粒度。

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/config%E4%B9%8B%E9%97%B4%E5%85%B3%E7%B3%BB.jpg)

这些不同的粒度的配置也有对应的覆盖关系，以timeout为例，其他retries，loadbalance，actives等类似，都遵守粒度之间的覆盖规则：

- 方法级别优先，接口级别次之，全局配置再次之。
- 如果级别一样，则消费方优先，提供方次之。

其中，服务提供方配置，通过 URL 经由注册中心传递给消费方。具体如下图所示：

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/dubbo-config-override.jpg)

**这里就timeout这个超时参数，建议由服务提供方设置超时，因为一个方法需要执行多长时间，服务提供方更清楚，如果一个消费方同时引用多个服务，就不需要关心每个服务的超时设置。**

这里的一个小tips：

引用缺省是延迟初始化的，只有引用被注入到其它 Bean，或被 `getBean()` 获取，才会初始化。如果需要饥饿加载，即没有人引用也立即生成动态代理，可以配置：`<dubbo:reference ... init="true" />`