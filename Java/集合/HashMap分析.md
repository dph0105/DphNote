## 1. HashMap简介

HashMap实现了Map，它是一个用于存储Key-Value键值对的集合，每一个键值对也叫做**Node**，继承`Map.Entry`。这些个键值对（Node）分散存储在一个数组当中，这个数组就是HashMap的主干。HashMap允许null键与null值，并且它对与Node的顺序不做保证。

### 1.1 HashMap的成员变量

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    private static final long serialVersionUID = 362498820763181265L;

    /**
     * HashMap数组的默认初始化容量为16
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * HashMap数组的最大容量为2^30
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 默认loadFactor（负载因子）为0.75，即当HashMap的元素数量大于容量*0.75时，会发生扩容。
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 默认当链表的长度大于8时，链表会转化为红黑树。(需要hashmap元素数量大于64)
     */
    static final int TREEIFY_THRESHOLD = 8;
    
    /**
     * 默认当树的长度小于6时，会转化为链表，前提是红黑树结构
     */
    static final int UNTREEIFY_THRESHOLD = 6;
    
    /**
     * 默认的当HashMap的长度大于等于64时，链表才能转化为树
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
    
	//存储元素的数组，transient关键字表示该属性不能被序列化
	transient Node<K,V>[] table;

    //将数据转换成set的另一种存储形式，这个变量主要用于迭代功能。
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * 元素数量
     */
    transient int size;

    /**
     * 统计该map修改的次数
     */
    transient int modCount;

    /**
     * 临界值，也就是元素数量达到临界值时，会对数组进行扩容。
     */
    int threshold;
	
	/**
     * 实际的loadFactor（负载因子）即 当元素数量 大于 数组容量 * loadFactor时，进行扩容
     * 比如 数组大小是16 loadFactor默认0.75 当HashMap的元素数量大于12时，就进行扩容
     */
    final float loadFactor;
}

```



### 1.2 HashMap的构造方法

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
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    
```

一般我们使用默认的无参构造方法创建HashMap对象，这会创建一个数组大小为16，loadFactor为0.75的HashMap，由于HashMap使用懒加载，这里并没有直接设置值，可以在put方法中看到。

当我们需要自己直接指定HashMap的大小时，可以通过有参的构造方法创建HashMap对象，`HashMap(int initialCapacity, float loadFactor)`中，通过`tableSizeFor(initialCapacity)`计算出threshold的值：

```java
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

该函数的作用是返回不小于给定容量大小的2的幂，即 3 -> 4; 6 -> 8; 10 ->16;

**threshold表示的是HashMap的元素数量大于此值时，会发生扩容。当我们直接指定HashMap的初始容量时，会直接计算出该值**。

### 1.3 HashMap的hash函数

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

hash值得计算是通过 key的hashCode() 方法的高 16 位异或低 16 位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度，功效和质量来考虑的，减少系统的开销，也不会造成因为高位没有参与下标的计算，从而引起的碰撞。

因为hash值是一个int值，范围为带符号的 -2^31 ~ 2^31-1，但是我们的数组一般长度并没有这么长，初始默认长度只有16，所以当我们计算元素在数组中的位置时，我们会想到 hash% 数组长度，但是这样的效率太低了，所以这里使用 **(数组长度 - 1) & hash**。而这里也解释了**为什么需要数组的长度要取2的整数幂**，因为这样（数组长度-1）正好相当于一个“低位掩码”。“与” 操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。

那么问题就来了，如果使用原本的hash值，一般只有低位的值得到了计算，而高位的值没有参与运算，这样会增加了碰撞的概率，例如：可能出现高位不相同，而低位正好相同的hash值

所以HashMap中的hash()函数，使key的hashCode值得高16位与低16位进行异或运算，既增加了随机性，并且混合后的值也变相的保留了高位的特征。



### 1.4 HashMap的扩容函数resize()

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {//oldCap大于0说明不是第一次，属于扩容的情况
            if (oldCap >= MAXIMUM_CAPACITY) {
                //如果当前数组容量大于等于最大容量，设置threshold为Int最大值
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //设置新的数组容量为当前的两倍，新的扩容阈值也为当前的两倍
                newThr = oldThr << 1;
        }
        //剩下的情况就是第一次初始化
        else if (oldThr > 0)
            //这种情况可能为构造方法为有参构造 例如：new HashMap(32)\new HashMap(32,0.75)
            //因为是第一次初始化，所以直接设置new容量为旧的threashold
            //threshold为2的幂，所以数组的容量也为2的幂
            newCap = oldThr;
        else {
            //new HashMap()，使用无参构造器创建，并且第一次加载
            //newCap 默认被设置为16
            //newThr 默认被设置为 16 * 0.75
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            //初始化时，newThr最终为数组容量*负载因子
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //设置threshold为newThr
        threshold = newThr;
        //创建数组
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //如果有旧数据，则要转移数据了
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        //如果数组j位置的元素没有下一个，即没有hash碰撞，
                        //通过e.hash & (newCap - 1)计算它在新数组中的位置
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //如果桶为红黑树结构，转移红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else {
                        //如果桶为链表结构，转移链表中的元素
                        Node<K,V> loHead = null, loTail = null;
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
        return newTab;
    }

```

