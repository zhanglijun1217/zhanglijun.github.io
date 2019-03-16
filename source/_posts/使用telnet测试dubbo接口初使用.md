---
title: 使用telnet测试dubbo接口初使用
copyright: true
date: 2019-02-25 20:27:13
tags:
	- telent
categories:
	- dubbo
---

## 背景

dubbo接口的测试不像controller的http接口那么容易测试，这里去了解了下使用telnet去测试参数没那么复杂的dubbo接口。

<!-- more -->

## 正题

首先看看一个dubbo接口的代码：

```java
public interface ShopAggregateRemoteService {
    /**
     * 获取所有产品类型 聚合字段的产品类型范围[软件+有伴服务]
     * @param careDeleteFlag 是否关心软删 true：只返回生效的 false：全部返回
     * @return
     */
    ListResult<AggregateProductTypeDTO> getAllAggregateProductType(boolean careDeleteFlag);

    /**
     * 获取所有产品类型 聚合字段的产品类型范围[软件+有伴服务]
     * @return
     */
    default ListResult<AggregateProductTypeDTO> getAllAggregateProductType() {
          return getAllAggregateProductType(false);
    }
}
```

 里面的实现：(就是一个简单的查数据库操作)

```java
@Override
public ListResult<AggregateProductTypeDTO> getAllAggregateProductType(boolean careDeleteFlag) {
    AggregateProductTypeCriteria example = new AggregateProductTypeCriteria();

    AggregateProductTypeCriteria.Criteria criteria = example.createCriteria();
    if (careDeleteFlag) {
        // 关心软删 加入软删标志
        criteria.andDeleteFlagEqualTo(0);
    }
    return DTOWrapper.wrap(customerConverter.convertList(aggregateProductTypeMapper.selectByCondition(example),
            AggregateProductTypeDTO.class));
}
```

这里想本地启动项目去测试一下这个接口有没有问题，这个参数也比较简单（这里直接调用default方法就可以了）。

这里使用的mac系统，但是mac系统中高版本未默认安装telnet，所以还需要brew去安装一下。

`brew install telnet`

安装完成之后，就可以在本地将服务启动。

输入命令去telnet这个服务的dubbo接口：

`telnet localhost 7100`

然后可以看到进入到了dubbo的命令行界面：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190225-235536%402x.png)

这里可以用dubbo的`ls`命令去查看有什么dubbo服务：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190225-235846%402x.png)

最后是我们的invoke调用这个命令

`invoke com.youzan.xxx.getAllAggregateProductType()`

可以看到结果是调通的：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190226-000252%402x.png)

## 总结

这里使用的dubbo的简单命令去测试了工作中的一个接口，之后会对dubbo的使用和测试做更多的分析。