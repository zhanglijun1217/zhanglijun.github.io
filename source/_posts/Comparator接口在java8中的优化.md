---
title: Comparator接口在java8中的优化
copyright: true
date: 2020-03-22 22:33:02
tags:
	- Comparator接口
categories:	
	- Java语法
	- Java8
---

# 开始

Comparator接口或者Comparable接口在日常开发工作中是经常用到的，用于比较一组数据或者对象，在java8之后，也可以看到在Comparator接口中加入了一些default方法和static方法，这里做一个简单说明。

<!--more -->

## Comparator接口和Comparable接口

这两个接口首先要做一个简单区别。

### Comparable接口

```java
* Lists (and arrays) of objects that implement this interface can be sorted
 * automatically by {@link Collections#sort(List) Collections.sort} (and
 * {@link Arrays#sort(Object[]) Arrays.sort}).  Objects that implement this
 * interface can be used as keys in a {@linkplain SortedMap sorted map} or as
 * elements in a {@linkplain SortedSet sorted set}, without the need to
 * specify a {@linkplain Comparator comparator}.<p>
```

可以看到注释中说明了实现了该接口的对象，在数组中可以使用Collections.sort或者Arrays.sort方法实现排序，或者实现了该接口的对象可以作为sortedMap或者SortedSet的key。这里也提到我们不用制定一个排序或者作为key的Comparator接口。

```java
public interface Comparable<T> {
    /**
     * 省略部分注释
     * <p>The implementor must ensure <tt>sgn(x.compareTo(y)) ==
     * -sgn(y.compareTo(x))</tt> for all <tt>x</tt> and <tt>y</tt>.  (This
     * implies that <tt>x.compareTo(y)</tt> must throw an exception iff
     * <tt>y.compareTo(x)</tt> throws an exception.)
     */
    public int compareTo(T o);
}
```

在compareTo方法上的注释中提到，必须确保 `x.compareTo(y)`和`y.compareTo(x)`的结果是一致的，并且这也意味着当`x.compartTo(y)`抛出一个异常，那么`y.compareTo(x)`也应该去抛出一个异常，那么这里就思考到了一个关于null的设计：`null.compareTo(obj)`我们肯定知道会有NPE，那么你在实现compareTo方法的时候，如果`obj.compareTo(null)`这里也应该去抛出NPE。

这里就不去写具体的demo去演示了，这里理解为一个对象实现了Comparable接口，那么这个对象就是可比较的，并且在排序等场景下调用实现接口中的compareTo方法。

### Comparator接口

Comparator接口要理解为比较器，实现其接口的类其实是比较器的一种实现，相当于一个比较的函数定义。来看下他的注释：

```java
* A comparison function, which imposes a <i>total ordering</i> on some
 * collection of objects.  Comparators can be passed to a sort method (such
 * as {@link Collections#sort(List,Comparator) Collections.sort} or {@link
 * Arrays#sort(Object[],Comparator) Arrays.sort}) to allow precise control
 * over the sort order.  Comparators can also be used to control the order of
 * certain data structures (such as {@link SortedSet sorted sets} or {@link
 * SortedMap sorted maps}), or to provide an ordering for collections of
 * objects that don't have a {@link Comparable natural ordering}.<p>
```

这里我们看到Arrays、Collections也提供了重载的sort方法，支持传入一个集合/数组和Comparator接口的实例。当然当前列表/数组中的对象不一定是实现了Comparable接口。

类实现了comparable接口之后，可以直接调用排序方法；而当使用comparator时，不需要类实现，具体使用时（也就是调用某些方法时）的需要类和该comparator绑定起来来实现。comparable实现内部排序，Comparator是外部排序。

个人感觉Comparator接口更符合解耦的思想，更好维护些。



## java8之后的Comparator接口

在java8之后Comparator接口增加了很多default方法和static方法来方便定义比较器。

#### reserved方法

```java
/** * Returns a comparator that imposes the reverse ordering of this * comparator. * * @return a comparator that imposes the reverse ordering of this *         comparator. * @since 1.8 */
default Comparator<T> reversed() {    return Collections.reverseOrder(this);}
```



#### comparing方法

