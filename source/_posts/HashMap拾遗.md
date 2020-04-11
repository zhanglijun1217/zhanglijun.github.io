---
title: HashMap拾遗（一）
copyright: true
date: 2020-03-28 23:07:34
tags:
	- HashMap
categories:	
	- Java基础
---

# 开始

HashMap是在开发工作中经常使用的集合类之一，熟悉其源码应该是基本要求。这篇文章对jdk1.8版本中的HashMap的一些常用方法的源码进行个记录。ps：这篇文章没有对其中的树化进行深究，比如提供的TreeNode内部类的结构和在扩容、Hash碰撞的时候的静态方法，之后有时间再研究下。

<!-- more -->

# 源码分析

## 1.1定义的变量

### 常量

```java

    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
```

- DEFAULT_INITIAL_CAPACITY：默认的初始化容量，必须是2的幂，这个值为16。
- MAXIMUM_CAPACITY：最大的容量，是1左移30位，相当于2的30次幂。
- DEFAULT_LOAD_FACTOR：默认的负载因子，值是0.75f。
- TREEIFY_THRESHOLD：树化的阈值，在整个数组的一个槽内，发生碰撞的链表如果超出整个阈值，就会转换为红黑树，在下面的代码也会分析到。
- UNTREEIFY_THRESHOLD：树变为链表的阈值。
- MIN_TREEIFY_CAPACITY：树化对应的最小容量阈值，上面的TREEIFY_THRESHOLD常量不是树化的唯一条件，HashMap在树化的方法中判断了当前的容量和这个值的比较，没有达到这个值，会先进行resize扩容操作。



### 变量字段

```java

    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * The number of key-value mappings contained in this map.
     */
    transient int size;

    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;

    /**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;

    /**
     * The load factor for the hash table.
     *
     * @serial
     */
    final float loadFactor;
```

- table：HashMap中的Node数组，Node是其中的内部类。
- entrySet：键值对的缓存。
- size：key-value对的数量。
- modCount：修改次数。比如在内部实现的foreach方法，可以看到在循环中应用传入的action之外，还做了对modCount的校验，这里有点cas的感觉，有预期值和当前值，如果不一致，就抛出`ConcurrentModificationException`这个异常，也被称为是HashMap的`fast-fail`机制。
- threshold：扩容的阈值。构造方法中直接通过tableSizeFor(initialCapacity)进行赋值，因为构造方法中tab数组并没有初始化，后来在put方法中初始化tab数组的时候重新对threshold进行了计算。
- loadFactor：负载因子。这里吐槽下经常看到面试问为啥是0.75，没get到问这个的意义在哪，设计jdk的人经过很多实验设置的值，并解释了符合泊松分布啥的，这种事了解下不就可以了吗，在工作中知道能自己设置这个值，不就可以了吗= =

### Node内部类

```java
 /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

可以看到其中只定义了简单的值：hash值、key、value、还有next节点。

## 1.2 构造方法

来看看HashMap的构造方法：

```java
   public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }


    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

 
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

可以看到提供了四个重载的构造方法，有：

- 设置初始化容量和负载因子的。
- 无参数。
- 设置初始化容量的。
- 参数是一个map的

可以看到在构造方法中并没初始化数组（除了传入一个map初始化），而是赋值了负载因子和通过`tableSizeFor`方法赋值了容量的阈值。

### tableSizeFor方法

来看看这个`tableSizeFor`方法：

```java

    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

这里将传入的容量参数-1，然后无符号右移1为左或操作，重复无符号右移2位、4位...，最后做了和最大容量值的比较。看注释知道这里返回的容量都是2的幂次方，这与下面定位数组中的位置有关，下面会做出解释。

## 1.3 put方法

```java
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

### hash方法

