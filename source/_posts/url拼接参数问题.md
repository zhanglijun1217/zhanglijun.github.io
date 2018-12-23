---
title: url拼接参数问题
copyright: true
date: 2018-11-23 01:07:57
tags:
	- url拼接参数
categories:
	- bug记录
---

## 问题现象

在最近的开发过程中要根据一堆id值去删除ES中的数据，就写了一个脚本接口，传入了idList。这里选择的是GET方式的接口，将idList以逗号分隔当做字符串传入当做参数，然后在接口中转换成List类型再对ES进行操作。

<!-- more -->

## 脚本代码

这个接口中的process是为了控制是否真正执行刷数据的逻辑，在一些刷数据的接口中加入这个参数，可以去在真正去刷数据之前，去看看捞出来的数据是否正确，然后再进行刷数据的逻辑。

```java
@GetMapping(value = "/es/fix/removeNotConsumerData")
@ResponseBody
public boolean removeNotConsumerData(@RequestParam(value = "kdtIdList") String kdtIdListString, @RequestParam(value = "process", defaultValue = "false") boolean process) {

    List<Long> kdtIdList = Arrays.stream(kdtIdListString.split(",")).map(Long::parseLong).collect(Collectors.toList());
    log.info("要刷的数据是：{}", JSON.toJSONString(kdtIdList));
    if (!process) {
        return true;
    }

    Set<Long> successSet = Sets.newHashSet();
    Set<Long> failSet = Sets.newHashSet();

    for (Long kdtId : kdtIdList) {
        try {
            ESResult esResult = esSyncHelper.deleteDoc(EsConstant.ESV5_TEAM_INDEX, EsConstant.ESV5_TEAM_TYPE, kdtId.toString(), ESResult.class);
            Thread.sleep(200);
            if (AppConstant.SUCCESS != esResult.getCode()) {
                log.warn("刷数据失败,kdtId={}, msg:{}", kdtId, esResult.getMessage());
                failSet.add(kdtId);
            } else {
                successSet.add(kdtId);
            }
        } catch (Throwable e) {
            log.warn("刷数据异常, kdtId={}", kdtId, e);
            failSet.add(kdtId);
        }

    }

    if (kdtIdList.size() != successSet.size()) {
        log.warn("有数据删除失败, failSet={}", JSON.toJSONString(failSet));
        return false;
    } else {
        return true;
    }

}
```



这里在机器上进行跑的时候，先去跑了单个id数据，发现是没有问题的。之后就想一把梭去将数据跑完，就把整个id集合数据放入了url要传入的参数之中，这个时候发现出现了问题。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203000852190.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

在数据的最后一条id值被截断了，只是去截止到了42122，所以这个会被转为42122存入要刷的id集合中。这个就可能去刷错了数据（所以之前的process参数还是很有用的 = =）。

这里有个比较稳的解决方案是要刷的数据可以把文件放在resource目录下，然后通过读取这个文件的内容直接去去刷数据，这样就不会存在这个问题了。这里去简单记录下这次参数被截断的过程。