在resize函数中，转移元素位置时使用 `e.hash & oldCap`是否为0将一个链表分为两个链表，一个是**原来的位置**，一个是**原来的位置+原来的容量**。

为什么使用oldCap呢？扩容时，数组的容量会变为原先的2倍。假设原来为16，那么扩容后为32。假设此时一个hash值为5和hash值为21的两个元素，根据 `hash & cap-1` ，扩容前他们都处于 table[5] 这里，而扩容后，hash值为5的元素没有变，hash值为21的元素根据公式则应该转移到 table[21] 处，转移的元素的新的位置就是 5 + 16，即原来的位置+原来的容量。那么我们可以知道0-15范围内hash值的元素不用转移，16-31范围内的就需要转移。即hash值低位为00000-01111的不用转移，而**1**0000-**1**1111范围的需要转移。则我们只需要判断 e.hash & oldCap 是否为0，即可判断该元素是否需要转移。



### 1.5 HashMap插入键值对

HashMap使用put方法插入键值对：

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //第一次使用put时，HashMap是没有初始化的，这里直接进行扩容，即初始化
            n = (tab = resize()).length;
        //直接根据(n - 1) & hash 即 数组长度对hash值取余得到数组的位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            //该位置元素为null，则直接放入新的元素
            tab[i] = newNode(hash, key, value, null);
        else { //如果该位置不为null
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
             	//若put的元素和该位置的相同
                e = p;
            else if (p instanceof TreeNode)
                //如果不是同一个元素，并且桶是红黑树结构，增加一个树节点
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //如果不是同一个Node，并且桶是链表结构，遍历该链表
                for (int binCount = 0; ; ++binCount) {
                    //遍历到该链表的末尾元素了，说明put的元素没有添加过，增加一个链表节点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            //如果链表的当前长度大于等于 7 ，转化为红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果在链表中发现有一个相同的，那么直接break
                    //因为e都等于 p.next了所以直接break，即e此时就是要put的元素
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果e不是null，则证明put的元素是一个已存在的，替换value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                //这里直接return
                return oldValue;
            }
        }
        //增加HashMap改变次数
        ++modCount;
        //增加HashMap大小，并且大小大于阈值时，扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```



### 1.6 HashMap取值

```java
   public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            //根据(n - 1) & hash 找到数组中的位置
            (first = tab[(n - 1) & hash]) != null) {
            //判断是否就是数组中的那个元素
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                //查找链表中的元素
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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



## 2. HashMap的一些问题

### 2.1 为什么hash(key)中，高16位与低16位使用异或运算符？

保证了对象的 hashCode 的 32 位值只要有一位发生改变，整个 hash() 返回值就会改变。尽可能的减少碰撞。

### 2.2 为什么链表过长要转化为红黑树？

​	之所以选择红黑树是为了解决二叉查找树的缺陷，二叉查找树在特殊情况下会变成一条线性结构（这就跟原来使用链表结构一样了，造成很深的问题），遍历查找会非常慢。而红黑树在插入新数据后可能需要通过左旋，右旋、变色这些操作来保持平衡，引入红黑树就是为了查找数据快，解决链表查询深度的问题。

#### 2.2.1 为什么不直接使用红黑树

​	我们知道红黑树属于平衡二叉树，但是为了保持“平衡”是需要付出代价的，但是该代价所损耗的资源要比遍历线性链表要少，所以当长度大于8的时候，会使用红黑树，如果链表长度很短的话，根本不需要引入红黑树，引入反而会慢。

### 2.3 HashMap在多线程环境中会遇到什么问题？

<https://juejin.im/post/5c8910286fb9a049ad77e9a3#heading-3>



































