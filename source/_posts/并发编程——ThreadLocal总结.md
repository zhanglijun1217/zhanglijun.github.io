---
title: 并发编程——ThreadLocal总结
copyright: true
date: 2018-08-16 11:06:09
tags:
    - ThreadLcoal
categories:
	- 并发编程
	- 并发工具
---

## 概念介绍
ThreadLocal是早期jdk版本中就有的一个工具，基本原理是同一个ThreadLocal所包含的对象（对ThreadLocal<String>而言即为String类型变量），在不同的Thread中有不同的副本（实际是不同的实例）。这里有几点需要注意：
​    <!-- more -->
  - 因为每个Thread内有自己的实例副本，且该副本只能由当前的Thread使用。这也是ThreadLocal命名的由来。
  - 既然每个Thread都有自己的实例副本，且其他的Thread不可访问，那么就不存在多线程共享的问题（其实ThreadLocal也不是去解决多线程共享的问题）。
  - 那么ThreadLocal解决了什么问题呢？ThreadLocal提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal变量通常被private static修饰。当一个线程结束时，它所使用的ThreadLocal相对的实例副本都可被回收。
  - ThreadLocal的适用场景：ThreadLocal适用于每个线程需要自己独立的实例且该实例需要在多个方法中使用，也即变量在线程间隔离而在方法或类间共享的场景。其实这种场景下并不只是可以用ThreadLocal去解决，只不过ThreadLocal更简洁。

## ThreadLocal原理
### 实现可能的猜想
#### ThreadLocal维护线程与实例的映射
- 既然每个访问ThreadLocal变量的线程都有自己的一个“本地”实例副本，那么可能的方案是ThreadLocal维护着一个Map，键是Thread，值是它在这个Thread中的实例。线程通过该ThreadLocal的get()方法获取实例时，只需要以线程为键，从map中获取实例即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203003111510.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)
- 这个方案却又有问题：
1. 增加线程和减少线程都需要去put、remove操作map,这个时候如果在一个ThreadLocal对该线程存入两个实例，就会有线程安全问题、
2. 线程结束时，需要保证它所访问的所有的ThreadLocal中的对应的映射均删除，否则可能会引起内存泄漏。
3. 第一个问题是jdk不去采取这种做法的原因。

#### ThreadLocal维护ThreadLocal与实例的映射
- 如果这个Map是每个线程去访问自己的一个Map，就不会产生多线程写的问题。map中维护着key为ThreadLocal实例，设计如下图所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203003103448.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)
- 这个方案中解决了map的线程安全问题，相当于第一种方法的倒转想法，map中key设置为ThreadLocal实例在不同线程中访问。
- 这种方案还是没有去解决内存泄漏问题。由于每个线程访问到ThreadLocal变量之后，都会在自己的Map内维护该ThreadLocal变量与具体实例的映射，如果不删除这些引用（映射），则这些ThreadLocal不能被回收，可能会造成内存泄漏。

### JDK中的解决
#### ThreadLocalMap
- 上边提到的维护的map是由ThreadLocal中的静态内部类ThreadLcoalMap去提供的，该类的实例维护着某个ThreadLocal与具体实例的映射。与HashMap不同的是，每个ThreadLocalMap的每一个Entry都是一个对键的弱引用，这一点可以从super(k)可以看出。每一个Entry对key的引用是强引用。使用ThreadLocal弱引用的原因是可以被及时回收。但是这里不能解决Entry引用内存泄漏的问题。当ThreadLocal变量被回收之后，该映射的键值变为null，该Entry无法被移除。从而也有可能造成内存泄漏。（下面会提到JDK的解决）ThreadLocalMap中的Entry代码如下：
```
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** The value associated with this ThreadLocal. */
  Object value;
  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

#### 读取实例
```
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
```
- 读取实例时，线程首先通过getMap(t)方法获取自身的ThreadLocalMap。获取到ThreadLocalMap后，通过map.getEntity(this)方法获取该ThreadLocal在当前线程的ThreadLocalMap中的Entry。该方法中的this即当前访问的ThreadLocal方法。如果获取到的Entry不为null，从Entry中取出值即为所需访问的本线程对应的实例。如果获取到的Entry为null，则通过setInitialValue()方法设置该ThreadLocal变量在该线程中对应的具体实例的初始值。设置初始值的方法如下：
```
private T setInitialValue() {
  T value = initialValue();
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
  return value;
}
```
- 注意此方法为private方法，无法被重载。
- 首先，通过initialValue()方法能生成一个初始值，这个方法是一个public方法，且默认值为null。所以典型用法中常常去重载该方法去给一个默认值。
然后，通过当前线程对象拿到ThreadLocalMap对象，若该对象不为null，则直接塞入map中set进去线程内实例的值。如果map为null，则去创建该ThreadLcoalMap对象。

#### 设置实例。
- 设置实例的方法也是采用了上述方法中的原理，不多做解释了。
```
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