```java
    /**
     * Accepts a function that extracts a {@link java.lang.Comparable
     * Comparable} sort key from a type {@code T}, and returns a {@code
     * Comparator<T>} that compares by that sort key.
     *
     * <p>The returned comparator is serializable if the specified function
     * is also serializable.
     *
     * @apiNote
     * For example, to obtain a {@code Comparator} that compares {@code
     * Person} objects by their last name,
     *
     * <pre>{@code
     *     Comparator<Person> byLastName = Comparator.comparing(Person::getLastName);
     * }</pre>
     */
    public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor)
    {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
    }
```

comparing方法参数是一个函数式接口keyExtractor，意识即为指定排序对象中的排序键，这里注意排序键这里标注了Comparable接口。

同时我们也可以看到有重载的comparing方法：

```java
   /**
     * Accepts a function that extracts a sort key from a type {@code T}, and
     * returns a {@code Comparator<T>} that compares by that sort key using
     * the specified {@link Comparator}.
      *
     * <p>The returned comparator is serializable if the specified function
     * and comparator are both serializable.
     *
     * @apiNote
     * For example, to obtain a {@code Comparator} that compares {@code
     * Person} objects by their last name ignoring case differences,
     *
     * <pre>{@code
     *     Comparator<Person> cmp = Comparator.comparing(
     *             Person::getLastName,
     *             String.CASE_INSENSITIVE_ORDER);
     * }</pre>
     */
  public static <T, U> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor,
            Comparator<? super U> keyComparator)
    {
        Objects.requireNonNull(keyExtractor);
        Objects.requireNonNull(keyComparator);
        return (Comparator<T> & Serializable)
            (c1, c2) -> keyComparator.compare(keyExtractor.apply(c1),
                                              keyExtractor.apply(c2));
    }

```

第二个参数也很好理解，提取完sort key之后，要定义关于这个key的Comparator，在注释中的例子也比较好理解。 这里有个小tips：在String类中，提供了一个实现Comparator接口的常量来标识不对语言敏感的字典序排序器。

```java
 /**
     * A Comparator that orders {@code String} objects as by
     * {@code compareToIgnoreCase}. This comparator is serializable.
     * <p>
     * Note that this Comparator does <em>not</em> take locale into account,
     * and will result in an unsatisfactory ordering for certain locales.
     * The java.text package provides <em>Collators</em> to allow
     * locale-sensitive ordering.
     *
     * @see     java.text.Collator#compare(String, String)
     * @since   1.2
     */
    public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                         = new CaseInsensitiveComparator();
    private static class CaseInsensitiveComparator
            implements Comparator<String>, java.io.Serializable {
        // use serialVersionUID from JDK 1.2.2 for interoperability
        private static final long serialVersionUID = 8575799808933029326L;

        public int compare(String s1, String s2) {
            int n1 = s1.length();
            int n2 = s2.length();
            int min = Math.min(n1, n2);
            for (int i = 0; i < min; i++) {
                char c1 = s1.charAt(i);
                char c2 = s2.charAt(i);
                if (c1 != c2) {
                    c1 = Character.toUpperCase(c1);
                    c2 = Character.toUpperCase(c2);
                    if (c1 != c2) {
                        c1 = Character.toLowerCase(c1);
                        c2 = Character.toLowerCase(c2);
                        if (c1 != c2) {
                            // No overflow because of numeric promotion
                            return c1 - c2;
                        }
                    }
                }
            }
            return n1 - n2;
        }

        /** Replaces the de-serialized object. */
        private Object readResolve() { return CASE_INSENSITIVE_ORDER; }
    }


// 这里其实可以看到compareToIgnoreCase也是调用了这个实例的compare方法
public int compareToIgnoreCase(String str) {
        return CASE_INSENSITIVE_ORDER.compare(this, str);
    }
```

在Comparator接口中，也直接提供了具体类型的三个comparing方法：

```java
public static <T> Comparator<T> comparingInt(ToIntFunction<? super T> keyExtractor) {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> Integer.compare(keyExtractor.applyAsInt(c1), keyExtractor.applyAsInt(c2));
    }


public static <T> Comparator<T> comparingLong(ToLongFunction<? super T> keyExtractor) {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> Long.compare(keyExtractor.applyAsLong(c1), keyExtractor.applyAsLong(c2));
    }
    
     public static<T> Comparator<T> comparingDouble(ToDoubleFunction<? super T> keyExtractor) {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> Double.compare(keyExtractor.applyAsDouble(c1), keyExtractor.applyAsDouble(c2));
    }
```

