---
title: java枚举拾遗
copyright: true
date: 2019-06-12 11:52:14
tags:
	- 枚举
categories:
	- Java基础
---

## 前言

java枚举是在开发过程中用的最多的类，这里对java之前的枚举常量类和枚举做了一个分析，并且对枚举相关知识拾遗。

## 枚举类

在出现枚举之前，通常是一个final类去表示"可枚举"这个概念，比如下面这个列举数字的枚举类

```java
/**
 * 模拟枚举类 (枚举类：在enum出现之前的表达 可枚举的含义的类)
 * 通常 private 构造函数
 * final class
 * private static final 本类型 成员
 *
 */

final class EnumClass{
    public static final EnumClass ONE = new EnumClass(1);
    public static final EnumClass TWO = new EnumClass(2);
    public static final EnumClass THREE = new EnumClass(3);
    public static final EnumClass FOUR = new EnumClass(4);

    @Getter
    @Setter
    private int value;
    private EnumClass(int value) {
        this.value = value;
    }
  
  	public void print() {
        System.out.println(this.toString());
    }

}
```

可以看到枚举类的特点：

- 成员用常量来表示，并且类型为当前类型(当前类型)
- 常被设置为final类
- 非public构造器(自己内部来创建实例)

这样有些缺点，比如：

- 枚举的输出打印的时候要怎么做？每个成员是第几个定义的？要想达到这些操作就必须要写一些方法，而每个枚举类去这样写这样的方法是比较蛋疼的，因为他不具有枚举的values方法。

## 枚举

这里写出对应的java枚举

```java
enum CountingEnum {
    ONE(1),
    TWO(2),
    THREE(3),
    FOUR(4)
    ;
    @Getter
    private int value;

    /** private */ CountingEnum (int value) {
        this.value =  value;
    }
}
```

这里如果想要输出对应的名字和顺序，那么就十分方便了。

```java
 // 输出 枚举 中的名字、位置、输出所有枚举
        Arrays.stream(CountingEnum.values()).forEach(e -> {
            System.out.println("输出枚举中的顺序: " + e.ordinal() + "名字：" + e.name() + "value：" + e.getValue());
        });
```

可以看到输出：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/WX20190614-153052%402x.png)

这是因为java所有的枚举都是继承Enum抽象类的，而valueOf()方法、ordinal()方法、name()方法都是定义在其中的，可以看下Eunm抽象类。

```java

public abstract class Enum<E extends Enum<E>>
        implements Comparable<E>, Serializable {
    
    private final String name;

    public final String name() {
        return name;
    }

    private final int ordinal;

   
    public final int ordinal() {
        return ordinal;
    }

   
    protected Enum(String name, int ordinal) {
        this.name = name;
        this.ordinal = ordinal;
    }

  
    public String toString() {
        return name;
    }

  
    public final boolean equals(Object other) {
        return this==other;
    }

   
    public final int hashCode() {
        return super.hashCode();
    }

    
    protected final Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException();
    }

   
    public final int compareTo(E o) {
        Enum<?> other = (Enum<?>)o;
        Enum<E> self = this;
        if (self.getClass() != other.getClass() && // optimization
            self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        return self.ordinal - other.ordinal;
    }

    
    @SuppressWarnings("unchecked")
    public final Class<E> getDeclaringClass() {
        Class<?> clazz = getClass();
        Class<?> zuper = clazz.getSuperclass();
        return (zuper == Enum.class) ? (Class<E>)clazz : (Class<E>)zuper;
    }

    
    public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                                String name) {
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }

    /**
     * enum classes cannot have finalize methods.
     */
    protected final void finalize() { }

    /**
     * prevent default deserialization
     */
    private void readObject(ObjectInputStream in) throws IOException,
        ClassNotFoundException {
        throw new InvalidObjectException("can't deserialize enum");
    }

    private void readObjectNoData() throws ObjectStreamException {
        throw new InvalidObjectException("can't deserialize enum");
    }
}

```

仔细看也许你会有两个疑问：

- 没看到显示的定义CountingEnum时继承Enum类的？
- values方法也没有看到在父类中定义？

