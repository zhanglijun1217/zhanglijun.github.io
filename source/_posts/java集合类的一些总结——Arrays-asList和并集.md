---
title: java集合类的一些总结——Arrays.asList和Guava操作集合
copyright: true
date: 2018-12-15 00:59:34
tags:
	- Arrays.asList
	- 集合操作
categories:
	- Java基础
---

## 使用背景
总结一下最近项目中使用到集合的两个点，一个是Arrays.asList这个方法使用的坑，另一个是利用Guava的Sets工具类去求并交集。

<!-- more -->

## 使用总结
### Arrays.asList的坑
先上结论：
- Arrays.asList这个方法不适用于基本类型：byte,short,int,long,float,double,boolean
- 该方法将数组和列表动态链接起来，当其中一个更新后，另一个也会更新
- 不支持add和remove方法
  （1）首先对于第一点，这个方法不适用于基本类型，并不是不能用于基本类型，看一下下面的代码：
```java
    public static void main(String[] args) {
        int[] int_array = {1, 2, 3};
        List<int[]> ints = Arrays.asList(int_array);
        // 对于基本类型，会打印一个元素结果
        System.out.println(ints);

    }
```
输出：
```
[[I@610455d6]
```
从Arrays.asList返回List的泛型可知，这个工具方法将基本类型的数组只转成了一个元素的列表。因为在Arrays.asList中，该方法接受一个变长参数，一般可以看做数组参数，但是因为int[]本身就是一个类型，所以data变量作为参数传递时，编译器认为只传了一个变量，这个变量的类型是int数组，所以会这样。基本类型是不能作为泛型的参数，按道理应该使用包装类型，这里没有报错是因为数组是可以泛型化的，所以在转换后list中有一个类型为int[]的数组。值得注意的是，如果这时打印出其中的class，会返回class[I。因为jvm不可能输出int[]对应的array类型，array属于java.lang.reflect包，通过反射访问数组这个类，编译时生成的这个class值。
（2）对于第二点，我们可以尝试改变数组中的值时，可以看看对应list中的值：
```java
 String[] string_array = {"aa", "bb", "cc"};
        List<String> stringList = Arrays.asList(string_array);

        System.out.println("============改变数组前============");
        System.out.println(stringList);

        // 改变数组
        string_array[1] = "change";
        System.out.println("============改变数组后============");
        System.out.println(stringList);
        // 改变list
        stringList.set(1, "change2");
        System.out.println("============改变list后============");
        System.out.println(string_array[1]);
```
输出结果：
```java
============改变数组前============
[aa, bb, cc]
============改变数组后============
[aa, change, cc]
============改变list后============
change2
```
这个通过源码很好理解，可以看一下asList的源码：
```java
   @SafeVarargs
    @SuppressWarnings("varargs")
    public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
```
new ArrayList构造函数代码：
```java
private final E[] a;
 ArrayList(E[] array) {
            a = Objects.requireNonNull(array);
        }
```
从源码中得到asList就是将数组作为返回的List的实际引用，即List中的数组就是要转成list的数组，所以当更改数组中的值时，list中的值也会变化，list列表中的值变更也相当于变更了其中的数组，也就是源数据数组。这就是所谓的链接在一起。Arrays.asList体现的是适配器模式，只是转换接口（array到list），后台的数据仍是数组。
（3）第三点是数组转换之后的List是不支持remove和add方法的。这里说的不支持，是调用add或者remove方法之后会抛出一个异常 java.lang.UnsupportedOperationException。一开始感到奇怪，ArrayList为什么不支持add和remove方法？这里去看了返回的ArrayList的源码，这个ArrayList并不是常用的ArrayList，而是在Arrays中定义的一个内部类：
```java
 private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable
    {
        private static final long serialVersionUID = -2764017481108945198L;
        private final E[] a;

        ArrayList(E[] array) {
            a = Objects.requireNonNull(array);
        }
        ... // 省略一些代码
    }
```
这个内部类虽然继承了AbstractList，但是没有去实现其中的add和remove方法，可以看下AbstractList中定义的默认add方法的实现：
```java
/**
     * {@inheritDoc}
     *
     * <p>This implementation always throws an
     * {@code UnsupportedOperationException}.
     *
     * @throws UnsupportedOperationException {@inheritDoc}
     * @throws ClassCastException            {@inheritDoc}
     * @throws NullPointerException          {@inheritDoc}
     * @throws IllegalArgumentException      {@inheritDoc}
     * @throws IndexOutOfBoundsException     {@inheritDoc}
     */
    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }
```
可以看到如果没有去实现add方法，那么会抛出这个异常，这里因为也叫做ArrayList，所以还是比较容易踩坑。clear、remove方法也是一样的原因。
关于Arrays.asList的代码github地址：
https://github.com/zhanglijun1217/java8/tree/master/src/Arrays_asList

### Guava中的Sets工具类求交集
近期也去使用了一个求集合交集的场景。这里去简单记录下Guava中的这个用法。
```
Sets.intersection(set1, set2);
```
这个方法返回的是一个新的集合去承接传入两个集合的交集的。Guava也提供了对集合操作很多其他方法，比如并集、差集、全集等，这个后面会去总结。