```java
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

hash方法将key的hashCode和其无符号右移16位的值进行了异或操作，这个操作 也是扰动函数，为了降低哈希码的冲突。右位移16位，正好是32bit的一半，高半区和低半区做异或，就是为了混合原始哈希码的高位和低位，以此来加大低位的随机性。而且混合后的低位掺杂了高位的部分特征，这样高位的信息也被变相保留下来。也就是保证考虑到高低Bit位都参与到Hash的计算中。 

这里看到我们已经做了相当于两次hash操作：
（1）取key的hashCode方法进行计算。

（2）hashCode无符号右移16位，与hashCode进行了异或。

一个key计算hash函数得到的值就是Node类中hash变量的值。

在后面定位元素在数组中的位置用了元素长度（这个上面提到了是2的次幂）-1去和hash值做了与操作：
`tab[i = (n - 1) & hash]`。这也相当于一个key传入HashMap定位到数组中的位置至少经历了3次hash操作。

在后边的方法代码中可以经常看到这行代码来定位数组中的位置，其实这里因为是2的次幂，所以做与操作相当于模除来定位数组中的位置。（参考这篇博客：[HashMap中的hash算法]( https://www.cnblogs.com/ysocean/p/9054804.html )）

### putVal方法

```java
/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key   //key的hash值
     * @param key the key //key的值
     * @param value the value to put // value值
     * @param onlyIfAbsent if true, don't change existing value // 是否覆盖
     * @param evict if false, the table is in creation mode. // false的话是在创建模式 
     * @return previous value, or null if none  // 返回前一个value，如果没有前值，返回null
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; // 数组变量 
        Node<K,V> p;
        int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            // 如果当前数组没有初始化或者length为0，则resize()方法进行初始化 
            n = (tab = resize()).length;//赋值初始化好的数组长度给n
        if ((p = tab[i = (n - 1) & hash]) == null) // 如果key对应数组下标i中的值为null
            tab[i] = newNode(hash, key, value, null); // 初始化一个Node对象作为存储在该位置的值
        else {
            Node<K,V> e; //e 这里理解为exist，即定位到数据下标i处已经有了node元素 
            K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // 如果i坐标下的hash与当前hash值相等 并且 key也和当前要put的值相等或者equals
                // exist的元素就是p 即当前i的元素
                e = p;
            else if (p instanceof TreeNode)
                // 如果当前i坐标的的node是树Node，则当前exist是调用了坐标i树Node元素的putTreeVal
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 当前i坐标的node元素不是树Node
                for (int binCount = 0; ; ++binCount) {
                    // 寻找当前坐标链表的下一个元素
                    if ((e = p.next) == null) {
                        // 如果下一个元素为null，new一个node对象放在当前节点上
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 当前循环次数大于树化的阈值，则进行树化
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                                            // 如果当前坐标下链表元素不为空，则比较hash值和key是否相等 如果相等则跳出循环
                        break;
                    p = e;// 都不满足，则e赋给p，其实是为了下次循环
                }
            }
            if (e != null) { // existing mapping for key
                // 在上面的流程中如果exist有了值，说明当前key在map已经有了
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
					// oldValue为null或者onlyIfAbsent为false时更新值为value。
                    e.value = value;
                // 回调方法
                afterNodeAccess(e);
				// 返回oldValue
                return oldValue;
            }
        }
        // 如果走到这里说明是插入了这个key、value
        ++modCount; // 更新modCount
        if (++size > threshold)
            resize(); // 去判断是否需要扩容 需要的话就去resize扩容
        afterNodeInsertion(evict);
        return null; // return null
    }
```

可以看到putVal方法流程是：

1. 如果当前tab数组没有初始化，先调用resize()方法去初始化数组，数组长度赋给n。

2. 通过tab[(n-1) & hash] 寻找数组位置，

   - 如果当前数组位置对象为null，则new一个Node放到当前数组的位置，走++modCount、判断并且resize扩容、返回null。

   -  把当前数组下标位置的元素赋给p变量，如果p不为null（说明通过该key对应的hash值映射到的数组下标处的元素不为空），那么就看是否存在key相同的元素（映射到同一下标不代表key相同）：
     -  如果满足判断条件：`(k = p.key) == key || (key != null && key.equals(k)))`这个判断条件，则p赋值给e，代表exist就是数组tab上的第一个元素。
     - 没有满足条件看p是否为TreeNode的一个实例，这里如果p是TreeNode，说明该hashMap已经树化，此时调用TreeNode的putTreeVal方法去往树中添加当前的key，value。并把这个方法的返回值赋给e。（这个方法先不深究，这里注意可能返回null）。
     -  如果也不是TreeNode的一个实例，则沿当前位置像遍历链表：一个for循环，维护了计数变量bitCount。
       -  将p.next赋值给e，如果这个e为null，则new一个node对象放在p.next的位置上，并且判断当前的bitCount是否大于等于树化的阈值8-1=7，注意bitCount是从0开始的，如果达到，要调用treeIfBin方法进行树化操作，并跳出循环。（调用树化的方法并不一定会树化，因为内部还判断了数组长度，如果未达到数组长度的树化阈值，会先去调用resize方法扩容）
       -  如果p.next不为null，那么再次判断条件``(k = e.key) == key || (key != null && key.equals(k)))`，即判断p.next的key是否和要put的key相等，如果相等，则跳出循环。
       - 如果都不满足（p.next不为null且与键值与key不相等），则将e赋给p，进行下次循环。
     -  判断e这个变量，即当前map中存在与当前要put的key相等的键值对，如果oldValue为null或者onlyIfAbsent是false，则覆盖oldValue并直接返回oldValue。（注意这步不会修改modCount）