对这两个疑问我们可以去看这个类对应的字节码：
![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E6%9E%9A%E4%B8%BEclass.png)

可以看到：

- enum其实也是final class， 虽然没有显示继承，但是其实是继承了`Enum<T>`类的，所以可以访问到对应的name，ordinal字段，这个设计让我们输出枚举一些信息的时候很便捷。也提供了valueOf方法，也可以在动态判断枚举的时候使用。

在来看下边的字节码：

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/%E6%9E%9A%E4%B8%BE2.png)

可以看到是有values方法，其实这个是jvm通过字节码提升的方式去为枚举做的优化。所以使用枚举可以快速遍历并且一些输出之类的操作。

可以总结下枚举的特点：

- 枚举其实就是final class，并且继承java.lang.Enum抽象类。
- 枚举可以实现接口。
- 枚举不能显示的继承和被继承
- 留个坑：既然是final class，那么枚举里可以定义抽象方法吗？



## 枚举中抽象方法的设计

我们在看基础语法的时候，总是说final 和 abstract是互斥的，所以想当然的认为枚举中不能定义抽象方法，但结论其实是可以的。

我们先看一个枚举来实现加减操作的例子：

```java
enum Opration {
    PLUS,
    DIVIDE
    ;
    public double apply(double x, double y) {
        switch (this) {

            case PLUS:
                return x + y;
            case DIVIDE:
                return x - y;
        }
        throw new AssertionError("unknown");
    }
}
```

这个实现其实是通过在枚举中加入了非枚举含义的方法和域来实现的操作的一个类型枚举。但是有个问题，当拓展新的操作符时，需要破坏switch中的逻辑，这个不太符合开闭原则，这时候就可以通过把apply作为抽象方法，使得拓展时只需要实现符合自己的抽象逻辑。

```java
/**
 * 通过抽象方法， 来实现加入新的操作的时候 能符合开闭原则，只关心自己操作符抽象的实现
 */
enum OperationOptimize {
    PLUS("+"){
        @Override
        public int apply(int x, int y) {
            return x + y;
        }
    },
    DIVIDE("-") {
        @Override
        public int apply(int x, int y) {
            return x - y;
        }
    }
    ;

    @Getter
    private String str;
    private OperationOptimize(String str) {
        this.str = str;
    }

    // 抽象方法
    public abstract int apply(int x, int y);
}
```

所以枚举是可以定义抽象方法的。

jdk中其实也有对应的例子，可以看下TimeUnit这个时间单位枚举，枚举类型都是通过实现抽象方法(其实是返回异常的普通方法，思想是一样的)来实现不同时间单位的转化。

## 彩蛋

如何给上边的枚举类实现一个values方法？

因为需要遍历所有的字段，所以很自然的想到了反射去实现。这里需要注意，因为枚举类定义的枚举都是public static final，而作为val变量是int的一个修饰符，需要将除了枚举外的val变量排除~

示例代码：

```java
final class EnumClass {
    public static final EnumClass ONE = new EnumClass(1);
    public static final EnumClass TWO = new EnumClass(2);
    public static final EnumClass THREE = new EnumClass(3);
    public static final EnumClass FOUR = new EnumClass(4);

    @Getter
    @Setter
    private int value;

    private EnumClass(int value) {
        this.value = value;
    }

    @Override
    public String toString() {
        return "EnumClass{" +
                "value=" + value +
                '}';
    }

    public void print() {
        System.out.println(this.toString());
    }

    /**
     * 为枚举类实现一个values方法
     */

    public static EnumClass[] values() {
        // 获取枚举类中所有字段
        return Stream.of(EnumClass.class.getDeclaredFields())
                // 过滤出 public static final的
                .filter(field -> {
                    // 修饰符
                    int modifiers = field.getModifiers();
                    return Modifier.isPublic(modifiers)
                            && Modifier.isStatic(modifiers)
                            && Modifier.isFinal(modifiers);
                })
                // 取出对应的字段值
                .map(field -> {
                    try {
                        return field.get(null);
                    } catch (IllegalAccessException e) {
                        throw new RuntimeException(e);
                    }
                }).toArray(EnumClass[]::new);
    }
}
```

