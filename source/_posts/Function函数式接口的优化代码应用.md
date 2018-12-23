---
title: Function函数式接口的优化代码应用
copyright: true
date: 2018-09-02 14:27:40
tags:
 - 函数式接口
 - 代码优化
categories:
	- Java语法
	- Java8
---

## 前言

函数式接口之前就一直在接触过，之前在github上写过关于几个函数式接口简单应用的代码，但一直没有记录在工作中的应用，这次就用Function接口优化了一次重复代码的警告。关于函数式接口不熟悉的同学，可以先看下我在github上的代码工程：[java8](https://github.com/zhanglijun1217/java8)

<!-- more -->

## 优化记录

### 优化前的代码

数据底层提供了查询报表四个不同纬度的接口，而接口中的方法其实都是一样的：比如通过部门去查、通过部门下的人去查找报表数据，而在应用层如果去写这些查询接口的话，就是每种报表都要去查子部门和人员的数据（真实的情况更多），那么很自然的就在每种报表的实现方法中去根据查询部门还是人或者其他的可能的去进行数据的查询。

![](http://pcsb3jzo3.bkt.clouddn.com/WechatIMG72.jpeg)

这里可以看到，这只是销售报表这一种情况，然后比如成单量、业绩额、成单率等等也是这样写的。这时候在IDEA中就会报一个警告：Duplicated Code，因为除了调用底层的接口不一样之外，其余的逻辑都是一样，并且这四个service没有去继承一个抽象类去做封装，并且这四个的返回值也不能去做抽象，并且在最后是要根据返回值的类型去进行bean转换，这时候不能简单的去根据泛型去抽象出private方法，这里想到了用函数式接口去做。

### 优化过程

#### Function接口定义

注意到查询底层报表数据是有的要传入两个参数的，也有的是要传入三个参数的，所以我们就需要多参数的Function接口，Java中为我们提供了BiFunction，也就是两个参数的Function接口，但是三个参数的函数式接口要我们自己定义。这里去定义一个传入三个参数和一个返回值的函数式接口。

```
@FunctionalInterface
public interface TripleFunction<T, U, K, R> {

    /**
     * Applies this function to the given arguments
     *
     * @param t the first function argument
     * @param u the second function argument
     * @param k the third function argument
     * @return the function result
     */
    R apply(T t, U u, K k);

}
```

这里可以看到这个接口中有@FunctionalInterface注解和apply方法。

#### 查询报表逻辑处理

这里去写一个查询报表的通用逻辑处理。首先看参数，condition是传入的筛选条件，clazz是返回值泛型R的class，queryUserFun是传入两个参数的计算出T的的函数，同样queryDeptByIdFunc是传入两个参数，计算出PlainResult的函数式参数。剩下两个函数参数是传入三个参数计算出ListResult参数。

这里为什么要去传R、T两个泛型？因为这里直接调用rpc接口返回的结果是他们封装的一个DTO对象，我们要自爱应用层自己运用orika工具进行DTO的转换（防污染和降低耦合）。那为什么要单独去传入一个最后返回值得class对象？这里是因为泛型在运行时会被擦除，要使用orika去转换DTO时要进行class的参数传入。

```
 /**
     * 查询报表的方法
     *
     * @param condition 筛选条件
     * @param clazz 结果class
     * @param queryUserFunc 查询user函数
     * @param queryUsersFunc 查询users函数
     * @param queryDeptFunc 查询dept函数
     * @param <T> 泛型1
     * @param <R> 泛型2
     * @return
     */
    private <T, R> Pagination<R> getReport(AchievementPkDTO condition, Class<R> clazz,
            BiFunction<Long, DateRangeDTO, ListResult<T>> queryUserFunc,
            BiFunction<Long, DateRangeDTO, PlainResult<T>> queryDeptByIdFunc,
            TripleFunction<Long, DateRangeDTO, OrderAndPageDTO, ListResult<T>> queryUsersFunc,
            TripleFunction<Long, DateRangeDTO, OrderAndPageDTO, ListResult<T>> queryDeptFunc) {

        DateRangeDTO dateRangeDTO = convertDateRangeDTO(condition);
        OrderAndPageDTO orderAndPageDTO = convertOrderAndPageDTO(condition);

        // 初始化
        ListResult<T> result = new ListResult<>();

        // 如果是有赞部门节点
        if (OrganizationType.YOUZAN_SUB_DEPARTMENTS.equals(condition.getOrganizationType())) {
            result = queryDeptFunc.apply(condition.getOrganizationId(), dateRangeDTO, orderAndPageDTO);
        } else if (OrganizationType.YOUZAN_USER.equals(condition.getOrganizationType())) {
            // 人的节点
            result = queryUserFunc.apply(condition.getOrganizationId(), dateRangeDTO);
        } else if (OrganizationType.YOUZAN_DEPARTMENT_USERS.equals(condition.getOrganizationType())) {
            // 部门人的节点
            result = queryUsersFunc.apply(condition.getOrganizationId(), dateRangeDTO, orderAndPageDTO);
        } else if (OrganizationType.SINGLE_PROVIDER.equals(condition.getOrganizationType())) {
            // 单个渠道商节点
            PlainResult<T> apply = queryDeptByIdFunc.apply(condition.getOrganizationId(), dateRangeDTO);
            // 设置result信息
            result.setData(Collections.singletonList(CheckWrapper.checkWrap(apply)));
            result.setCount(1);
            result.setSuccess(true);
        }

        List<R> rList = orikaBeanUtil.convertList(CheckWrapper.checkWrap(result), clazz);
        return new Pagination<>(rList, condition.getPage(), condition.getPageSize(), result.getCount());
    }
```

#### 实现不同种报表的查询

有了上述通过Function接口的改造，使得不同的业务场景（业绩、成单量等）都可以传入一个lambda表达式去调用上边封装的获取参数的方法。比如下面的这两个方法（这样并不会报Duplicated Code警告）：

```
 @Override
    public Pagination<DealAmountStatistics> getDealReport(AchievementPkDTO condition) {


        if (Objects.equals(OrganizationType.PROVIDER, condition.getOrganizationType())) {
           return doWithDealProvider(condition);
        } else {
            return getReport(condition, DealAmountStatistics.class,
                    (organizationId, date) -> dealAmountStatisticsService.getUserById(organizationId, date),
                    (organizationId, date) -> dealAmountStatisticsService.getDepartmentById(organizationId, date),
                    (organizationId, date, range) -> dealAmountStatisticsService.getUsersByDepartmentId(organizationId, date, range),
                    (organizationId, date, range) -> dealAmountStatisticsService.getDepartmentsByParentId(organizationId, date, range));
        }

    }
    
    

    @Override
    public Pagination<OrderNumStatistics> getOrderNumberReport(AchievementPkDTO condition) {
        if (Objects.equals(OrganizationType.PROVIDER, condition.getOrganizationType())) {
            return doWithOrderNumProvider(condition);
        } else {
            return getReport(condition, OrderNumStatistics.class,
                    (organizationId, date) -> orderNumStatisticsService.getUserById(organizationId, date),
                    (organizationId, date) -> orderNumStatisticsService.getDepartmentById(organizationId, date),
                    (organizationId, date, range) -> orderNumStatisticsService.getUsersByDepartmentId(organizationId, date, range),
                    (organizationId, date, range) -> orderNumStatisticsService.getDepartmentsByParentId(organizationId, date, range));
        }

    }
```

这里去因为PROVIDER类型节点要对不同的返回值做特殊处理，否则这两个方法还可以抽象成为一个公用方法，这里不去对抽象过多要求，主要想记录学习的是用函数式接口去优化代码减少了代码重复行数，并且lambda表达式作为参数的一个使用也没有降低可读性。之后在写代码的过程中可以多应用这种设计，去让代码变得更加简洁，使用更多的新特性。