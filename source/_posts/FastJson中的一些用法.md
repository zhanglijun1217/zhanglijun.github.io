---
title: FastJson中的一些用法
copyright: true
date: 2018-11-21 22:25:11
tags:
	- FastJson
categories:
	- 小知识
---

## FastJson中的一些用法总结

fastJson在工作过程中经常用到的一个工具类，之前用到的最多的是在输出日志的时候的对java对象输出序列化之后的json字符串，最近在消费消息端也用到了JsonObject这个类的一些功能，做个简单的FastJson功能类的查漏补缺。

### JSONObject

通过观察fastJson的源码我们可以发现JSONObject是实现了Map<String, Object>接口的，而且其中也定义了一个map字段，在初始化的时候也是根据是否需要有序来初始化为LinkedHashMap或者HashMap。可以说JSONObject就相当于一个Map<String,Object>。

<!-- more -->

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203001210655.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

#### JSONObject.parseObject方法

这个方法其实继承的JSON类中的方法，有多个参数的重载，返回JsonObject对象。那么其实就是将json转换成Map接口类型，然后将JSONObject中的map设置值返回的对象。

用法示例：

```java
JSONObject jsonObject = JSONObject.parseObject(msg);
```

当然和Json.parseObject一样，也可以在参数中加入Class参数，直接转成对应的类型对象

```
TeamES teamES1 = JSONObject.parseObject(msg, TeamES.class);
```

#### JSONObject.getXxx()方法

JSONObject对象既然存在，那么就会提供访问其中一些键值对的访问，其中的键值对就是json字符串对应的键值对。

1. Object get(Object key) 
   - 这个是根据key值获取对应的value，看方法签名就知道这个方法返回的是Object对象。

2. JSONObject getJSONObject(String key)
   - 这个方法是根据String的key（在使用的时候就是json字符串中的变量名称），再将value转换成对应的JsonObject对象。因为有时候我们json字符串反序列化出来的是一个嵌套对象，所以嵌套内部的json字符串也可以转换成为一个JSONObject对象。当然如果内部就是一个值的对象的时候会调用JSON的toJSON方法返回一个Object。
3. JSONArray getJSONArray(String key)
   - 有时候key对应的对象是一个数组，那么可以直接转换成一个JSONArray对象，JsonArray对象和JSONObject对象的设计是一样的，实现了List<Object>接口并且把list作为一个字段进行使用。
4. T getObject(String key, Class clazz)
   - 这个方法支持将JSONObject中的值根据传入的key和对应的Class直接转换为对应的T对象，这里T是泛型。同样的这个方法也支持传入Type和TypeReference
5. getXxx(String key)
   - 类似于getBoolean、getDouble、getLong、getBigDecimal等方法就非常明确了，就是根据String的key（变量名称）转换成对应的包装类型或者基本类型的值。这在我们去解析json字符串中的部分键值对比较有用。

#### 在代码中的应用

这里根据传入到底是{"KDT_ID":xxx}还是传入的{"TEAM_ES":"{...}"}，拿到对应的值，这里就可以应用我们上面提到的JSONObject类对应的方法。

```java
private TeamES convertTeamESFromMap(JSONObject map) {

    String key = map.keySet().iterator().next();

    TeamES teamES = null;
    switch (key) {
        case KDT_ID :
            Long kdtId = map.getLong(KDT_ID);
            teamES = constructFromDB(kdtId);
            break;
        case TEAM_ES :
            teamES = map.getObject(TEAM_ES, TeamES.class);
            break;
        default:break;
    }

    return teamES;
}
```