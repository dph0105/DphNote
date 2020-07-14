## 1. HashMap简介

HashMap实现了Map，它是一个用于存储Key-Value键值对的集合，每一个键值对也叫做**Node**，继承`Map.Entry`。这些个键值对（Node）分散存储在一个数组当中，这个数组就是HashMap的主干。HashMap允许null键与null值，并且它对与Node的顺序不做保证。

### 1.1 HashMap的成员变量

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    private static final long serialVersionUID = 362498820763181265L;

    /**
     * HashMap的默认大小为16
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * HashMap的最大容量为2^30
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 默认loadFactor（负载因子）为0.75，即当HashMap的长度大于容量*0.75时，会发生扩容。
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 默认当链表的长度大于8时，链表会转化为红黑树。(需要hashmap长度大于64)
     */
    static final int TREEIFY_THRESHOLD = 8;
    
    /**
     * 默认当树的长度小于6时，会转化为链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;
    
    /**
     * 默认的当HashMap的长度大于等于64时，链表才能转化为树
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
	
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

一般我们使用默认的无参构造方法创建HashMap对象，这会创建一个