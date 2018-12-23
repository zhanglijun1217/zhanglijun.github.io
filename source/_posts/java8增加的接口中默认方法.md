---
title: java8增加的接口中默认方法
copyright: true
date: 2018-10-10 17:06:06
tags:
	- Java8
	- 接口默认方法
categories:
	- Java语法
	- Java8
---

## 前言

最近在工作中的一次小修改让自己应用到了java8中的新特性：接口默认方法，这里去简单记录下。在java8之后可以在接口定义方法的实现，成为default方法，类似于Scala中的trait。比如在Iterable接口中新增了foreach默认方法：<!-- more -->

```
/**
 * Performs the given action for each element of the {@code Iterable}
 * until all elements have been processed or the action throws an
 * exception.  Unless otherwise specified by the implementing class,
 * actions are performed in the order of iteration (if an iteration order
 * is specified).  Exceptions thrown by the action are relayed to the
 * caller.
 *
 * @implSpec
 * <p>The default implementation behaves as if:
 * <pre>{@code
 *     for (T t : this)
 *         action.accept(t);
 * }</pre>
 *
 * @param action The action to be performed for each element
 * @throws NullPointerException if the specified action is null
 * @since 1.8
 */
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

这个default方法的主要目的是为java8的Lambda表达式提供支持，如果将这个方法定义为普通接口方法，则会对现有的JDK的其他使用Iterable接口的类造成影响，因此提供了default方法的功能。

### 工作中的例子

有一个小需求是对一个表的插入的实体的name字段做一个处理，这里name字段如果为空则用手机号（默认手机号不为空）进行插入。因为调用这个insert方法的业务代码比较多，每个都去做这个逻辑会显得很麻烦并且很重复。所以就想到了直接在mapper的xml中去进行修改： 

```
insert into table_name (
		<if test="name ==null or name == ''">NAME,</if> 
) values(
        <if test="name == null or name == ''">#{mobile}</if>
)

```

这样确实能够实现我们这个小需求，但是这个insert的mapper.xml是通过代码工具自动生成的标准insert方法，并且这样写可读性也不好，给人一种很奇怪的感觉。

这时候就用到了默认方法。这里我们可以在接口中定义一个默认方法insert，然后将之前的insert方法更换名称，在默认方法中去调用更换之后插入方法，而在默认方法中去做如果name为空则用手机号去代替这个逻辑。

```
int insertDefault(Clues entity);

/**
 *
 * @param entity
 * @return
 */
default int insert(Clues entity) {
    if (StringUtils.isEmpty(entity.getName()) && StringUtils.isNotEmpty(entity.getMobile())) {
        entity.setName(entity.getMobile());
    }
    return insertDefault(entity);
}
```

这样原来调用的insert方法也不需要去做更改，并且也不用在xml中进行改动就实现了这个小逻辑。注意要将xml中方insert方法改为insertDefault，这个更改比上边那种修改要显得合理的多。

### 默认方法的一个总结

java是面向对象的语言，那么就会有实现接口和继承父类，那么这些会对接口的默认方法有什么影响呢？下边参考博客：[默认方法](http://irusist.github.io/2016/01/02/Java-8%E4%B9%8Bdefault%E6%96%B9%E6%B3%95/)

存在一个父接口，定义了一个default方法：

```
public interface Parent {
    default String doit() {
        return "Parent";
    }
}
```

有一个类实现该接口，使用了默认的default方法：

```
public class ParentImpl implements Parent{

}
```

有一个ParentImpl2继承了ParentImpl，里面重写了接口中的默认方法：

```
public class ParentImpl2 extends ParentImpl {

    @Override
    public String doit() {
        return "ParentImpl2";
    }
}
```

有一个Child接口继承了Parent接口，并且重写了Parent接口中的默认方法：

```
public interface Child extends Parent {
    /**
     * 重写父接口中的default方法
     * @return
     */
    @Override
    default String doit() {
        return "Child";
    }
}
```

有一个ChildImpl实现了Child：

```
public class ChildImpl implements Child {
}
```

又有一个ChildImpl2继承了ParentImpl2也实现了Child接口，为了测试当子类实现的接口和继承的父类中都有默认方法的场景:

```
public class ChildImpl2 extends ParentImpl2 implements Child {
}
```

测试类：

```
public class Main {

    public static void main(String[] args) {
        /**
         * 测试实现类可以直接调用接口中的default方法
         */
        Parent parentImpl = new ParentImpl();
        // 将输出Parent
        System.out.println(parentImpl.doit());

        /**
         * 测试Child接口重写了Parent接口的default方法
         */
        Child child = new ChildImpl();
        // 将输出Child
        System.out.println(child.doit());

        /**
         * 测试ParentImpl2重写了Parent接口中default方法
         */
        Parent parentImpl2 = new ParentImpl2();
        // 将输出ParentImpl2
        System.out.println(parentImpl2.doit());

        /**
         * 测试ChildImpl2父类和实现的接口都有default方法，优先使用父类中定义的方法
         */
        Child childImpl2 = new ChildImpl2();
        // 将输出ParentImpl2
        System.out.println(childImpl2.doit());

    }
}
```

从上述测试结果可以看出：

1. 实现类可以直接使用父接口中定义的default方法。

2. 接口可以重写父接口中定义的default方法。

3. 实现类可以重写父接口中定义的方法、

4. 当父类和父接口都存在default方法时，使用父类中重写的default方法

特别的，如果一个类实现了两个接口，这两个接口中有同名的default方法签名时，此时会编译不通过，必须在子类中重写这个default方法。