3. modCount++、size++，这里判断了是否需要去扩容，如果超过了负载因子和长度的乘积这个阈值，则调用resize()方法进行扩容。返回null。

#### resize()方法

可以在putVal方法的源码中看到resize()方法在多个地方被调用，来看看这个方法的源码：

```java
 /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table; // table数组赋值oldTab变量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;// oldCap为数组长度
        int oldThr = threshold;// 老的阈值
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 如果数组已经初始化
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 如果当前数组长度已经超过最大的容量限制
                threshold = Integer.MAX_VALUE;// 阈值设置外Integer.MAX_VALUE
                return oldTab;// 返回老数组
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 如果老数组扩容2倍之后小于最大容量限制并且老数组长度大于等于默认初始化容量
                newThr = oldThr << 1; // double threshold 直接将阈值也乘2
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 如果当前数组长度为0，但是阈值已经初始化了，那么直接将oldThr赋给newCap变量。这里场景即运行了构造函数但是没有进行put操作，那时的threshold值会为tableSizeFor(initCap)方法的结果。
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 如果oldThr、oldCap都为0 赋值newCap、newThr都为默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            // 如果newThr为0，则通过现在的newCap和负载因子计算newThr
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;// 赋值threshold
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // 初始化新的node数组
        table = newTab;// newTab赋给table
        if (oldTab != null) {
            // 如果oldTab中有数据
            for (int j = 0; j < oldCap; ++j) {
                // 遍历oldTab
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // 如果oldTab[j]元素不为null
                    oldTab[j] = null; // oldTab[j] 置为 null
                    if (e.next == null)
                        // 如果当前节点的next无元素，则将e（oldTab[i]的值）放在newTab[e.hash & (newCap - 1)]的位置。这个e.hash & newCap - 1使得resize之后分散的更加均匀。（1.8的一个优化）
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 如果为树节点，调用TreeNode的split方法去分离扩容
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // resize处理链表，这里注释可以看到弃用了1.7的头插法，避免形成环。
                        // 链表可能会被拆分，因为容量扩大，可能要移动到oldCap +当前索引值处。
                        // loHead存储不需要移动到新的下标处的node
                        Node<K,V> loHead = null, loTail = null;
                        // hiHead存储需要移动到新的下标处的node
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        // 返回newTab
        return newTab;
    }
```

从注释上可以看到，这个方法不仅仅承担着两倍扩容的作用，也负责初始化table数组的作用。

resize方法的流程：

一、初始化变量阶段

1. 当前tab数组赋给oldTab，当前tab数组长度赋给oldCap，当前threshold赋给oldThr。初始化newCap、newThr为0
2. 判断如果oldCap>0，即当前数组长度大于0
   - 如果oldCap大于等于最大的Cap值，那么设置threshold为Integer.MAX_VALUE，直接返回oldTab。
   -  赋值newCap为oldCap的2倍，如果newCap小于最大容量并且oldCap大于等于默认初始化容量，则直接将当前的threshold扩大2倍。
3. oldCap为0，如果oldThr>0，则直接将newCap设置为oldThr容量。（这里说明下，如果当前数组长度为0，但是阈值已经初始化了，那么直接将oldThr赋给newCap变量。这里场景即运行了构造函数但是没有进行put操作，那时的threshold值会为tableSizeFor(initCap)方法的结果）
4. 如果oldCap和oldThr都为0，则初始化newCap为默认大小，newThr为默认Cap大小 * 默认负载因子。
5. 如果上述流程中newThr为0，那么根据newCap和当前的负载因子去计算threshold值赋给newThr。

二、数据迁移阶段

如果oldTab不为null，那么需要进行数据到扩容之后的数组的映射。

1. 循环老数组
   - 如果当前下标元素old[j]不为null，则进行下面的操作：
     - oldTab[j]元素赋给变量e，oldTab[j]赋值null。
     -  如果e.next为null，则说明直接将当前元素迁移到新的坐标即可。这里定位新的坐标是`e.hash & (newCap - 1)` 即用hash值和newCap-1去做与操作，1.8中的这个操作很妙的是newCap绝对是2的次幂，具体的可以参考上面提到的扰动函数的位置的说明。
     - 如果e.next不为null并且当前e是一个TreeNode，那么调用TreeNode的split方法去处理数组。这里红黑树的方法不去深究。这个操作中其实也有可能去做树的退化操作。因为和1.7一样，扩容原来在一条链上的元素，在扩容之后可能不在一条链上，这里也一样，如果链的长度达到了树退化的阈值，那么这里的要做树退化为链表的操作。
     - 如果e.next不为null并且是一个链表结构，那么这里会将e所在的链表元素重新计算新的下标值，映射到新的数组上去，刚才也提到了链表可能会被拆分，因为容量扩大，可能要移动到[oldCap +当前索引值]处。(**可以看到注释说明了保证了链的顺序，这里弃用了1.7中的头插法避免出现环**)
     - 继续循环oldCap
   - 最后返回newTab



