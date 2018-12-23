---
title: 并发编程——CopyOnWrite容器
copyright: true
date: 2018-08-04 15:03:36
tags:
	- copyOnWrite
categories:
	- 并发编程
	- 并发工具
---
## 前言
   Copy-On-Write简称COW，是一种用于程序设计中的优化策略。其基本思路是，从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，才会真正把内容Copy出去形成一个新的内容然后再改，这是一种延时懒惰策略。从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是CopyOnWriteArrayList和CopyOnWriteArraySet。CopyOnWrite容器非常有用，可以在非常多的并发场景中使用到。

## 什么是CopyOnWrite容器
从字面意思上看是写时复制的容器。通俗理解就是我们往一个容器中添加元素时，不直接往容器中添加，而是先将容器copy，复制出一个新的容器，然后新的容器里添加元素，添加完新的元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，但不需要加锁，因为当前在读的容器中不会添加新的元素，运用一种读写分离容器的思想。
<!-- more -->
## copyOnWriteArrayList实现原理
来看几个方法
（1）add (E e)
这里是要加锁的，因为在add的时候是要Arrays.copyOf出一个容器副本的，如果多线程访问会造成copy多个容器副本出来。
可以看到是在copy完成添加元素之后去将引用指向这个新的数组。
```
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
（2）get(int index)
读的时候并没有加锁，如果读的时候有线程同时去添加元素，还是会读到之前的旧的容器，所以并不用加速，但要求对读的数据一致性没那么高。
```

    /**
     * {@inheritDoc}
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        return get(getArray(), index);
    }

```
## 应用场景
- copyOnWrite并发容器用于读多写少的并发场景。比如白名单、黑名单、商品类目等。比如有一个搜索网站，用户在这个网站的搜索框中，输入关键字搜索，但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单中，黑名单每晚会更新一次。当用户搜索时，会检查关键字在不在黑名单中，如果在，则提示不能搜索。
- 另外迭代操作远远大于修改操作时，才应该使用“写入时复制”容器。这个准则很好的描述了许多事件通知系统：在分发通知时需要迭代已注册监听器链表，并调用每个监听器，在大多数情况下，注册和注销事件监听器的操作远少于接受事件通知的操作。
- copyOnWriteArrayList用于替代同步list，在某些情况下它提供了更好的并发性能，并且能在迭代期间不需要对容器进行加锁或者复制。

## 实现一个copyOnWriteMap容器
- 简单根据“写入时复制”的思想实现一个map容器，并做简单测试
```
package collection;

import java.util.AbstractMap;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 简单根据CopyOnWrite容器的思想去实现一个map 只实现了get put putAll方法 且一些临界异常条件没有去处理
 *
 * @author 夸克
 * @create 2018/7/8 15:58
 */
public class CopyOnWriteMap<K, V> extends AbstractMap<K, V> implements Cloneable {

    private volatile Map<K, V> internalMap;

    public CopyOnWriteMap() {
        internalMap = new HashMap<>();
    }

    /**
     * 此方法未实现
     */
    @Override
    public Set<Entry<K, V>> entrySet() {
        return null;
    }

    @Override
    public V get(Object key) {
        // 读时不加锁
        return internalMap.get(key);
    }

    @Override
    public V put(K key, V value) {
        // 写时复制加锁
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }

    @Override
    public void putAll(Map<? extends K, ? extends V> data) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<>(internalMap);
            newMap.putAll(data);
            internalMap = newMap;
        }
    }


    public static void main(String[] args) {
        CopyOnWriteMap<Integer, Integer> copyOnWriteMap = new CopyOnWriteMap();
        // 初始化数据
        Map<Integer, Integer> map = new HashMap();
        for (int i = 0; i < 10; i++) {
            map.put(i, i);
        }

        // 读五次线程
        for (int i = 0; i < 5; i++) {
            Thread read = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        // 方便写线程写入数据 如果不加sleep 是读不到数据的，因为是在新复制的容器中写。 
                        // 测试copyOnWrite思想
                        Thread.sleep(500);
                    } catch (InterruptedException e) {

                    }
                    System.out.println(copyOnWriteMap.get(5));
                }
            });
            read.start();
        }

        // 写线程
        Thread write = new Thread(new Runnable() {

            @Override
            public void run() {
                map.forEach((k, v) -> {
                    copyOnWriteMap.put(k, v);
                });
            }
        });
        write.start();

    }

}

```

## 代码地址
https://github.com/zhanglijun1217/juc

## 缺点
- 显然，每次修改容器的时候都会复制底层数组，这回造成一定的内存开销，特别是当容器的规模很大的时候，可能有将内存撑爆的可能性存在。这时候可能要考虑别的容器。另外上边也提到了这种复制一份新的容器延迟的做法会有数据一致性的问题，如果你对写入的数据读出来实时性很高，那么久不要去选择copyOnWrite容器。

## 其他文章
- copyOnWriteArrayList于同步集合工具容器性能比较：[性能比较](http://blog.csdn.net/wind5shy/article/details/5396887)
- 简单使用：[简单使用](http://blog.csdn.net/imzoer/article/details/9751591)

## 引用说明
> 1.https://www.cnblogs.com/dolphin0520/p/3938914.html
2.《java并发编程实战》