#### 防止内存泄漏
- 对于已经不再使用且已被回收的ThreadLocal对象，它在每个线程内对应的实例由于被线程的TheradLcoalMap的Entry强引用，无法被回收，可能会造成内存泄漏。
- 针对该问题，ThreadLocal的set方法中去做了处理。replaceStaleEntry方法将所有键为null 的Entry的值设置为null，从而使得该值可被回收。另外，会在rehash方法中通过 expungeStaleEntry 方法将键和值为null的Entry设置为null从而使得该 Entry可被回收。通过这种方式，ThreadLocal可防止内存泄漏。

```
private void set(ThreadLocal<?> key, Object value) {
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);
  for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();
    if (k == key) {
      e.value = value;
      return;
    }
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}
```

## ThreadLocal的适用场景
- 每个线程需要自己有单独的实例
- 实例需要在对个方法中共享，但不希望被多线程共享。

## Threadlocal一个工具类总结
```
/**
 * threadLocal工具类
 * @author 夸克
 * @create 2018/8/15 16:47
 */
public class ThreadLocalUtil {

    /**
     * 不同的业务区分ThreadLocal中map的key
     * （这里的map不是threadLocal中对应线程的threadLocalMap，而是要塞入线程中的map的值，
     * 这里可能在一个业务域中一个线程存在多次使用ThreadLocal，所以在threadLocal中塞入的是个map。而
     * 当前线程中存放的是<threadLocal对象,<业务key, 真正要使用的变量>>）
     * threadLocal内存泄漏问题（（1）ThreadLocalMap中Entry的引用没有释放）在jdk8中得到了解决，
     * 对ThreadLocalMap中的键值threadLocal实例的引用改为弱引用
     * 所以建议使用ThreadLocal
     */


    /**
     * 业务前缀key值的维护
     */
    public enum Key {

        /**
         * 测试使用
         */
        COMMON_TEST("COMMON_TEST");

        private String key;

        Key(String key) {
            this.key = key;
        }

    }

    private static final ThreadLocal<Map<String, Object>> THREAD_LOCAL = new ThreadLocal<>();

    /**
     * set方法
     * @param key
     * @param value
     */
    public static void set(String key, Object value) {
        if(THREAD_LOCAL.get() == null) {
            // 初始化
            init();
        }
        if (StringUtils.isEmpty(key) || Objects.isNull(value)) {
            return;
        }
        THREAD_LOCAL.get().put(key, value);
    }

    /**
     * get方法
     * @param key
     * @return
     */
    public static Object get(String key) {
        if (StringUtils.isEmpty(key)) {
            return null;
        }
        return THREAD_LOCAL.get().get(key);
    }

    /**
     * 刷新方法
     */
    public static void refresh() {
        if (THREAD_LOCAL.get() == null) {
            return;
        }
        // map清除 key value
        THREAD_LOCAL.get().clear();
        // 清除map
        THREAD_LOCAL.set(null);
        // 线程中ThreadLocalMap remove
        THREAD_LOCAL.remove();
    }

    private static void init() {
        THREAD_LOCAL.set(Maps.newHashMap());
    }
```