#### thenComparing方法

```java
/**
     * Returns a lexicographic-order comparator with another comparator.
     * If this {@code Comparator} considers two elements equal, i.e.
     * {@code compare(a, b) == 0}, {@code other} is used to determine the order.
     *
     * <p>The returned comparator is serializable if the specified comparator
     * is also serializable.
     *
     * @apiNote
     * For example, to sort a collection of {@code String} based on the length
     * and then case-insensitive natural ordering, the comparator can be
     * composed using following code,
     *
     * <pre>{@code
     *     Comparator<String> cmp = Comparator.comparingInt(String::length)
     *             .thenComparing(String.CASE_INSENSITIVE_ORDER);
     * }</pre>
     */
    default Comparator<T> thenComparing(Comparator<? super T> other) {
        Objects.requireNonNull(other);
        return (Comparator<T> & Serializable) (c1, c2) -> {
            int res = compare(c1, c2);
            return (res != 0) ? res : other.compare(c1, c2);
        };
    }
```

从方法名称上知道这是当比较相同时的使用的一个排序规则，这里需要注意看具体实现是会先调用比较器实例中的compare方法来进行比较一轮，当结果等于0的时候才会调用other这个比较器规则进行比较。比如下面的代码：

```java
List<String> strings = Arrays.asList("def", "abc", "hel", "world");
        strings.sort(Comparator.comparingInt(String::length).reversed() //（1）
                .thenComparing(String::compareToIgnoreCase) // （2）
                .thenComparing(Comparator.reverseOrder()) // （3）这个比较器不会被应用 因为比较器（2）已经把结果比较出来了，并且没有相等的结果，这里不会再应用（3）比较器
        );

        System.out.println(strings); // 输出[world, abc, def, hel]
```

当然因为有了 comparing方法的支持，所以也就有了下面两个thenComparing的重载方法

```java
  default <U extends Comparable<? super U>> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor)
    {
        return thenComparing(comparing(keyExtractor));
    }
```



```java
 default <U> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor,
            Comparator<? super U> keyComparator)
    {
        return thenComparing(comparing(keyExtractor, keyComparator));
    }
```

这里也不赘述提供的thenComparingInt、thenComparingDouble这类的方法。

#### null 友好的比较器

看到Comparator接口中有两个对null友好的比较器方法：

```java
   /**
     * Returns a null-friendly comparator that considers {@code null} to be
     * less than non-null. When both are {@code null}, they are considered
     * equal. If both are non-null, the specified {@code Comparator} is used
     * to determine the order. If the specified comparator is {@code null},
     * then the returned comparator considers all non-null values to be equal.
     *
     * <p>The returned comparator is serializable if the specified comparator
     * is serializable.
     *
     * @param  <T> the type of the elements to be compared
     * @param  comparator a {@code Comparator} for comparing non-null values
     * @return a comparator that considers {@code null} to be less than
     *         non-null, and compares non-null objects with the supplied
     *         {@code Comparator}.
     * @since 1.8
     */
    public static <T> Comparator<T> nullsFirst(Comparator<? super T> comparator) {
        return new Comparators.NullComparator<>(true, comparator);
    }
    
    
    // null比非null元素都大的
     public static <T> Comparator<T> nullsLast(Comparator<? super T> comparator) {
        return new Comparators.NullComparator<>(false, comparator);
    }
```

这里是通过Comparators这个工厂类提供的NullComparator比较器实现的，看到注释有一条需要注意是如果不指定comparator参数，即传入null，那么所有的非null参数都会被视为相等。

集合中插入null记录的场景也不是很常见，知道有这么个null友好的Comparator即可。

## 彩蛋

这里遇到了一个使用比较器 类型推断导致编译不过的问题：

```java
        // 这里尝试使用lambda方式去实现根据字符串长度降序排序  使用方法引用不会有这问题 因为指定了String::length
        list.sort(Comparator.comparingInt(item -> item.length()).reversed()); 
// 注意如果调用了reversed方法，那么item.length会报编译错误。这里把item推断成了object类型。
// 但是不调用reversed是可以编译通过的
        // 这里是因为 失去了lambda的类型推断 具体见博客：https://blog.csdn.net/u013096088/article/details/69367260
```