## 1.4 get方法

我们来看看也是经常使用的HashMap的get方法:

```java
/**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

看注释有一点是：如果get方法返回了null，那么不一定是这个map中不包含当前传入的key，也有可能是在map中key映射了null值value，并可以通过containsKey方法去区分这两种情况。

可以看到get方法是将hash值传入getNode方法的，下面看看getNode方法的源码：

```java

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab;// tab数组 
        Node<K,V> first, e; 
        int n; 
        K k;
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
         // 如果当前的数组不为null，并且数组长度大于0，并且通过(n-1)&hash求得数组下标位置的值不为null
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                // 如果当前位置的第一个node节点的key与相等，直接返回当前节点
                return first;
            // 如果first节点的key与当前传入的key不相等
            if ((e = first.next) != null) {
               // 如果first还有下一个节点
                if (first instanceof TreeNode)
                    // 判断当前Node是否TreeNode，如果是则调用树的getTreeNode方法获取value
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 非树节点，其实就是循环链表，挨个对比key是否相等，直到链表尾部。
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

getNode的流程比较简单，注释上写的流程这里不去做过多解释。

## 1.4 remove方法

来看看remove方法的代码：

```java
  /**
     * Removes the mapping for the specified key from this map if present.
     *
     * @param  key key whose mapping is to be removed from the map
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
```

这里注意返回null可以代表map中没有这个映射，或map中这个key映射的value是null。

同样这里也是把hashCode计算好之后传入一个removeNode方法中：

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab; 
    Node<K,V> p; 
    int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
        // 和getNode一样的套路，这里如果table数组不为null并且长度大于0并且hash值定位到的数组坐标值不为空
            Node<K,V> node = null, e; // 定义node变量
        K k; V v;
            if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
                node = p;// 如果当前坐标的第一个node的key与传入的key相等，p赋值node
            else if ((e = p.next) != null) {
                // 如果不相等且还有下边的节点
                if (p instanceof TreeNode)
                    // 如果是树化的节点，调用树化的getTreeNode方法获取node
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    // 不是树化的节点，则遍历链表找对应的键值对
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;// 注意这里遍历时更新了p变量，方便在下面删除时使用
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                // 如果node不为null，并且matchValue的值和当前value的值设置一致（不是值一致，而是这个判断）
                if (node instanceof TreeNode)
                    // 树化的node节点调用TreeNode的removeTreeNode方法删除
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    // 不是树化，则判断node是否为p节点，即数组的第一个节点，如果是则将node的第一个节点在数组位置上删除即可
                    tab[index] = node.next;
                else
                    // 不是树化并且不是第一个节点，则将当前node的next置为p.next即可 即删除node节点
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

可以看到删除时的操作和getNode方法大同小异，只不过在找到对应的key对应的node节点之后，去做了对node节点的删除逻辑而已。



# 彩蛋

上面有提到在1.7中resize时采用的头插法来转移之前数组上的链表，下面这个图是在1.7中扩容的简单的说明：

图片来源： https://blog.csdn.net/pange1991/article/details/82347284 

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/hashMap_1.7_resize.png)

而这个在并发的场景下可能形成环，耗子叔的博客也介绍了很详细： https://coolshell.cn/articles/9606.html 

# 总结

本文简单介绍了HashMap中的成员变量、构造函数、改查删方法的源码，还对里面一些hash取值、定位数组下标的方法做了简单介绍，其中对1.8之后的红黑树没有深究，之后可能在后面的拾遗过程中介绍红黑树的用法。这里知道在链表长度过大时，红黑树能提高对应的效率即可。

关于HashMap的结构和容量总的来说：

- HashMap 底层数据结构在JDK1.7之前是由数组+链表组成的，1.8之后又加入了红黑树；链表长度小于8的时候，发生Hash冲突后会增加链表的长度，当链表长度大于8的时候，会先判读数组的容量，如果容量小于64会先扩容（原因是数组容量越小，越容易发生碰撞，因此当容量过小的时候，首先要考虑的是扩容），如果容量大于64，则会将链表转化成红黑树以提升效率。
- hashMap 的容量是2的n次幂，无论在初始化的时候传入的初始容量是多少，最终都会转化成2的n次幂，这样做的原因是为了在取模运算的时候可以使用&运算符，而不是%取余，可以极大的提上效率，同时也降低hash碰撞的概率。



## 参考文章

 https://juejin.im/post/5c8f461c5188252da90125ba 

 https://www.cnblogs.com/ysocean/p/9054804.html 

 https://blog.csdn.net/pange1991/article/details/82347284 