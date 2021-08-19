# JUC源码深入再深入

# ConcurrentHashMap源码

ai![image-20210325164441884](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210325164441884.png)

大致的节点构造。

FWD节点是用与扩容相关的。

TreeBin节点有两个结构，红黑树和链表

支持并发扩容，也就是多线程一起扩容。迁移操作一定是从桶位的最后一点点向上迁移的。解决冲突。

## 成员变量源码分析

```java
  /**
       散列表最大长度，也就是2的30次方
     */
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
      默认初始化容量16
     */
    private static final int DEFAULT_CAPACITY = 16;

    /**
       最大可能(非二次幂)数组大小，需要通过toArray和相关方法
     */
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * The default concurrency level for this table. Unused but
     * defined for compatibility with previous versions of this class.
        默认当前散列表的并发数。未使用，但是为当前类的早期版本兼容而定义
        compatibility兼容  previous兼容 1.8中不代表并发级别，1.7中代表的并发级别
        1.8的并发级别是取决于散列表的长度
     */
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    /**
       负载因子0.75
       */
    private static final float LOAD_FACTOR = 0.75f;

    /**
       树化阈值
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
      将树转为链表的阈值
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
    最小达到树化的表容量
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * Minimum number of rebinnings per transfer step. Ranges are
     * subdivided to allow multiple resizer threads.  This value
     * serves as a lower bound to avoid resizers encountering
     * excessive memory contention.  The value should be at least
     * DEFAULT_CAPACITY.
      线程迁移数据最小步长，也就是桶位的跨度。一个线程不允许超过该长度去棒迁移。
      
     */
    private static final int MIN_TRANSFER_STRIDE = 16;

    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
      扩容相关，计算扩容时生成的一个标识戳。
     */
    private static int RESIZE_STAMP_BITS = 16;

    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
       65535 标识并发扩容最多线程数。
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    /**
     * The bit shift for recording size stamp in sizeCtl.
       扩容相关
       sizeCtl中记录大小戳的比特变化
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    /*
     * Encodings for Node hash fields. See above for explanation.
      movend：表示当前节点为FWD节点
      TreeBin：当node节点的hash值为-2，表示已经树化，并且该TreeBin对象代理操作红黑树
      Reserved：临时保留的hash？
      HASH_BITS：普通节点的hash
     */
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
   //0x7fffffff = 01111111111111111111111111111111 
   //十进制2 147 483 648 20多亿的数据，同时也是Integer.MAX_VALUE值
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

    /** Number of CPUS, to place bounds on some sizings 
    CPU核心数
    */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /** 
    For serialization compatibility. 对于兼容序列化。
    */
    private static final ObjectStreamField[] serialPersistentFields = {
        new ObjectStreamField("segments", Segment[].class),
        new ObjectStreamField("segmentMask", Integer.TYPE),
        new ObjectStreamField("segmentShift", Integer.TYPE)
    };
```

```java
 /**
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
       散列表  在第一次插入的时候懒加载
       大小总是2的次方。   directly直接  Accessed访问  根据迭代器
     */
    transient volatile Node<K,V>[] table;

    /**
     * The next table to use; non-null only while resizing.
       下一个表去使用，只有当扩容的时候不为Null
     */
    private transient volatile Node<K,V>[] nextTable;

    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
      基础数量值，主要在没有竞争的时候使用，当LongAdder对应的Cell处于加锁，则加入到baseCount
     */
    private transient volatile long baseCount;

    /**
       
     * Table initialization and resizing control.  When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads).  Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
      sizeCtl < 0
      表的初始化和扩容控制，当为负数的时候，-1表示正在初始化，
      小于-1表示正在进行扩容 低16位表示1+正在帮助扩容的线程数，高16位表示扩容的标识戳。
      sizeCtl = 0 ，表示创建table数组的时候，使用默认初始容量16
      sizeCtl > 0
      当table未初始化
      表示初始化大小
      当表已经初始化
      下一次要调整表大小的临界值，到达该值进行扩容散列表
      
     */
    private transient volatile int sizeCtl;

    /**
     * The next table index (plus one) to split while resizing.
       扩容过程中，记录当前进度。所有线程都需要从transferIndex中分配区间任务，去执行自己的任务。
     */
    private transient volatile int transferIndex;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
       调整大小或者创建单元格的时候使用的自旋锁，通过CAS锁定
       0无锁，1有锁。
     */
    private transient volatile int cellsBusy;

    /**
     * Table of counter cells. When non-null, size is a power of 2.
       表的计算器单元格，当不为空的时候，大小是2的次方。
     */
    private transient volatile CounterCell[] counterCells;

    // views
    private transient KeySetView<K,V> keySet; //
    private transient ValuesView<K,V> values; 
    private transient EntrySetView<K,V> entrySet;
```

```java
  // Unsafe mechanics
    private static final sun.misc.Unsafe U;
   //获取到SIZECTL在ConcurrentHashMap中的内存偏移量，根据内存偏移量和期待值和新值来CAS
    private static final long SIZECTL;
    private static final long TRANSFERINDEX;
    private static final long BASECOUNT;
    private static final long CELLSBUSY;
    private static final long CELLVALUE;
   //表示数组第一个元素的偏移地址
    private static final long ABASE;
    private static final int ASHIFT;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            SIZECTL = U.objectFieldOffset
                (k.getDeclaredField("sizeCtl"));
            TRANSFERINDEX = U.objectFieldOffset
                (k.getDeclaredField("transferIndex"));
            BASECOUNT = U.objectFieldOffset
                (k.getDeclaredField("baseCount"));
            //cellsBusy的偏移量
            CELLSBUSY = U.objectFieldOffset
                (k.getDeclaredField("cellsBusy"));
            Class<?> ck = CounterCell.class;
            //每个Cell数组元素的大小值
            CELLVALUE = U.objectFieldOffset
                (ck.getDeclaredField("value"));
            Class<?> ak = Node[].class;
            ABASE = U.arrayBaseOffset(ak);
            //表示数组单元所占用空间大小，sacle表示Node[]数组中每一个单元的所占用空间大小。    
            //通过上面的ABASE第一个元素的偏移地址，加每一个单元的所占用空间大小，就可以找到每一个对应的Node下标元素的位置。因为每个Node元素大小都是一致的。
            int scale = U.arrayIndexScale(ak);
            //保证该值是2的次方数。
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
        
      /**
   numberOfLeadingZeros该方法是返回当前数值从高位到低位有连续多少个0
  比如上面的是int数组，那么sacle的值就是4
  numberOfLeadingZeros(4) = 29   ASHIFT = 2；
  那么这个2能干什么呢？ 通过ASHIFT来访问到数组中的某个下标的内存地址，因为scale是局部变量，静态代码块执行完了以后就没作用了。如果正常访问的话可以使用ABASE + 4 * 4
  表示我想要访问下标为4的int数组 偏移地址+下标*大小
  jdk使用ASHIFT方式实现了访问数组下标的内存位置。
  ASHIFT = 2
  ABASE + （4 << ASHIFT） 相当于4左移2位，那么也就是4 * 4  用于寻址。
       
     */
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

## 内部类源码分析

```java
  static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; //经过扰动运算之后key的hash值 不能修改
        final K key; //key 不能修改
        volatile V val; //value，可以修改 可见性
        volatile Node<K,V> next; //指向下一个节点，可以修改

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
        public final String toString(){ return key + "=" + val; }
        public final V setValue(V value) {
            throw new UnsupportedOperationException();
        }

        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```

```java
/**
 该节点在扩容的时候使用，里面的nextTable是正在进行扩容之后的新表。当写线程在扩容之前的表写的时候会被让其一起协助扩容，当读线程读的时候通过ForwardingNode节点的nextTable表的引用去新表上读数据。
*/
static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }

        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }
```

```java
/**
parent是树的父亲节点
left左子节点 right右子节点 red是否是红色的，false为黑色
prev是前一个结点  所以TreeNode相当于既是红黑树也是双向链表。
*/
static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }

        Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null);
        }

        /**
         * Returns the TreeNode (or null if not found) for the given key
         * starting at given root.
         */
        final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
            if (k != null) {
                TreeNode<K,V> p = this;
                do  {
                    int ph, dir; K pk; TreeNode<K,V> q;
                    TreeNode<K,V> pl = p.left, pr = p.right;
                    if ((ph = p.hash) > h)
                        p = pl;
                    else if (ph < h)
                        p = pr;
                    else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                        return p;
                    else if (pl == null)
                        p = pr;
                    else if (pr == null)
                        p = pl;
                    else if ((kc != null ||
                              (kc = comparableClassFor(k)) != null) &&
                             (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.findTreeNode(h, k, kc)) != null)
                        return q;
                    else
                        p = pl;
                } while (p != null);
            }
            return null;
        }
    }
```



## 小函数工具方法源码分解

### spread(hash)源码分析

将传入的hash值无符号右移16位，然后和当前hash值进行异或，然后再&0x7fffffff

意思就是低16位和高16位进行异或，然后再进行去符号。扰动函数，为了让找桶的位置更加均匀。

```java
static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

### tabAt(tab,index)源码分析

根据index索引获取Node数组中对应的下标节点。为什么使用该方式取呢？为了保证线程可见性获取。去主存中读取该值。

```java
 static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```

### casTabAt(tab,index,c,v)源码分析

替换对应Index下标的Node节点，尝试使用CAS替换，c是期待值，v是新值

```java
  static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```

### setTabAt(tab,index,x)源码分析

```java
  static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

向对应的index位置添加Node节点

### resizeStamp(int n)源码分析

得到扩容的标识戳，线程必须获取。标识戳必须一致的情况下才可以参与扩容。

numberOfLeadingZeros(16)  

0000 0000 0001 1011

1000 0000 0000 0000

1000 0000 0001 1011 

如果是16-32，都会拿到1000 0000 0001 1011 这个戳

```java
static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```

### tableSizeFor(int c)源码解析

返回>=c的最小的2进制大小的数值，如果传入的是9则返回16，如果传入的是7则返回8

通过一堆右移和或运算。

相当于目的是为了让每个Bit位的值都为1，那么就是二进制，或运算只要有一个1就为1

然后n+1 比如说是1111 + 1  1 0000 那么就是2的次方数了

```java
 private static final int tableSizeFor(int c) {
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

## 构造方法源码解析

无参构造方法，都采取默认值。

```java
  public ConcurrentHashMap() {
    }
```

指定初始容量的构造方法，比如传入的值为15，那么

tableSizeFor(15 + 15>>>1 + 1)，

1111 + 0111 + 1 =  15 + 7 + 1  =  23，然后经过tableSizeFor方法会得到sizeCtl为32

```java
 public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```

将map中的数据复制到当前新Map

```java
  public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }
```

指定容量和负载因子

```java
 public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }
```

指定初始容量，负载因为，并发级别

loadFactor和concurrencyLevel只是用于初始化，不会被设置。loadFactor是被final修饰的。

```java
   public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

## 核心方法解析

### putVal(k,v,onlyIfAbsent)源码解析

```java
  public V put(K key, V value) {
        return putVal(key, value, false);
    }
```

```java
   public V putIfAbsent(K key, V value) {
        return putVal(key, value, true);
    }
/**
 onlyIfAbsent 如果存在则返回，否则插入。put默认为false
 putIfAbsent 为true
*/
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode()); //扰动hash
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) { 
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0) //初始化hash
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { //对应的桶位无元素，直接cas放入。
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //hash值为-1,该桶位正在进行数据迁移，帮助迁移
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //说明当前桶位就是普通链表。
                        if (fh >= 0) {
                            //当前插入元素与链表中的key都不相等，追加到链表末尾，表示链表长度，相等的话表示链表的冲突位置(bincount-1)
                   
                            binCount = 1;
                            //循环遍历该链表，如果key相等，则替换，如果是onlyAbsent为true，则不替换，否则插入到链表的尾部。
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //单向链表，插入尾部。
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            //强制设置binCount为2
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
       //统计table表中的数据
       //判断是否达到扩容阈值标准，触发扩容。
        addCount(1L, binCount);
        return null;
    }
```

```java
 /**
 初始化散列表，通过cas将SizeCtl的值设置为-1，告诉其他线程正在进行初始化，其他线程会执行Thread.Yield()方法让出CPU使用权，使其正在初始化的线程尽快完成初始化。
 初始化完成后重新将SizeCtl设置为大于0的值
 */
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            //将相当于就是将sizeCtl的值改为-1，SIZECTL是sizectl在内存的偏移量。
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    //此处如果其他线程已经Break，另外线程刚好进入该方法，就会再次将sizeCtl改为-1，所以再次增加判断防止丢失数据。
                    if ((tab = table) == null || tab.length == 0) {
                        //sc和sizeCtl此时无关系，sizeCtl是-1，而sc不是
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //两种情况，第一次就是初始化后的扩容阈值
                    //第二种就是第其他线程再次进来通过if判断发现表不为空，那么将sc又恢复给sizeCtl，相当于还是释放锁，和解锁。
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

### addCount源码方法分析

```java
   private final void addCount(long x, int check) {
        //as 表示 LongAdder.cells
        //b 表示LongAdder.base
        //s 表示当前map.table中元素的数量
        CounterCell[] as; long b, s;
        //条件一：true->表示cells已经初始化了，当前线程应该去使用hash寻址找到合适的cell 去累加数据
        //       false->表示当前线程应该将数据累加到 base
        //条件二：false->表示写base成功，数据累加到base中了，当前竞争不激烈，不需要创建cells
        //       true->表示写base失败，与其他线程在base上发生了竞争，当前线程应该去尝试创建cells。
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            //有几种情况进入到if块中？
            //1.true->表示cells已经初始化了，当前线程应该去使用hash寻址找到合适的cell 去累加数据
            //2.true->表示写base失败，与其他线程在base上发生了竞争，当前线程应该去尝试创建cells。

            //a 表示当前线程hash寻址命中的cell
            CounterCell a;
            //v 表示当前线程写cell时的期望值
            long v;
            //m 表示当前cells数组的长度
            int m;
            //true -> 未竞争  false->发生竞争
            boolean uncontended = true;


            //条件一：as == null || (m = as.length - 1) < 0
            //true-> 表示当前线程是通过 写base竞争失败 然后进入的if块，就需要调用fullAddCount方法去扩容 或者 重试.. LongAdder.longAccumulate
            //条件二：a = as[ThreadLocalRandom.getProbe() & m]) == null   前置条件：cells已经初始化了
            //true->表示当前线程命中的cell表格是个空，需要当前线程进入fullAddCount方法去初始化 cell，放入当前位置.
            //条件三：!(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x)
            //      false->取反得到false，表示当前线程使用cas方式更新当前命中的cell成功
            //      true->取反得到true,表示当前线程使用cas方式更新当前命中的cell失败，需要进入fullAddCount进行重试 或者 扩容 cells。
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
            ) {
                fullAddCount(x, uncontended);
                //考虑到fullAddCount里面的事情比较累，就让当前线程 不参与到 扩容相关的逻辑了，直接返回到调用点。
                return;
            }

            if (check <= 1)
                return;

            //获取当前散列表元素个数，这是一个期望值
            s = sumCount();
        }

        //表示一定是一个put操作调用的addCount
        if (check >= 0) {
            //tab 表示map.table
            //nt 表示map.nextTable
            //n 表示map.table数组的长度
            //sc 表示sizeCtl的临时值
            Node<K,V>[] tab, nt; int n, sc;


            /**
             * sizeCtl < 0
             * 1. -1 表示当前table正在初始化（有线程在创建table数组），当前线程需要自旋等待..
             * 2.表示当前table数组正在进行扩容 ,高16位表示：扩容的标识戳   低16位表示：（1 + nThread） 当前参与并发扩容的线程数量
             *
             * sizeCtl = 0，表示创建table数组时 使用DEFAULT_CAPACITY为大小
             *
             * sizeCtl > 0
             *
             * 1. 如果table未初始化，表示初始化大小
             * 2. 如果table已经初始化，表示下次扩容时的 触发条件（阈值）
             */

            //自旋
            //条件一：s >= (long)(sc = sizeCtl)
            //       true-> 1.当前sizeCtl为一个负数 表示正在扩容中..
            //              2.当前sizeCtl是一个正数，表示扩容阈值
            //       false-> 表示当前table尚未达到扩容条件
            //条件二：(tab = table) != null
            //       恒成立 true
            //条件三：(n = tab.length) < MAXIMUM_CAPACITY
            //       true->当前table长度小于最大值限制，则可以进行扩容。
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {

                //扩容批次唯一标识戳
                //16 -> 32 扩容 标识为：1000 0000 0001 1011
                int rs = resizeStamp(n);

                //条件成立：表示当前table正在扩容
                //         当前线程理论上应该协助table完成扩容
                if (sc < 0) {
                    //条件一：(sc >>> RESIZE_STAMP_SHIFT) != rs
                    //      true->说明当前线程获取到的扩容唯一标识戳 非 本批次扩容
                    //      false->说明当前线程获取到的扩容唯一标识戳 是 本批次扩容
                    //条件二： JDK1.8 中有bug jira已经提出来了 其实想表达的是 =  sc == (rs << 16 ) + 1
                    //        true-> 表示扩容完毕，当前线程不需要再参与进来了
                    //        false->扩容还在进行中，当前线程可以参与
                    //条件三：JDK1.8 中有bug jira已经提出来了 其实想表达的是 = sc == (rs<<16) + MAX_RESIZERS
                    //        true-> 表示当前参与并发扩容的线程达到了最大值 65535 - 1
                    //        false->表示当前线程可以参与进来
                    //条件四：(nt = nextTable) == null
                    //        true->表示本次扩容结束
                    //        false->扩容正在进行中
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;

                    //前置条件：当前table正在执行扩容中.. 当前线程有机会参与进扩容。
                    //条件成立：说明当前线程成功参与到扩容任务中，并且将sc低16位值加1，表示多了一个线程参与工作
                    //条件失败：1.当前有很多线程都在此处尝试修改sizeCtl，有其它一个线程修改成功了，导致你的sc期望值与内存中的值不一致 修改失败
                    //        2.transfer 任务内部的线程也修改了sizeCtl。
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //协助扩容线程，持有nextTable参数
                        transfer(tab, nt);
                }
                //1000 0000 0001 1011 0000 0000 0000 0000 +2 => 1000 0000 0001 1011 0000 0000 0000 0010
                //条件成立，说明当前线程是触发扩容的第一个线程，在transfer方法需要做一些扩容准备工作
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    //触发扩容条件的线程 不持有nextTable
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

### transfer源码分解

<img src="https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210330172647489.png" alt="image-20210330172647489" style="zoom:67%;" />

```java
 /**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        //n 表示扩容之前table数组的长度
        //stride 表示分配给线程任务的步长
        int n = tab.length, stride;
        //方便讲解源码  stride 固定为 16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range


        //条件成立：表示当前线程为触发本次扩容的线程，需要做一些扩容准备工作
        //条件不成立：表示当前线程是协助扩容的线程..
        if (nextTab == null) {            // initiating
            try {
                //创建了一个比扩容之前大一倍的table
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            //赋值给对象属性 nextTable ，方便协助扩容线程 拿到新表
            nextTable = nextTab;
            //记录迁移数据整体位置的一个标记。index计数是从1开始计算的。
            transferIndex = n;
        }

        //表示新数组的长度
        int nextn = nextTab.length;
        //fwd 节点，当某个桶位数据处理完毕后，将此桶位设置为fwd节点，其它写线程 或读线程看到后，会有不同逻辑。
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        //推进标记。
        boolean advance = true;
        //完成标记,只有当扩容的最后一个线程执行检查或者完成操作的时候才会生效。
        boolean finishing = false; // to ensure sweep before committing nextTab

        //i 表示分配给当前线程任务，执行到的桶位
        //bound 表示分配给当前线程任务的下界限制
        int i = 0, bound = 0;
        //自旋
        for (;;) {
            //f 桶位的头结点
            //fh 头结点的hash
            Node<K,V> f; int fh;


            /**
             * 1.给当前线程分配任务区间
             * 2.维护当前线程任务进度（i 表示当前处理的桶位）
             * 3.维护map对象全局范围内的进度
             */
            while (advance) {
                //分配任务的开始下标
                //分配任务的结束下标
                int nextIndex, nextBound;

                //CASE1:
                //条件一：--i >= bound
                //成立：表示当前线程的任务尚未完成，还有相应的区间的桶位要处理，--i 就让当前线程处理下一个 桶位.
                //不成立：表示当前线程任务已完成 或 者未分配
                if (--i >= bound || finishing)
                    advance = false;
                //CASE2:
                //前置条件：当前线程任务已完成 或 者未分配
                //条件成立：表示对象全局范围内的桶位都分配完毕了，没有区间可分配了，设置当前线程的i变量为-1 跳出循环后，执行退出迁移任务相关的程序
                //条件不成立：表示对象全局范围内的桶位尚未分配完毕，还有区间可分配
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                //CASE3:
                //前置条件：1、当前线程需要分配任务区间  2.全局范围内还有桶位尚未迁移
                //条件成立：说明给当前线程分配任务成功
                //条件失败：说明分配给当前线程失败，应该是和其它线程发生了竞争吧
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {

                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }

            //CASE1：
            //条件一：i < 0
            //成立：表示当前线程未分配到任务
            if (i < 0 || i >= n || i + n >= nextn) {
                //保存sizeCtl 的变量
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }

                //条件成立：说明设置sizeCtl 低16位  -1 成功，当前线程可以正常退出
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    //1000 0000 0001 1011 0000 0000 0000 0000
                    //条件成立：说明当前线程不是最后一个退出transfer任务的线程
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        //正常退出
                        return;

                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //前置条件：【CASE2~CASE4】 当前线程任务尚未处理完，正在进行中

            //CASE2:
            //条件成立：说明当前桶位未存放数据，只需要将此处设置为fwd节点即可。
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //CASE3:
            //条件成立：说明当前桶位已经迁移过了，当前线程不用再处理了，直接再次更新当前线程任务索引，再次处理下一个桶位 或者 其它操作
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            //CASE4:
            //前置条件：当前桶位有数据，而且node节点 不是 fwd节点，说明这些数据需要迁移。
            else {
                //sync 加锁当前桶位的头结点
                synchronized (f) {
                    //防止在你加锁头对象之前，当前桶位的头对象被其它写线程修改过，导致你目前加锁对象错误...
                    if (tabAt(tab, i) == f) {
                        //ln 表示低位链表引用
                        //hn 表示高位链表引用
                        Node<K,V> ln, hn;

                        //条件成立：表示当前桶位是链表桶位
                        if (fh >= 0) {


                            //lastRun
                            //可以获取出 当前链表 末尾连续高位不变的 node
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }

                            //条件成立：说明lastRun引用的链表为 低位链表，那么就让 ln 指向 低位链表
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            //否则，说明lastRun引用的链表为 高位链表，就让 hn 指向 高位链表
                            else {
                                hn = lastRun;
                                ln = null;
                            }



                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }



                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        //条件成立：表示当前桶位是 红黑树 代理结点TreeBin
                        else if (f instanceof TreeBin) {
                            //转换头结点为 treeBin引用 t
                            TreeBin<K,V> t = (TreeBin<K,V>)f;

                            //低位双向链表 lo 指向低位链表的头  loTail 指向低位链表的尾巴
                            TreeNode<K,V> lo = null, loTail = null;
                            //高位双向链表 lo 指向高位链表的头  loTail 指向高位链表的尾巴
                            TreeNode<K,V> hi = null, hiTail = null;


                            //lc 表示低位链表元素数量
                            //hc 表示高位链表元素数量
                            int lc = 0, hc = 0;

                            //迭代TreeBin中的双向链表，从头结点 至 尾节点
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                // h 表示循环处理当前元素的 hash
                                int h = e.hash;
                                //使用当前节点 构建出来的 新的 TreeNode
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);

                                //条件成立：表示当前循环节点 属于低位链 节点
                                if ((h & n) == 0) {
                                    //条件成立：说明当前低位链表 还没有数据
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    //说明 低位链表已经有数据了，此时当前元素 追加到 低位链表的末尾就行了
                                    else
                                        loTail.next = p;
                                    //将低位链表尾指针指向 p 节点
                                    loTail = p;
                                    ++lc;
                                }
                                //当前节点 属于 高位链 节点
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }



                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

### helpTransfer源码解析

```java
 /**
     * Helps transfer if a resize is in progress.
     */
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        //nextTab 引用的是 fwd.nextTable == map.nextTable 理论上是这样。
        //sc 保存map.sizeCtl
        Node<K,V>[] nextTab; int sc;

        //条件一：tab != null 恒成立 true
        //条件二：(f instanceof ForwardingNode) 恒成立 true
        //条件三：((ForwardingNode<K,V>)f).nextTable) != null 恒成立 true
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {

            //拿当前标的长度 获取 扩容标识戳   假设 16 -> 32 扩容：1000 0000 0001 1011
            int rs = resizeStamp(tab.length);

            //条件一：nextTab == nextTable
            //成立：表示当前扩容正在进行中
            //不成立：1.nextTable被设置为Null 了，扩容完毕后，会被设为Null
            //       2.再次出发扩容了...咱们拿到的nextTab 也已经过期了...
            //条件二：table == tab
            //成立：说明 扩容正在进行中，还未完成
            //不成立：说明扩容已经结束了，扩容结束之后，最后退出的线程 会设置 nextTable 为 table

            //条件三：(sc = sizeCtl) < 0
            //成立：说明扩容正在进行中
            //不成立：说明sizeCtl当前是一个大于0的数，此时代表下次扩容的阈值，当前扩容已经结束。
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {


                //条件一：(sc >>> RESIZE_STAMP_SHIFT) != rs
                //      true->说明当前线程获取到的扩容唯一标识戳 非 本批次扩容
                //      false->说明当前线程获取到的扩容唯一标识戳 是 本批次扩容
                //条件二： JDK1.8 中有bug jira已经提出来了 其实想表达的是 =  sc == (rs << 16 ) + 1
                //        true-> 表示扩容完毕，当前线程不需要再参与进来了
                //        false->扩容还在进行中，当前线程可以参与
                //条件三：JDK1.8 中有bug jira已经提出来了 其实想表达的是 = sc == (rs<<16) + MAX_RESIZERS
                //        true-> 表示当前参与并发扩容的线程达到了最大值 65535 - 1
                //        false->表示当前线程可以参与进来
                //条件四：transferIndex <= 0
                //      true->说明map对象全局范围内的任务已经分配完了，当前线程进去也没活干..
                //      false->还有任务可以分配。
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;


                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

### FWD的find方法源码

```java
 /**
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }

        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            //tab 一定不为空
            outer: for (Node<K,V>[] tab = nextTable;;;) {
                //n 表示为扩容而创建的 新表的长度
                //e 表示在扩容而创建新表使用 寻址算法 得到的 桶位头结点
                Node<K,V> e; int n;

                //条件一：永远不成立
                //条件二：永远不成立
                //条件三：永远不成立
                //条件四：在新扩容表中 重新定位 hash 对应的头结点
                //true -> 1.在oldTable中 对应的桶位在迁移之前就是null
                //        2.扩容完成后，有其它写线程，将此桶位设置为了null
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;

                //前置条件：扩容后的表 对应hash的桶位一定不是null，e为此桶位的头结点
                //e可能为哪些node类型？
                //1.node 类型
                //2.TreeBin 类型
                //3.FWD 类型

                for (;;) {
                    //eh 新扩容后表指定桶位的当前节点的hash
                    //ek 新扩容后表指定桶位的当前节点的key
                    int eh; K ek;
                    //条件成立：说明新扩容 后的表，当前命中桶位中的数据，即为 查询想要数据。
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;

                    //eh<0
                    //1.TreeBin 类型    2.FWD类型（新扩容的表，在并发很大的情况下，可能在此方法 再次拿到FWD类型..）
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            //说明此桶位 为 TreeBin 节点，使用TreeBin.find 查找红黑树中相应节点。
                            return e.find(h, k);
                    }

                    //前置条件：当前桶位头结点 并没有命中查询，说明此桶位是 链表
                    //1.将当前元素 指向链表的下一个元素
                    //2.判断当前元素的下一个位置 是否为空
                    //   true->说明迭代到链表末尾，未找到对应的数据，返回Null
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }
```

### get()方法源码解析

```java
   public V get(Object key) {
        //tab 引用map.table
        //e 当前元素
        //p 目标节点
        //n table数组长度
        //eh 当前元素hash
        //ek 当前元素key
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        //扰动运算后得到 更散列的hash值
        int h = spread(key.hashCode());

        //条件一：(tab = table) != null
        //true->表示已经put过数据，并且map内部的table也已经初始化完毕
        //false->表示创建完map后，并没有put过数据，map内部的table是延迟初始化的，只有第一次写数据时会触发创建逻辑。
        //条件二：(n = tab.length) > 0 true->表示table已经初始化
        //条件三：(e = tabAt(tab, (n - 1) & h)) != null
        //true->当前key寻址的桶位 有值
        //false->当前key寻址的桶位中是null，是null直接返回null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //前置条件：当前桶位有数据

            //对比头结点hash与查询key的hash是否一致
            //条件成立：说明头结点与查询Key的hash值 完全一致
            if ((eh = e.hash) == h) {
                //完全比对 查询key 和 头结点的key
                //条件成立：说明头结点就是查询数据
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }

            //条件成立：
            //1.-1  fwd 说明当前table正在扩容，且当前查询的这个桶位的数据 已经被迁移走了
            //2.-2  TreeBin节点，需要使用TreeBin 提供的find 方法查询。
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;




            //当前桶位已经形成链表的这种情况
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }

        }
        return null;
    }
```

### remove方法源码解析

```java
 final V replaceNode(Object key, V value, Object cv) {
        //计算key经过扰动运算后的hash
        int hash = spread(key.hashCode());
        //自旋
        for (Node<K,V>[] tab = table;;) {
            //f表示桶位头结点
            //n表示当前table数组长度
            //i表示hash命中桶位下标
            //fh表示桶位头结点 hash
            Node<K,V> f; int n, i, fh;

            //CASE1：
            //条件一：tab == null  true->表示当前map.table尚未初始化..  false->已经初始化
            //条件二：(n = tab.length) == 0  true->表示当前map.table尚未初始化..  false->已经初始化
            //条件三：(f = tabAt(tab, i = (n - 1) & hash)) == null true -> 表示命中桶位中为null，直接break， 会返回
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;

            //CASE2：
            //前置条件CASE2 ~ CASE3：当前桶位不是null
            //条件成立：说明当前table正在扩容中，当前是个写操作，所以当前线程需要协助table完成扩容。
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);

            //CASE3:
            //前置条件CASE2 ~ CASE3：当前桶位不是null
            //当前桶位 可能是 "链表" 也可能 是  "红黑树" TreeBin
            else {
                //保留替换之前的数据引用
                V oldVal = null;
                //校验标记
                boolean validated = false;
                //加锁当前桶位 头结点，加锁成功之后会进入 代码块。
                synchronized (f) {
                    //判断sync加锁是否为当前桶位 头节点，防止其它线程，在当前线程加锁成功之前，修改过 桶位 的头结点。
                    //条件成立：当前桶位头结点 仍然为f，其它线程没修改过。
                    if (tabAt(tab, i) == f) {
                        //条件成立：说明桶位 为 链表 或者 单个 node
                        if (fh >= 0) {
                            validated = true;

                            //e 表示当前循环处理元素
                            //pred 表示当前循环节点的上一个节点
                            Node<K,V> e = f, pred = null;
                            for (;;) {
                                //当前节点key
                                K ek;
                                //条件一：e.hash == hash true->说明当前节点的hash与查找节点hash一致
                                //条件二：((ek = e.key) == key || (ek != null && key.equals(ek)))
                                //if 条件成立，说明key 与查询的key完全一致。
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    //当前节点的value
                                    V ev = e.val;

       //条件一：cv == null  成立：不必对value，就做替换或者删除操作
                                //条件二：cv == pv ||(pv != null && cv.equals(pv)) 成立：说明“对比值”与当前p节点的值 一致
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        //删除 或者 替换

                                        //将当前节点的值 赋值给 oldVal 后续返回会用到
                                        oldVal = ev;

                                        //条件成立：说明当前是一个替换操作
                                        if (value != null)
                                            //直接替换
                                            e.val = value;
                                        //条件成立：说明当前节点非头结点
                                        else if (pred != null)
                                            //当前节点的上一个节点，指向当前节点的下一个节点。
                                            pred.next = e.next;

                                        else
                                            //说明当前节点即为 头结点，只需要将 桶位设置为头结点的下一个节点。
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }
                                pred = e;
                                if ((e = e.next) == null)
                                    break;
                            }
                        }

                        //条件成立：TreeBin节点。
                        else if (f instanceof TreeBin) {
                            validated = true;

                            //转换为实际类型 TreeBin t
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            //r 表示 红黑树 根节点
                            //p 表示 红黑树中查找到对应key 一致的node
                            TreeNode<K,V> r, p;

                            //条件一：(r = t.root) != null 理论上是成立
                            //条件二：TreeNode.findTreeNode 以当前节点为入口，向下查找key（包括本身节点）
                            //      true->说明查找到相应key 对应的node节点。会赋值给p
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                //保存p.val 到pv
                                V pv = p.val;

                                //条件一：cv == null  成立：不必对value，就做替换或者删除操作
                                //条件二：cv == pv ||(pv != null && cv.equals(pv)) 成立：说明“对比值”与当前p节点的值 一致
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    //替换或者删除操作


                                    oldVal = pv;

                                    //条件成立：替换操作
                                    if (value != null)
                                        p.val = value;


                                    //删除操作
                                    else if (t.removeTreeNode(p))
                                        //这里没做判断，直接搞了...很疑惑
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                //当其他线程修改过桶位 头结点时，当前线程 sync 头结点 锁错对象时，validated 为false，会进入下次for 自旋
                if (validated) {

                    if (oldVal != null) {
                        //替换的值 为null，说明当前是一次删除操作，oldVal ！=null 成立，说明删除成功，更新当前元素个数计数器。
                        if (value == null)
                            addCount(-1L, -1);
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
```

## TreeBin源码分解

### 成员变量分析

```java
 //红黑树 根节点 小刘讲师录制的红黑树教程：av83540396
        TreeNode<K,V> root;
        //链表的头节点
        volatile TreeNode<K,V> first;
        //等待者线程（当前lockState是读锁状态）
        volatile Thread waiter;
        /**
         * 1.写锁状态 写是独占状态，以散列表来看，真正进入到TreeBin中的写线程 同一时刻 只有一个线程。 1
         * 2.读锁状态 读锁是共享，同一时刻可以有多个线程 同时进入到 TreeBin对象中获取数据。 每一个线程 都会给 lockStat + 4
         * 3.等待者状态（写线程在等待），当TreeBin中有读线程目前正在读取数据时，写线程无法修改数据，那么就将lockState的最低2位 设置为 0b 10
         */
        volatile int lockState;

        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock

```

### 构造方法源码

```java
   TreeBin(TreeNode<K,V> b) {
            //设置节点hash为-2 表示此节点是TreeBin节点
            super(TREEBIN, null, null, null);
            //使用first 引用 treeNode链表
            this.first = b;
            //r 红黑树的根节点引用
            TreeNode<K,V> r = null;

            //x表示遍历的当前节点
            for (TreeNode<K,V> x = b, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                //强制设置当前插入节点的左右子树为null
                x.left = x.right = null;
                //条件成立：说明当前红黑树 是一个空树，那么设置插入元素 为根节点
                if (r == null) {
                    //根节点的父节点 一定为 null
                    x.parent = null;
                    //颜色改为黑色
                    x.red = false;
                    //让r引用x所指向的对象。
                    r = x;
                }

                else {
                    //非第一次循环，都会来带else分支，此时红黑树已经有数据了

                    //k 表示 插入节点的key
                    K k = x.key;
                    //h 表示 插入节点的hash
                    int h = x.hash;
                    //kc 表示 插入节点key的class类型
                    Class<?> kc = null;
                    //p 表示 为查找插入节点的父节点的一个临时节点
                    TreeNode<K,V> p = r;

                    for (;;) {
                        //dir (-1, 1)
                        //-1 表示插入节点的hash值大于 当前p节点的hash
                        //1 表示插入节点的hash值 小于 当前p节点的hash
                        //ph p表示 为查找插入节点的父节点的一个临时节点的hash
                        int dir, ph;
                        //临时节点 key
                        K pk = p.key;

                        //插入节点的hash值 小于 当前节点
                        if ((ph = p.hash) > h)
                            //插入节点可能需要插入到当前节点的左子节点 或者 继续在左子树上查找
                            dir = -1;
                        //插入节点的hash值 大于 当前节点
                        else if (ph < h)
                            //插入节点可能需要插入到当前节点的右子节点 或者 继续在右子树上查找
                            dir = 1;

                        //如果执行到 CASE3，说明当前插入节点的hash 与 当前节点的hash一致，会在case3 做出最终排序。最终
                        //拿到的dir 一定不是0，（-1， 1）
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);

                        //xp 想要表示的是 插入节点的 父节点
                        TreeNode<K,V> xp = p;
                        //条件成立：说明当前p节点 即为插入节点的父节点
                        //条件不成立：说明p节点 底下还有层次，需要将p指向 p的左子节点 或者 右子节点，表示继续向下搜索。
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            //设置插入节点的父节点 为 当前节点
                            x.parent = xp;
                            //小于P节点，需要插入到P节点的左子节点
                            if (dir <= 0)
                                xp.left = x;

                                //大于P节点，需要插入到P节点的右子节点
                            else
                                xp.right = x;

                            //插入节点后，红黑树性质 可能会被破坏，所以需要调用 平衡方法
                            r = balanceInsertion(r, x);
                            break;
                        }
                    }
                }
            }
            //将r 赋值给 TreeBin对象的 root引用。
            this.root = r;
            assert checkInvariants(root);
        }
```

### TreeBin的find方法源码

```java
 final Node<K,V> find(int h, Object k) {
            if (k != null) {

                //e 表示循环迭代的当前节点   迭代的是first引用的链表
                for (Node<K,V> e = first; e != null; ) {
                    //s 保存的是lock临时状态
                    //ek 链表当前节点 的key
                    int s; K ek;


                    //(WAITER|WRITER) => 0010 | 0001 => 0011
                    //lockState & 0011 != 0 条件成立：说明当前TreeBin 有等待者线程 或者 目前有写操作线程正在加锁
                    if (((s = lockState) & (WAITER|WRITER)) != 0) {
                        if (e.hash == h &&
                            ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                        e = e.next;
                    }

                    //前置条件：当前TreeBin中 等待者线程 或者 写线程 都没有
                    //条件成立：说明添加读锁成功
                    else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                                 s + READER)) {
                        TreeNode<K,V> r, p;
                        try {
                            //查询操作
                            p = ((r = root) == null ? null :
                                 r.findTreeNode(h, k, null));
                        } finally {
                            //w 表示等待者线程
                            Thread w;
                            //U.getAndAddInt(this, LOCKSTATE, -READER) == (READER|WAITER)
                            //1.当前线程查询红黑树结束，释放当前线程的读锁 就是让 lockstate 值 - 4
                            //(READER|WAITER) = 0110 => 表示当前只有一个线程在读，且“有一个线程在等待”
                            //当前读线程为 TreeBin中的最后一个读线程。

                            //2.(w = waiter) != null 说明有一个写线程在等待读操作全部结束。
                            if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                                (READER|WAITER) && (w = waiter) != null)
                                //使用unpark 让 写线程 恢复运行状态。
                                LockSupport.unpark(w);
                        }
                        return p;
                    }
                }
            }
            return null;
        }
```

### putTreeVal源码解析

```java
   /**
         * Finds or adds a node.
         * @return null if added
         */
        final TreeNode<K,V> putTreeVal(int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if (p == null) {
                    first = root = new TreeNode<K,V>(h, k, v, null, null);
                    break;
                }
                else if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.findTreeNode(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.findTreeNode(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }


                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    //当前循环节点xp 即为 x 节点的爸爸

                    //x 表示插入节点
                    //f 老的头结点
                    TreeNode<K,V> x, f = first;
                    first = x = new TreeNode<K,V>(h, k, v, f, xp);

                    //条件成立：说明链表有数据
                    if (f != null)
                        //设置老的头结点的前置引用为 当前的头结点。
                              //头插法
                        f.prev = x;


                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;


                    if (!xp.red)
                        x.red = true;
                    else {
                        //表示 当前新插入节点后，新插入节点 与 父节点 形成 “红红相连”
                        lockRoot();
                        try {
                            //平衡红黑树，使其再次符合规范。
                            root = balanceInsertion(root, x);
                        } finally {
                            unlockRoot();
                        }
                    }
                    break;
                }
            }
            assert checkInvariants(root);
            return null;
        }
```

### contentLock源码

```java
   private final void contendedLock() {
            boolean waiting = false;
            for (int s;;) {
                //...00000 0000 0010 取反  11...111 1111 1101
                //11...111 1111 1101  & 0100 代表此时没有读线程了 如果==0代表此时没有任何读线程
                if (((s = lockState) & ~WAITER) == 0) {
                    //直接可以开始写操作了
                    if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {
                        if (waiting)
                            waiter = null;
                        return;
                    }
                }
                //代表还有线程正在读。
                else if ((s & WAITER) == 0) {
                     //0100 | 0010  == 0110 代表该线程正在等待。
                    if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) {
                        waiting = true;
                        waiter = java.lang.Thread.currentThread();
                    }
                }
                else if (waiting)
                    //如果waiting为true,代表已经是等待线程已经开始等待，则直接阻塞。等待最后一个读线程唤醒。
                    LockSupport.park(this);
            }
        }
```

# FutureTask源码解析

## 成员变量解析

```java
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    //表示当前task状态
    private volatile int state;
    //当前任务运行中或者处于线程池队列中。
    private static final int NEW          = 0;
    //当前任务正在结束，即将完全结束，一种临界状态
    private static final int COMPLETING   = 1;
    //当前任务正常结束
    private static final int NORMAL       = 2;
    //当前任务执行过程中发生了异常。 内部封装的 callable.run() 向上抛出异常了
    private static final int EXCEPTIONAL  = 3;
    //当前任务被取消
    private static final int CANCELLED    = 4;
    //当前任务中断中..
    private static final int INTERRUPTING = 5;
    //当前任务已中断
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    //submit(runnable/callable)   runnable 使用 装饰者模式 伪装成 Callable了。
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    //正常情况下：任务正常执行结束，outcome保存执行结果。 callable 返回值。
    //非正常情况：callable向上抛出异常，outcome保存异常
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    //当前任务被线程执行期间，保存当前执行任务的线程对象引用。
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    //因为会有很多线程去get当前任务的结果，所以 这里使用了一种数据结构 stack 头插 头取 的一个队列。
    private volatile WaitNode waiters;
```

## run方法源码解析

```java
   //线程程序执行入口,Windows下对应的是线程，Linux是轻量级进程
    public void run() {
        //如果不是NEW状态代表已经被执行过了或者被取消了。 如果线程池中队列的任务被取消，或者被中断，也不会执行
        if (state != NEW ||
                //将runner成员变量的值修改为当前线程，有可能失败，被其他线程抢占
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            //state == NEW有可能被外部线程取消掉
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    /**我觉得在程序员自己编写的程序中运行的时候，可能会存在响应中断的操作，所以在运行的时候设置中断将执行任务的线程响应中断操作*/
                    result = c.call();
                    //如果没异常ran为true，否则为false封装异常结果
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    //为true代表程序员自己编写的逻辑没异常
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;

            //让出CPU让calcel方法尽快执行完毕，将INTERRUPTING状态改为INTERRUPTED，代表真正的结束
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```



## finishCompletion源码解析

```java
    private void finishCompletion() {
        //首先使用cas方式将等待获取结果的线程修改为Null，因为可能有多个线程
        //所以取出每一个WaitNode的线程执行unPark唤醒操作
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();
       //将该值赋值为空，代表该线程已经没有任务
        callable = null;        // to reduce footprint
    }
```

```java
    protected void set(V v) {
        //使用CAS方式设置当前任务状态为 完成中..
        //有没有可能失败呢？ 外部线程等不及了，直接在set执行CAS之前 将  task取消了。  很小概率事件。
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {

            outcome = v;
            //将结果赋值给 outcome之后，马上会将当前任务状态修改为 NORMAL 正常结束状态。
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state

            //猜一猜？
            //最起码得把get() 再此阻塞的线程 唤醒..
            finishCompletion();
        }
    }
```

```java
 protected void setException(Throwable t) {
        //使用CAS方式设置当前任务状态为 完成中..
        //有没有可能失败呢？ 外部线程等不及了，直接在set执行CAS之前 将  task取消了。  很小概率事件。
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //引用的是 callable 向上层抛出来的异常。
            outcome = t;
            //将当前任务的状态 修改为 EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            //回头讲完 get() 。
            finishCompletion();
        }
    }
```

## get方法源码

```java
  public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //如果成立，代表还正在执行或者在任务队列中，需要阻塞
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        
        
        //返回结果
        return report(s);
    }
```

```java
    /**
     * @throws CancellationException {@inheritDoc}
     带超时时间的
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }
```

## awaitDone方法源码

```java
//阻塞get获取结果线程的方法   
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            //判断线程是否是可中断的，并且清除中断标志位，恢复为默认值。
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }
            int s = state;
            //代表已经正常/异常结束了或者是被取消掉或者执行任务的线程被中断
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            //让出cpu资源让线程尽快完成任务
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();

            //普遍情况下第一次来到这里
            else if (q == null)
                q = new WaitNode();

            else if (!queued)
                //CAS方式去将q设置为waiters的头结点，头插法。
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            //前提条件是s < COMPLETING ？ 只有可能是NEW的状态。并且线程还被头插了
            else if (timed) {
                nanos = deadline - System.nanoTime();
                //代表时间已经到
                if (nanos <= 0L) {
                    //移除当前线程的waitNode结点然后返回
                    removeWaiter(q);
                    return state;
                }

                LockSupport.parkNanos(this, nanos);
            }
            else
                //当cancel()方法或者set()方法被调用的时候会唤醒该任务阻塞的所有线程，然后所有线程
               //进行自旋，park方法可以阻塞当前线程，如果调用unpark方法或者中断当前线程，则会从park方法中返回。
                // 那么上面的自旋判断会成立，然后移出等待队列
               //然后扔出异常
                LockSupport.park(this);
        }
    }
```

## calcel方法源码解析

```java
    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {

        //必须要是NEW状态，代表正在执行中或者在线程池的队列中。
        if (!(state == NEW &&
                //将state变量从NEW改为INTERRUPTING或者CANCELLED 根据mayInterruptIfRunning参数而定
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                //代表为true,可以在运行中响应中断
                try {
                    //拿到正在运行该任务的线程，有可能是NULL，处于线程池的任务队列中。
                    Thread t = runner;

                    /** 此处假设？ 如果是正在运行的任务，会在run()方法响应中断，如果没有在运行，runner会为Null，不会被设置为中断，只会改状态*/
                    if (t != null)
                        t.interrupt();
                } finally {

                    //强制将state变量设置为INTERRUPTED代表中断完成
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            //唤醒get阻塞的所有线程
            finishCompletion();
        }
        return true;
    }
```

## report方法源码解析

```java
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        //代表任务以正常状态结束
        if (s == NORMAL)
            return (V)x;
        
        //代表被取消或者被中断
        if (s >= CANCELLED)
            throw new CancellationException();
        
        //代表任务以异常状态结束，外部线程调用get会get到异常
        throw new ExecutionException((Throwable)x);
    }
```

## removeWaiter方法源码

```java
//移除node对应的线程 
private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                //s为该链表的下一个结点 Pre为上一个 q为当前结点
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    //上面属于该链表中的线程的thead被置为null，找到该线程。
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        //代表当前线程的上一个线程也可能为Null，因为可能会有并发，所以需要跳出里层循环重新赋值重新检查
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    //代表为头结点
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

```

# 线程池源码解析

## 线程池工作原理

![image-20210402111559002](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210402111559002.png)

* workerCount, indicating the effective number of threads    工作数量，表示有效的线程数量

* runState,    indicating whether running, shutting down etc  运行状态表示是否运行，关闭等。

  继承类图 Executor-> ExecutorService->AbstractExcutorService->ThreadPoolExecutor

## 线程池状态

![image-20210402113511304](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210402113511304.png)

 The runState provides the main lifecycle control, taking on values:

- **RUNNING:  Accept new tasks and process queued tasks**  接受新任务并且处理任务队列

-    **SHUTDOWN: Don't accept new tasks, but process queued tasks **不接受新任务，但是处理队列任务

-    **STOP:     Don't accept new tasks, don't process queued tasks,and interrupt in-progress tasks** 不接受新任务，也不处理队列任务并且中断执行任务中的线程

-    **TIDYING:  All tasks have terminated, workerCount is zero,** **the thread transitioning to state TIDYING** **will run the terminated() hook method**

  所有任务已经结束，工作线程数量也为0，线程将状态改为整体TIDYING,然后会运行terminated钩子方法

-   **TERMINATED: terminated() has completed **  terminated方法结束

- RUNNING -> SHUTDOWN
      On invocation of shutdown(), perhaps implicitly in finalize()

-  (RUNNING or SHUTDOWN) -> STOP
      On invocation of shutdownNow()

-  SHUTDOWN -> TIDYING

   When both queue and pool are empty

-  STOP -> TIDYING
      When pool is empty

-  TIDYING -> TERMINATED
      When the terminated() hook method has completed

## 线程池成员变量字段解析

```java
 //两种语义，高三位表示线程池运行状态，低29位低位表示当前线程池所拥有的线程数量
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    /**默认COUNT_BITS = 29,被final修饰的 为什么不直接写29，害怕Integer在其他的jdk版本中字节不是四位，*/
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //线程的最大容量 1左移29位 -1，000 111111111111111111111111111代表5亿多
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    //高三位表示运行状态，从小到大依次
    // runState is stored in the high-order bits
    /** 五种运行状态，RUNNING的值 -1左移29位 低29位为0
     * 111 00000 00000000 00000000 00000000*/
    private static final int RUNNING    = -1 << COUNT_BITS;
    /** SHUTDOWN的值为0，代表外部线程调用了线程池的shutdown方法
     * 000 00000 00000000 00000000 00000000*/
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    /** 1左移29位，001 00000 00000000 00000000 00000000 5亿多，代表外部线程调用了线程池的shutdownNow方法*/
    private static final int STOP       =  1 << COUNT_BITS;
    /**2左移29位，010 00000 00000000 00000000 00000000  -1073741824 -10亿？？  */
    private static final int TIDYING    =  2 << COUNT_BITS;
    /**三左移29位 011 00000 00000000 00000000 00000000  */
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    // CAPACITY的反 那么就是 111 00000000000000000000000 & ctl的值，值拿高三位，算出来是多少就是多少
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    //只拿低29位就是workerCount
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    //重置线程池ctl的值
    //rs表示线程池状态  wc表示线程池状态
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    /*
     * Bit field accessors that don't require unpacking ctl.
     * These depend on the bit layout and on workerCount being never negative.
     */

    //判断运行状态c是否小于s
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    //判断运行状态 c 是否大于等于 s
    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    //判断是否事运行状态
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }

    /**
     * Attempts to CAS-increment the workerCount field of ctl.
     * 比较并且增加1工作线程数量
     */
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    /**
     * Attempts to CAS-decrement the workerCount field of ctl.
     * 比较并且减1工作线程数量。
     */
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    /**
     * Decrements the workerCount field of ctl. This is called only on
     * abrupt termination of a thread (see processWorkerExit). Other
     * decrements are performed within getTask.
     * 将工作线程数量减1，一定会成功
     */
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

        /**工作线程的任务队列*/
    private final BlockingQueue<Runnable> workQueue;

      /**全局锁，增加Worker,减少Worker需要持有锁，修改线程池运行状态*/
    private final ReentrantLock mainLock = new ReentrantLock();

    /**
     * Set containing all worker threads in pool. Accessed only when
     * holding mainLock.
     */
    /**所有的工作线程*/
    private final HashSet<Worker> workers = new HashSet<Worker>();

    /**
     * Wait condition to support awaitTermination
     * 当外部线程调用awaitTermination方法的时候，外部线程会等待线程池的状态为Termination为止
     * 等待实现的方式就是封装为waitNode加入到Condition队列中
     */
    private final Condition termination = mainLock.newCondition();

       /**
     * Tracks largest attained pool size. Accessed only under
     * mainLock.
     * 记录线程池生命周期内的最大线程数量
     */
    private int largestPoolSize;

    /**
     * Counter for completed tasks. Updated only on termination of
     * worker threads. Accessed only under mainLock.
     */
    /**完成的任务数量*/
    private long completedTaskCount;

   //不推荐使用Defaultxxx工厂来生产线程比如Executors.Fix... Executors.Cachexxx，不容易业务定位错误，线程名字是不易阅读的。
    private volatile ThreadFactory threadFactory;

    /**
     * Handler called when saturated or shutdown in execute.
     */
    /**线程池的拒绝策略*/
    private volatile RejectedExecutionHandler handler;

    /**
     * Timeout in nanoseconds for idle threads waiting for work.
     * Threads use this timeout when there are more than corePoolSize
     * present or if allowCoreThreadTimeOut. Otherwise they wait
     * forever for new work.
     */
    /**最大线程数的线程存活时间 allowCoreThreadTimeOut为true的时候，核心线程数也会被回收
     * allowCoreThreadTimeOut为false,代表会维护核心线程数量的线程不会回收*/
    private volatile long keepAliveTime;

    /**
     * If false (default), core threads stay alive even when idle.
     * If true, core threads use keepAliveTime to time out waiting
     * for work.
     */
    //allowCoreThreadTimeOut为true的时候 核心线程数也会被回收，allowCoreThreadTimeOut为false,   代表会维护核心线程数量的线程不会回收
    private volatile boolean allowCoreThreadTimeOut;

    /**
     * Core pool size is the minimum number of workers to keep alive
     * (and not allow to time out etc) unless allowCoreThreadTimeOut
     * is set, in which case the minimum is zero.
     * 核心数
     */
    private volatile int corePoolSize;

    /**
     * Maximum pool size. Note that the actual maximum is internally
     * bounded by CAPACITY.
     * 最大线程数，最大不能超过CAPACITY值
     */
    private volatile int maximumPoolSize;

    /**
     * The default rejected execution handler
     */
    /**默认是抛出异常的拒绝策略*/
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```

## 内部类Worker源码解析

```java
  private final class Worker
            extends AbstractQueuedSynchronizer
            implements Runnable
    {
        //Worker采用了AQS的独占模式
        //独占模式：两个重要属性  state  和  ExclusiveOwnerThread
        //state：0时表示未被占用 > 0时表示被占用   < 0 时 表示初始状态，这种情况下不能被抢锁。
        //ExclusiveOwnerThread:表示独占锁的线程。
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        //worker内部封装的工作线程
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        //假设firstTask不为空，那么当worker启动后（内部的线程启动)会优先执行firstTask，当执行完firstTask后，会到queue中去获取下一个任务。
        Runnable firstTask;

        /** Per-thread task counter */
        //记录当前worker所完成任务数量。
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        //firstTask可以为null。为null 启动后会到queue中获取。
        Worker(Runnable firstTask) {
            //设置AQS独占模式为初始化中状态，这个时候 不能被抢占锁。
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            //使用线程工厂创建了一个线程，并且将当前worker 指定为 Runnable，也就是说当thread启动的时候，会以worker.run()为入口。
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        //当worker启动时，会执行run()
        public void run() {
            //ThreadPoolExecutor->runWorker() 这个是核心方法，等后面分析worker启动后逻辑时会以这里切入。
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.
        //判断当前worker的独占锁是否被独占。
        //0 表示未被占用
        //1 表示已占用
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }


        //尝试去占用worker的独占锁
        //返回值 表示是否抢占成功
        protected boolean tryAcquire(int unused) {
            //使用CAS修改 AQS中的 state ，期望值为0(0时表示未被占用），修改成功表示当前线程抢占成功
            //那么则设置 ExclusiveOwnerThread 为当前线程。
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }

            return false;
        }

        //外部不会直接调用这个方法 这个方法是AQS 内调用的，外部调用unlock时 ，unlock->AQS.release() ->tryRelease()
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        //加锁，加锁失败时，会阻塞当前线程，直到获取到锁位置。
        public void lock()        { acquire(1); }

        //尝试去加锁，如果当前锁是未被持有状态，那么加锁成功后 会返回true，否则不会阻塞当前线程，直接返回false.
        public boolean tryLock()  { return tryAcquire(1); }

        //一般情况下，咱们调用unlock 要保证 当前线程是持有锁的。
        //特殊情况，当worker的state == -1 时，调用unlock 表示初始化state 设置state == 0
        //启动worker之前会先调用unlock()这个方法。会强制刷新ExclusiveOwnerThread == null State==0
        public void unlock()      { release(1); }

        //就是返回当前worker的lock是否被占用。
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

## ThreadPoolExecutor构造方法源码解析

```java
 //三个构造方法，套娃，没指定线程工厂的话默认使用DefaultThreadFactory，没指定拒绝策略的话默认使用Abort 终止策略，也就是抛出异常  
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

```java
 //三个构造方法，套娃，没指定线程工厂的话默认使用DefaultThreadFactory
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }
```

```java
   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        //一堆判断，最大线程数不能小于核心线程数可以等于，最大线程数的存活时间也不能小于0;
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        //如果没有指定队列或者线程工厂或者拒绝策略都会抛出空指针异常
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();

        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        //将你指定的时间转换为纳秒然后全部赋为初始值
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

## execute方法源码解析

```java
  public void execute(Runnable command) {
       //非空判断
        if (command == null)
            throw new NullPointerException();
        //拿到ctl的值
        int c = ctl.get();

        //当前线程数量小于核心线程数，那么就会创建worker线程并且执行该任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;

            //增加worker线程失败，可能存在并发，其他线程已经添加到线程并且线程数已经达到corePoolSize数量，所以会添加失败
            //线程被外部线程调用线程池的shutDown或者shutdownNow方法，总之不处于running状态
            c = ctl.get();
        }
        //前置条件，当前线程数量大于等于核心线程数
        //或者非running状态
        if (isRunning(c) && workQueue.offer(command)) {

            //进入该逻辑代表是running状态并且任务入队成功
            int recheck = ctl.get();

            //上面如果是running状态的线程池就不会再移除了，直接false，执行下面的else if
            if (! isRunning(recheck) && remove(command))
                //1.线程被外部线程调用线程池的shutDown或者shutdownNow方法，总之不处于running状态并且将任务从队列移除成功，会执行相应的拒绝策略
                //可以理解为在程序进行上层if之后被外部线程将状态修改。
                reject(command);

            //代表不处于running状态
            //代表处于running状态，但是将任务从任务队列中移除的时候失败，可能任务已经被线程池中的线程消费掉
            //主要是防止处于running状态下的线程数量为0，然后增加线程去执行任务。
            //担保机制，为了保证线程池running状态下，线程数不为0，担心刚提交的任务没有线程被消费
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //已经大于等于corePoolSize数并且
        //非running状态，或者是running状态但是增加到队列失败(有界队列并且队列已经满)
        //再增加最大线程数，如果也失败，则会执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }

```

<img src="https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210402232635364.png" alt="image-20210402232635364" style="zoom:50%;" />

## 线程池提交任务源码解析

```java
//提交一个任务，sumbit方法会被封装为FutureTask,正常逻辑为NEW->Compeltion->Normal
  ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 5, 1, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(100));
        
        poolExecutor.submit(new Callable<Object>() {
            @Override
            public Object call() throws Exception {
                return null;
            }
        });
```

```java
   public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        //将callable接口封装为FutureTask
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        //FutureTask接口返回给程序员，程序员可以调用get方法获取结果
        return ftask;
    }

   //封装为FutureTask
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

**正常逻辑**

当程序员调用get方法获取结果的时候，如果未完成则会阻塞，直到内部的线程消费该线程，FutureTask的run方法会被执行，并且会调用FutureTask里面的Callable的call方法，进行业务逻辑的执行，执行完毕后则会唤醒get获取结果的线程。

提交的runnable会被FutrueTask使用适配器封装为callable。

线程池提交的runnable会被futrueTask封装为callable接口，并且向程序员返回futureTask接口



## addWorker源码解析

```java
  //firstTask 可以为null，表示启动worker之后，worker自动到queue中获取任务.. 如果不是null，则worker优先执行firstTask
    //core 采用的线程数限制 如果为true 采用 核心线程数限制  false采用 maximumPoolSize线程数限制.

    //返回值总结：
    //true 表示创建worker成功，且线程启动

    //false 表示创建失败。
    //1.线程池状态rs > SHUTDOWN (STOP/TIDYING/TERMINATION)
    //2.rs == SHUTDOWN 但是队列中已经没有任务了 或者 当前状态是SHUTDOWN且队列未空，但是firstTask不为null
    //3.当前线程池已经达到指定指标（coprePoolSize 或者 maximumPoolSIze）
    //4.threadFactory 创建的线程是null
    private boolean addWorker(Runnable firstTask, boolean core) {
        //自旋 判断当前线程池状态是否允许创建线程的事情。
        retry:
        for (;;) {
            //获取当前ctl值保存到c
            int c = ctl.get();
            //获取当前线程池运行状态 保存到rs长
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.

            //条件一：rs >= SHUTDOWN 成立：说明当前线程池状态不是running状态
            //条件二：前置条件，当前的线程池状态不是running状态  ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty())
            //rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()
            //表示：当前线程池状态是SHUTDOWN状态 & 提交的任务是空，addWorker这个方法可能不是execute调用的。 & 当前任务队列不是空
            //排除掉这种情况，当前线程池是SHUTDOWN状态，但是队列里面还有任务尚未处理完，这个时候是允许添加worker，但是不允许再次提交task。
            if (rs >= SHUTDOWN &&
                    ! (rs == SHUTDOWN &&
                            firstTask == null &&
                            ! workQueue.isEmpty()))
                //什么情况下回返回false?
                //线程池状态 rs > SHUTDOWN
                //rs == SHUTDOWN 但是队列中已经没有任务了 或者 rs == SHUTDOWN 且 firstTask != null
                return false;

            //上面这些代码，就是判断 当前线程池状态 是否允许添加线程。


            //内部自旋 获取创建线程令牌的过程。
            for (;;) {
                //获取当前线程池中线程数量 保存到wc中
                int wc = workerCountOf(c);

                //条件一：wc >= CAPACITY 永远不成立，因为CAPACITY是一个5亿多大的数字
                //条件二：wc >= (core ? corePoolSize : maximumPoolSize)
                //core == true ,判断当前线程数量是否>=corePoolSize，会拿核心线程数量做限制。
                //core == false,判断当前线程数量是否>=maximumPoolSize，会拿最大线程数量做限制。
                if (wc >= CAPACITY ||
                        wc >= (core ? corePoolSize : maximumPoolSize))
                    //执行到这里，说明当前无法添加线程了，已经达到指定限制了
                    return false;

                //条件成立：说明记录线程数量已经加1成功，相当于申请到了一块令牌。
                //条件失败：说明可能有其它线程，修改过ctl这个值了。
                //可能发生过什么事？
                //1.其它线程execute() 申请过令牌了，在这之前。导致CAS失败
                //2.外部线程可能调用过 shutdown() 或者 shutdownNow() 导致线程池状态发生变化了，咱们知道 ctl 高3位表示状态
                //状态改变后，cas也会失败。
                if (compareAndIncrementWorkerCount(c))
                    //进入到这里面，一定是cas成功啦！申请到令牌了
                    //直接跳出了 retry 外部这个for自旋。
                    break retry;

                //CAS失败，没有成功的申请到令牌
                //获取最新的ctl值
                c = ctl.get();  // Re-read ctl
                //判断当前线程池状态是否发生过变化,如果外部在这之前调用过shutdown. shutdownNow 会导致状态变化。
                if (runStateOf(c) != rs)
                    //状态发生变化后，直接返回到外层循环，外层循环负责判断当前线程池状态，是否允许创建线程。
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }





        //表示创建的worker是否已经启动，false未启动  true启动
        boolean workerStarted = false;
        //表示创建的worker是否添加到池子中了 默认false 未添加 true是添加。
        boolean workerAdded = false;

        //w表示后面创建worker的一个引用。
        Worker w = null;
        try {
            //创建Worker，执行完后，线程应该是已经创建好了。
            w = new Worker(firstTask);

            //将新创建的worker节点的线程 赋值给 t
            final Thread t = w.thread;

            //为什么要做 t != null 这个判断？
            //为了防止ThreadFactory 实现类有bug，因为ThreadFactory 是一个接口，谁都可以实现。
            //万一哪个 小哥哥 脑子一热，有bug，创建出来的线程 是null、、
            //Doug lea考虑的比较全面。肯定会防止他自己的程序报空指针，所以这里一定要做！
            if (t != null) {
                //将全局锁的引用保存到mainLock
                final ReentrantLock mainLock = this.mainLock;
                //持有全局锁，可能会阻塞，直到获取成功为止，同一时刻 操纵 线程池内部相关的操作，都必须持锁。
                mainLock.lock();
                //从这里加锁之后，其它线程 是无法修改当前线程池状态的。
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    //获取最新线程池运行状态保存到rs中
                    int rs = runStateOf(ctl.get());

                    //条件一：rs < SHUTDOWN 成立：最正常状态，当前线程池为RUNNING状态.
                    //条件二：前置条件：当前线程池状态不是RUNNING状态。
                    //(rs == SHUTDOWN && firstTask == null)  当前状态为SHUTDOWN状态且firstTask为空。其实判断的就是SHUTDOWN状态下的特殊情况，
                    //只不过这里不再判断队列是否为空了
                    if (rs < SHUTDOWN ||
                            (rs == SHUTDOWN && firstTask == null)) {
                        //t.isAlive() 当线程start后，线程isAlive会返回true。
                        //防止脑子发热的程序员，ThreadFactory创建线程返回给外部之前，将线程start了。。
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();

                        //将咱们创建的worker添加到线程池中。
                        workers.add(w);
                        //获取最新当前线程池线程数量
                        int s = workers.size();
                        //条件成立：说明当前线程数量是一个新高。更新largestPoolSize
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //表示线程已经追加进线程池中了。
                        workerAdded = true;
                    }
                } finally {
                    //释放线程池全局锁。
                    mainLock.unlock();
                }
                //条件成立:说明 添加worker成功
                //条件失败：说明线程池在lock之前，线程池状态发生了变化，导致添加失败。
                if (workerAdded) {
                    //成功后，则将创建的worker启动，线程启动。
                    t.start();
                    //启动标记设置为true
                    workerStarted = true;
                }
            }

        } finally {
            //条件成立：! workerStarted 说明启动失败，需要做清理工作。
            if (! workerStarted)
                //失败时做什么清理工作？
                //1.释放令牌
                //2.将当前worker清理出workers集合
                addWorkerFailed(w);
        }

        //返回新创建的线程是否启动。
        return workerStarted;
    }
```

```java
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                //移除线程
                workers.remove(w);
            //将线程数量减1，肯定成功
            decrementWorkerCount();
            //试着结束线程池
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```

## runWorker源码解析

```java
   //w 就是启动worker
    final void runWorker(Worker w) {
        //wt == w.thread
        Thread wt = Thread.currentThread();
        //将初始执行task赋值给task
        Runnable task = w.firstTask;
        //清空当前w.firstTask引用
        w.firstTask = null;
        //这里为什么先调用unlock? 就是为了初始化worker state == 0 和 exclusiveOwnerThread ==null
        w.unlock(); // allow interrupts

        //是否是突然退出，true->发生异常了，当前线程是突然退出，回头需要做一些处理
        //false->正常退出。
        boolean completedAbruptly = true;

        try {
            //条件一：task != null 指的就是firstTask是不是null，如果不是null，直接执行循环体里面。
            //条件二：(task = getTask()) != null   条件成立：说明当前线程在queue中获取任务成功，getTask这个方法是一个会阻塞线程的方法
            //getTask如果返回null，当前线程需要执行结束逻辑。
            while (task != null || (task = getTask()) != null) {
                //worker设置独占锁 为当前线程
                //为什么要设置独占锁呢？shutdown时会判断当前worker状态，根据独占锁是否空闲来判断当前worker是否正在工作。
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt

                //条件一：runStateAtLeast(ctl.get(), STOP)  说明线程池目前处于STOP/TIDYING/TERMINATION 此时线程一定要给它一个中断信号
                //条件一成立：runStateAtLeast(ctl.get(), STOP)&& !wt.isInterrupted()
                //上面如果成立：说明当前线程池状态是>=STOP 且 当前线程是未设置中断状态的，此时需要进入到if里面，给当前线程一个中断。

                //假设：runStateAtLeast(ctl.get(), STOP) == false
                // (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)) 在干吗呢？
                // Thread.interrupted() 获取当前中断状态，且设置中断位为false。连续调用两次，这个interrupted()方法 第二次一定是返回false.
                // runStateAtLeast(ctl.get(), STOP) 大概率这里还是false.
                // 其实它在强制刷新当前线程的中断标记位 false，因为有可能上一次执行task时，业务代码里面将当前线程的中断标记位 设置为了 true，且没有处理
                // 这里一定要强制刷新一下。不会再影响到后面的task了。
                //假设：Thread.interrupted() == true  且 runStateAtLeast(ctl.get(), STOP)) == true
                //这种情况有发生几率么？
                //有可能，因为外部线程在 第一次 (runStateAtLeast(ctl.get(), STOP) == false 后，有机会调用shutdown 、shutdownNow方法，将线程池状态修改
                //这个时候，也会将当前线程的中断标记位 再次设置回 中断状态。
                if ((runStateAtLeast(ctl.get(), STOP) ||
                        (Thread.interrupted() &&
                                runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                    wt.interrupt();


                try {
                    //钩子方法，留给子类实现的
                    beforeExecute(wt, task);
                    //表示异常情况，如果thrown不为空，表示 task运行过程中 向上层抛出异常了。
                    Throwable thrown = null;
                    try {
                        //task 可能是FutureTask 也可能是 普通的Runnable接口实现类。
                        //如果前面是通过submit()提交的 runnable/callable 会被封装成 FutureTask。这个不清楚，请看上一期，在b站。
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //钩子方法，留给子类实现的
                        afterExecute(task, thrown);
                    }
                } finally {
                    //将局部变量task置为Null
                    task = null;
                    //更新worker完成任务数量
                    w.completedTasks++;

                    //worker处理完一个任务后，会释放掉独占锁
                    //1.正常情况下，会再次回到getTask()那里获取任务  while(getTask...)
                    //2.task.run()时内部抛出异常了..
                    w.unlock();
                }
            }

            //什么情况下，会来到这里？
            //getTask()方法返回null时，说明当前线程应该执行退出逻辑了。
            completedAbruptly = false;
        } finally {

            //task.run()内部抛出异常时，直接从 w.unlock() 那里 跳到这一行。
            //正常退出 completedAbruptly == false
            //异常退出 completedAbruptly == true
            processWorkerExit(w, completedAbruptly);
        }
    }
```

## getTask源码解析

```java
   //什么情况下会返回null？
    //1.rs >= STOP 成立说明：当前的状态最低也是STOP状态，一定要返回null了
    //2.前置条件 状态是 SHUTDOWN ，workQueue.isEmpty()
    //3.线程池中的线程数量 超过 最大限制时，会有一部分线程返回Null
    //4.线程池中的线程数超过corePoolSize时，会有一部分线程 超时后，返回null。
    private Runnable getTask() {
        //表示当前线程获取任务是否超时 默认false true表示已超时
        boolean timedOut = false; // Did the last poll() time out?

        //自旋
        for (;;) {
            //获取最新ctl值保存到c中。
            int c = ctl.get();
            //获取线程池当前运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //条件一：rs >= SHUTDOWN 条件成立：说明当前线程池是非RUNNING状态，可能是 SHUTDOWN/STOP....
            //条件二：(rs >= STOP || workQueue.isEmpty())
            //2.1:rs >= STOP 成立说明：当前的状态最低也是STOP状态，一定要返回null了
            //2.2：前置条件 状态是 SHUTDOWN ，workQueue.isEmpty()条件成立：说明当前线程池状态为SHUTDOWN状态 且 任务队列已空，此时一定返回null。
            //返回null，runWorker方法就会将返回Null的线程执行线程退出线程池的逻辑。
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                //使用CAS+死循环的方式让 ctl值 -1
                decrementWorkerCount();
                return null;
            }

            //执行到这里，有几种情况？
            //1.线程池是RUNNING状态
            //2.线程池是SHUTDOWN状态 但是队列还未空，此时可以创建线程。

            //获取线程池中的线程数量
            int wc = workerCountOf(c);

            // Are workers subject to culling?
            //timed == true 表示当前这个线程 获取 task 时 是支持超时机制的，使用queue.poll(xxx,xxx); 当获取task超时的情况下，下一次自旋就可能返回null了。
            //timed == false 表示当前这个线程 获取 task 时 是不支持超时机制的，当前线程会使用 queue.take();

            //情况1：allowCoreThreadTimeOut == true 表示核心线程数量内的线程 也可以被回收。
            //所有线程 都是使用queue.poll(xxx,xxx) 超时机制这种方式获取task.
            //情况2：allowCoreThreadTimeOut == false 表示当前线程池会维护核心数量内的线程。
            //wc > corePoolSize
            //条件成立：当前线程池中的线程数量是大于核心线程数的，此时让所有路过这里的线程，都是用poll 支持超时的方式去获取任务，
            //这样，就会可能有一部分线程获取不到任务，获取不到任务 返回Null，然后..runWorker会执行线程退出逻辑。
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;


            //条件一：(wc > maximumPoolSize || (timed && timedOut))
            //1.1：wc > maximumPoolSize  为什么会成立？setMaximumPoolSize()方法，可能外部线程将线程池最大线程数设置为比初始化时的要小
            //1.2: (timed && timedOut) 条件成立：前置条件，当前线程使用 poll方式获取task。上一次循环时  使用poll方式获取任务时，超时了
            //条件一 为true 表示 线程可以被回收，达到回收标准，当确实需要回收时再回收。

            //条件二：(wc > 1 || workQueue.isEmpty())
            //2.1: wc > 1  条件成立，说明当前线程池中还有其他线程，当前线程可以直接回收，返回null
            //2.2: workQueue.isEmpty() 前置条件 wc == 1， 条件成立：说明当前任务队列 已经空了，最后一个线程，也可以放心的退出。
            if ((wc > maximumPoolSize || (timed && timedOut))
                    && (wc > 1 || workQueue.isEmpty())) {
                //使用CAS机制 将 ctl值 -1 ,减1成功的线程，返回null
                //CAS成功的，返回Null
                //CAS失败？ 为什么会CAS失败？
                //1.其它线程先你一步退出了
                //2.线程池状态发生变化了。
                if (compareAndDecrementWorkerCount(c))
                    return null;
                //再次自旋时，timed有可能就是false了，因为当前线程cas失败，很有可能是因为其它线程成功退出导致的，再次咨询时
                //检查发现，当前线程 就可能属于 不需要回收范围内了。
                continue;
            }

            try {
                //获取任务的逻辑
                Runnable r = timed ?
                        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                        workQueue.take();

                //条件成立：返回任务
                if (r != null)
                    return r;

                //说明当前线程超时了...
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

## processWorkerExit源码解析

```java
  private void processWorkerExit(Worker w, boolean completedAbruptly) {
        //条件成立：代表当前w 这个worker是发生异常退出的，task任务执行过程中向上抛出异常了..
        //异常退出时，ctl计数，并没有-1
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        //获取线程池的全局锁引用
        final ReentrantLock mainLock = this.mainLock;
        //加锁
        mainLock.lock();
        try {
            //将当前worker完成的task数量，汇总到线程池的completedTaskCount
            completedTaskCount += w.completedTasks;
            //将worker从池子中移除..
            workers.remove(w);
        } finally {
            //释放全局锁
            mainLock.unlock();
        }


        tryTerminate();

        //获取最新ctl值
        int c = ctl.get();
        //条件成立：当前线程池状态为 RUNNING 或者 SHUTDOWN状态
        if (runStateLessThan(c, STOP)) {

            //条件成立：当前线程是正常退出..
            if (!completedAbruptly) {

                //min表示线程池最低持有的线程数量
                //allowCoreThreadTimeOut == true => 说明核心线程数内的线程，也会超时被回收。 min == 0
                //allowCoreThreadTimeOut == false => min == corePoolSize
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;


                //线程池状态：RUNNING SHUTDOWN
                //条件一：假设min == 0 成立
                //条件二：! workQueue.isEmpty() 说明任务队列中还有任务，最起码要留一个线程。
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;

                //条件成立：线程池中还拥有足够的线程。
                //考虑一个问题： workerCountOf(c) >= min  =>  (0 >= 0) ?
                //有可能！
                //什么情况下？ 当线程池中的核心线程数是可以被回收的情况下，会出现这种情况，这种情况下，当前线程池中的线程数 会变为0
                //下次再提交任务时，会再创建线程。
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }

            //1.当前线程在执行task时 发生异常，这里一定要创建一个新worker顶上去。
            //2.!workQueue.isEmpty() 说明任务队列中还有任务，最起码要留一个线程。 当前状态为 RUNNING || SHUTDOWN
            //3.当前线程数量 < corePoolSize值，此时会创建线程，维护线程池数量在corePoolSize个。
            addWorker(null, false);
        }
    }
```

## shutdown源码解析

```java
 public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        //获取线程池全局锁
        mainLock.lock();
        try {
            checkShutdownAccess();
            //设置线程池状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            //中断空闲线程
            interruptIdleWorkers();
            //空方法，子类可以扩展
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            //释放线程池全局锁
            mainLock.unlock();
        }
        //回头说..
        tryTerminate();
    }
```

## tryTerminate源码解析

```java
   final void tryTerminate() {
        //自旋
        for (;;) {
            //获取最新ctl值
            int c = ctl.get();
            //条件一：isRunning(c)  成立，直接返回就行，线程池很正常！
            //条件二：runStateAtLeast(c, TIDYING) 说明 已经有其它线程 在执行 TIDYING -> TERMINATED状态了,当前线程直接回去。
            //条件三：(runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())
            //SHUTDOWN特殊情况，如果是这种情况，直接回去。得等队列中的任务处理完毕后，再转化状态。
            if (isRunning(c) ||
                    runStateAtLeast(c, TIDYING) ||
                    (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;

            //什么情况会执行到这里？
            //1.线程池状态 >= STOP
            //2.线程池状态为 SHUTDOWN 且 队列已经空了

            //条件成立：当前线程池中的线程数量 > 0
            if (workerCountOf(c) != 0) { // Eligible to terminate
                //中断一个空闲线程。
                //空闲线程，在哪空闲呢？ queue.take() | queue.poll()
                //1.唤醒后的线程 会在getTask()方法返回null
                //2.执行退出逻辑的时候会再次调用tryTerminate() 唤醒下一个空闲线程
                //3.因为线程池状态是 （线程池状态 >= STOP || 线程池状态为 SHUTDOWN 且 队列已经空了） 最终调用addWorker时，会失败。
                //最终空闲线程都会在这里退出，非空闲线程 当执行完当前task时，也会调用tryTerminate方法，有可能会走到这里。
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            //执行到这里的线程是谁？
            //workerCountOf(c) == 0 时，会来到这里。
            //最后一个退出的线程。 咱们知道，在 （线程池状态 >= STOP || 线程池状态为 SHUTDOWN 且 队列已经空了）
            //线程唤醒后，都会执行退出逻辑，退出过程中 会 先将 workerCount计数 -1 => ctl -1。
            //调用tryTerminate 方法之前，已经减过了，所以0时，表示这是最后一个退出的线程了。

            final ReentrantLock mainLock = this.mainLock;
            //获取线程池全局锁
            mainLock.lock();
            try {
                //设置线程池状态为TIDYING状态。
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //调用钩子方法
                        terminated();
                    } finally {
                        //设置线程池状态为TERMINATED状态。
                        ctl.set(ctlOf(TERMINATED, 0));
                        //唤醒调用 awaitTermination() 方法的线程。
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                //释放线程池全局锁。
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

## interruptWorkers源码解析

```java
 private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        //获取线程池全局锁
        mainLock.lock();
        try {
            //遍历所有worker
            for (Worker w : workers)
                //interruptIfStarted() 如果worker内的thread 是启动状态，则给它一个中断信号。。
                w.interruptIfStarted();
        } finally {
            //释放线程池全局锁
            mainLock.unlock();
        }
    }
```

## interruptIdleWorkers源码解析

```java
 //onlyOne == true 说明只中断一个线程 ，false 则中断所有线程
    //共同前提，worker是空闲状态。
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        //持有全局锁
        mainLock.lock();
        try {
            //迭代所有worker
            for (Worker w : workers) {
                //获取当前worker的线程 保存到t
                Thread t = w.thread;
                //条件一：条件成立：!t.isInterrupted()  == true  说明当前迭代的这个线程尚未中断。
                //条件二：w.tryLock() 条件成立：说明当前worker处于空闲状态，可以去给它一个中断信号。 目前worker内的线程 在 queue.take() | queue.poll()
                //阻塞中。因为worker执行task时，是加锁的!
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        //给当前线程中断信号..处于queue阻塞的线程，会被唤醒，唤醒后，进入下一次自旋时，可能会return null。执行退出相关的逻辑。
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        //释放worker的独占锁。
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }

        } finally {
            //释放全局锁。
            mainLock.unlock();
        }
    }

```

# AbstractQueuedSynchronizer源码解析

## 独占模式和AQS条件队列

![image-20210409161034283](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210409161034283.png)

![image-20210409161054258](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210409161054258.png)

sync同步锁关键词管程模型大致原理

![image-20210409161131624](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210409161131624.png)

## Node内部类源码解析

```java
  static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        //枚举：共享模式
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        //枚举：独占模式
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        //表示当前节点处于 取消 状态
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        //注释：表示当前节点需要唤醒他的后继节点。（SIGNAL 表示其实是 后继节点的状态，需要当前节点去喊它...）
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        //先不说...
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        //先不说...
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        //node状态，可选值（0 ， SIGNAl（-1）, CANCELLED（1）, CONDITION, PROPAGATE）
        // waitStatus == 0  默认状态
        // waitStatus > 0 取消状态
        // waitStatus == -1 表示当前node如果是head节点时，释放锁之后，需要唤醒它的后继节点。
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        //因为node需要构建成  fifo 队列， 所以 prev 指向 前继节点
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        //因为node需要构建成  fifo 队列， 所以 next 指向 后继节点
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        //当前node封装的 线程本尊..
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        //reentrantLock 未用到...先不说..
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

```

## 成员变量解析

```java
 //头结点 任何时刻 头结点对应的线程都是当前持锁线程。
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    //阻塞队列的尾节点   (阻塞队列不包含 头结点  head.next --->  tail 认为是阻塞队列)
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    //表示资源
    //独占模式：0 表示未加锁状态   >0 表示已经加锁状态
    private volatile int state;
```

## 内部类ConditionObject源码解析

### 成员变量解析

```java
 /** First node of condition queue. */
        //指向条件队列的第一个node节点
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        //指向条件队列的最后一个node节点
        private transient Node lastWaiter;
```

### addConditionWaiter方法源码解析

```java
 private Node addConditionWaiter() {
            //获取当前条件队列的尾节点的引用 保存到局部变量 t中
            Node t = lastWaiter;

            //条件一：t != null 成立：说明当前条件队列中，已经有node元素了..
            //条件二：node 在 条件队列中时，它的状态是 CONDITION（-2）
            //     t.waitStatus != Node.CONDITION 成立：说明当前node发生中断了..
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                //清理条件队列中所有取消状态的节点
                unlinkCancelledWaiters();
                //更新局部变量t 为最新队尾引用，因为上面unlinkCancelledWaiters可能会更改lastWaiter引用。
                t = lastWaiter;
            }

            //为当前线程创建node节点，设置状态为 CONDITION(-2)
            Node node = new Node(Thread.currentThread(), Node.CONDITION);

            //条件成立：说明条件队列中没有任何元素，当前线程是第一个进入队列的元素。
            //让firstWaiter 指向当前node
            if (t == null)
                firstWaiter = node;
            else//说明当前条件队列已经有其它node了 ，做追加操作
                t.nextWaiter = node;


            //更新队尾引用指向 当前node。
            lastWaiter = node;
            //返回当前线程的node
            return node;
        }
```

### doSignal方法源码解析

```java
 private void doSignal(Node first) {
            do {
                //firstWaiter = first.nextWaiter 因为当前first马上要出条件队列了，
                //所以更新firstWaiter为 当前节点的下一个节点..
                //如果当前节点的下一个节点 是 null，说明条件队列只有当前一个节点了...当前出队后，整个队列就空了..
                //所以需要更新lastWaiter = null
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;


                //当前first节点 出 条件队列。断开和下一个节点的关系.
                first.nextWaiter = null;


                //transferForSignal(first)
                //boolean：true 当前first节点迁移到阻塞队列成功  false 迁移失败...
                //while循环 ：(first = firstWaiter) != null  当前first迁移失败，则将first更新为 first.next 继续尝试迁移..
                //直至迁移某个节点成功，或者 条件队列为null为止。
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

### unlinkCancelledWaiters源码解析

```java
  private void unlinkCancelledWaiters() {
            //表示循环当前节点，从链表的第一个节点开始 向后迭代处理.
            Node t = firstWaiter;
            //当前链表上一个正常状态的node节点
            Node trail = null;

            while (t != null) {
                //当前节点的下一个节点.
                Node next = t.nextWaiter;
                //条件成立：说明当前节点状态为 取消状态
                if (t.waitStatus != Node.CONDITION) {
                    //更新nextWaiter为null
                    t.nextWaiter = null;
                    //条件成立：说明遍历到的节点还未碰到过正常节点..
                    if (trail == null)
                        //更新firstWaiter指针为下个节点就可以
                        firstWaiter = next;
                    else
                        //让上一个正常节点指向 取消节点的 下一个节点..中间有问题的节点 被跳过去了..
                        trail.nextWaiter = next;

                    //条件成立：当前节点为队尾节点了，更新lastWaiter 指向最后一个正常节点 就Ok了
                    if (next == null)
                        lastWaiter = trail;
                }
                else//条件不成立执行到else，说明当前节点是正常节点
                    trail = t;

                t = next;
            }
        }
```

### signal源码解析

```java
public final void signal() {
            //判断调用signal方法的线程是否是独占锁持有线程，如果不是，直接抛出异常..
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();


            //获取条件队列的一个node
            Node first = firstWaiter;
            //第一个节点不为null，则将第一个节点 进行迁移到 阻塞队列的逻辑..
            if (first != null)
                doSignal(first);



        }
```

### reportInterruptAfterWait源码解析



```java
  private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            //条件成立：说明在条件队列内发生过中断，此时await方法抛出中断异常
            if (interruptMode == THROW_IE)
                throw new InterruptedException();

            //条件成立：说明在条件队列外发生的中断，此时设置当前线程的中断标记位 为true
            //中断处理 交给 你的业务处理。 如果你不处理，那什么事 也不会发生了...
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```

### await源码解析

```java
  public final void await() throws InterruptedException {
            //判断当前线程是否是中断状态，如果是则直接给个中断异常了..
            if (Thread.interrupted())
                throw new InterruptedException();

            //将调用await方法的线程包装成为node并且加入到条件队列中，并返回当前线程的node。
            Node node = addConditionWaiter();
            //完全释放掉当前线程对应的锁（将state置为0）
            //为什么要释放锁呢？  加着锁 挂起后，谁还能救你呢？
            int savedState = fullyRelease(node);
            //0 在condition队列挂起期间未接收过过中断信号
            //-1 在condition队列挂起期间接收到中断信号了
            //1 在condition队列挂起期间为未接收到中断信号，但是迁移到“阻塞队列”之后 接收过中断信号。
            int interruptMode = 0;

            //isOnSyncQueue 返回true 表示当前线程对应的node已经迁移到 “阻塞队列” 了
            //返回false 说明当前node仍然还在 条件队列中，需要继续park！
            while (!isOnSyncQueue(node)) {
                //挂起当前node对应的线程。  接下来去看signal过程...
                LockSupport.park(this);
                //什么时候会被唤醒？都有几种情况呢？
                //1.常规路径：外部线程获取到lock之后，调用了 signal()方法 转移条件队列的头节点到 阻塞队列， 当这个节点获取到锁后，会唤醒。
                //2.转移至阻塞队列后，发现阻塞队列中的前驱节点状态 是 取消状态，此时会唤醒当前节点
                //3.当前节点挂起期间，被外部线程使用中断唤醒..


                //checkInterruptWhileWaiting ：就算在condition队列挂起期间 线程发生中断了，对应的node也会被迁移到 “阻塞队列”。
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }

            //执行到这里，就说明 当前node已经迁移到 “阻塞队列”了


            //acquireQueued ：竞争队列的逻辑..
            //条件一：返回true 表示在阻塞队列中 被外部线程中断唤醒过..
            //条件二：interruptMode != THROW_IE 成立，说明当前node在条件队列内 未发生过中断
            //设置interruptMode = REINTERRUPT
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;


            //考虑下 node.nextWaiter != null 条件什么时候成立呢？
            //其实是node在条件队列内时 如果被外部线程 中断唤醒时，会加入到阻塞队列，但是并未设置nextWaiter = null。
            if (node.nextWaiter != null) // clean up if cancelled
                //清理条件队列内取消状态的节点..
                unlinkCancelledWaiters();

            //条件成立：说明挂起期间 发生过中断（1.条件队列内的挂起 2.条件队列之外的挂起）
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

## checkInterruptWhileWaiting源码解析

```java
  private int checkInterruptWhileWaiting(Node node) {
            //Thread.interrupted() 返回当前线程中断标记位，并且重置当前标记位 为 false 。
            return Thread.interrupted() ?
                    //transferAfterCancelledWait 这个方法只有在线程是被中断唤醒时 才会调用！
                    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                    0;
        }
    private void reportInterruptAfterWait(int interruptMode)
                throws InterruptedException {
            //条件成立：说明在条件队列内发生过中断，此时await方法抛出中断异常
            if (interruptMode == THROW_IE)
                throw new InterruptedException();

                //条件成立：说明在条件队列外发生的中断，此时设置当前线程的中断标记位 为true
                //中断处理 交给 你的业务处理。 如果你不处理，那什么事 也不会发生了...
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```



## CancelAcquire源码解析

```java
 /**
         * 取消指定node参与竞争。
         */
        private void cancelAcquire(Node node) {
            //空判断..
            if (node == null)
                return;

            //因为已经取消排队了..所以node内部关联的当前线程，置为Null就好了。。
            node.thread = null;

            //获取当前取消排队node的前驱。
            Node pred = node.prev;

            while (pred.waitStatus > 0)
                node.prev = pred = pred.prev;

            //拿到前驱的后继节点。
            //1.当前node
            //2.可能也是 ws > 0 的节点。
            Node predNext = pred.next;

            //将当前node状态设置为 取消状态  1
            node.waitStatus = Node.CANCELLED;



            /**
             * 当前取消排队的node所在 队列的位置不同，执行的出队策略是不一样的，一共分为三种情况：
             * 1.当前node是队尾  tail -> node
             * 2.当前node 不是 head.next 节点，也不是 tail
             * 3.当前node 是 head.next节点。
             */


            //条件一：node == tail  成立：当前node是队尾  tail -> node
            //条件二：compareAndSetTail(node, pred) 成功的话，说明修改tail完成。
            if (node == tail && compareAndSetTail(node, pred)) {
                //修改pred.next -> null. 完成node出队。
                compareAndSetNext(pred, predNext, null);

            } else {


                //保存节点 状态..
                int ws;

                //第二种情况：当前node 不是 head.next 节点，也不是 tail
                //条件一：pred != head 成立， 说明当前node 不是 head.next 节点，也不是 tail
                //条件二： ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL)))
                //条件2.1：(ws = pred.waitStatus) == Node.SIGNAL   成立：说明node的前驱状态是 Signal 状态   不成立：前驱状态可能是0 ，
                // 极端情况下：前驱也取消排队了..
                //条件2.2:(ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))
                // 假设前驱状态是 <= 0 则设置前驱状态为 Signal状态..表示要唤醒后继节点。
                //if里面做的事情，就是让pred.next -> node.next  ,所以需要保证pred节点状态为 Signal状态。
                if (pred != head &&
                        ((ws = pred.waitStatus) == Node.SIGNAL ||
                                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                        pred.thread != null) {
                    //情况2：当前node 不是 head.next 节点，也不是 tail
                    //出队：pred.next -> node.next 节点后，当node.next节点 被唤醒后
                    //调用 shouldParkAfterFailedAcquire 会让node.next 节点越过取消状态的节点
                    //完成真正出队。
                    Node next = node.next;
                    if (next != null && next.waitStatus <= 0)
                        compareAndSetNext(pred, predNext, next);

                } else {
                    //当前node 是 head.next节点。  更迷了...
                    //类似情况2，后继节点唤醒后，会调用 shouldParkAfterFailedAcquire 会让node.next 节点越过取消状态的节点
                    //队列的第三个节点 会 直接 与 head 建立 双重指向的关系：
                    //head.next -> 第三个node  中间就是被出队的head.next 第三个node.prev -> head
                    unparkSuccessor(node);
                }

                node.next = node; // help GC
            }
        }
```

## release方法源码解析

```java
   //AQS#release方法
        //ReentrantLock.unlock() -> sync.release()【AQS提供的release】
        public final boolean release(int arg) {
            //尝试释放锁，tryRelease 返回true 表示当前线程已经完全释放锁
            //返回false，说明当前线程尚未完全释放锁..
            if (tryRelease(arg)) {

                //head什么情况下会被创建出来？
                //当持锁线程未释放线程时，且持锁期间 有其它线程想要获取锁时，其它线程发现获取不了锁，而且队列是空队列，此时后续线程会为当前持锁中的
                //线程 构建出来一个head节点，然后后续线程  会追加到 head 节点后面。
                Node h = head;

                //条件一:成立，说明队列中的head节点已经初始化过了，ReentrantLock 在使用期间 发生过 多线程竞争了...
                //条件二：条件成立，说明当前head后面一定插入过node节点。
                if (h != null && h.waitStatus != 0)
                    //唤醒后继节点..
                    unparkSuccessor(h);
                return true;
            }

            return false;
        }

```

## unparkSuccessor源码解析

```java
   /**
         * 唤醒当前节点的下一个节点。
         */
        private void unparkSuccessor(Node node) {
            //获取当前节点的状态
            int ws = node.waitStatus;

            if (ws < 0)//-1 Signal  改成零的原因：因为当前节点已经完成喊后继节点的任务了..
                compareAndSetWaitStatus(node, ws, 0);

            //s是当前节点 的第一个后继节点。
            Node s = node.next;

            //条件一：
            //s 什么时候等于null？
            //1.当前节点就是tail节点时  s == null。
            //2.当新节点入队未完成时（1.设置新节点的prev 指向pred  2.cas设置新节点为tail   3.（未完成）pred.next -> 新节点 ）
            //需要找到可以被唤醒的节点..

            //条件二：s.waitStatus > 0    前提：s ！= null
            //成立：说明 当前node节点的后继节点是 取消状态... 需要找一个合适的可以被唤醒的节点..
            if (s == null || s.waitStatus > 0) {
                //查找可以被唤醒的节点...
                s = null;
                for (Node t = tail; t != null && t != node; t = t.prev)
                    if (t.waitStatus <= 0)
                        s = t;

              //上面循环，会找到一个离当前node最近的一个可以被唤醒的node。 node 可能找不到  node 有可能是null、、
            }


            //如果找到合适的可以被唤醒的node，则唤醒.. 找不到 啥也不做。
            if (s != null)
                LockSupport.unpark(s.thread);
        }
```

## acquire源码解析

```java
   public final void acquire(int arg) {
            //条件一：!tryAcquire 尝试获取锁 获取成功返回true  获取失败 返回false。
            //条件二：2.1：addWaiter 将当前线程封装成node入队
            //       2.2：acquireQueued 挂起当前线程   唤醒后相关的逻辑..
            //      acquireQueued 返回true 表示挂起过程中线程被中断唤醒过..  false 表示未被中断过..
            if (!tryAcquire(arg) &&
                    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                //再次设置中断标记位 true
                selfInterrupt();
        }
```

## acquireQueued源码解析

```java
 //acquireQueued 需要做什么呢？
        //1.当前节点有没有被park? 挂起？ 没有  ==> 挂起的操作
        //2.唤醒之后的逻辑在哪呢？   ==> 唤醒之后的逻辑。
        //AQS#acquireQueued
        //参数一：node 就是当前线程包装出来的node，且当前时刻 已经入队成功了..
        //参数二：当前线程抢占资源成功后，设置state值时 会用到。
        final boolean acquireQueued(final Node node, int arg) {
            //true 表示当前线程抢占锁成功，普通情况下【lock】 当前线程早晚会拿到锁..
            //false 表示失败，需要执行出队的逻辑... （回头讲 响应中断的lock方法时再讲。）
            boolean failed = true;
            try {
                //当前线程是否被中断
                boolean interrupted = false;
                //自旋..
                for (;;) {


                    //什么时候会执行这里？
                    //1.进入for循环时 在线程尚未park前会执行
                    //2.线程park之后 被唤醒后，也会执行这里...


                    //获取当前节点的前置节点..
                    final Node p = node.predecessor();
                    //条件一成立：p == head  说明当前节点为head.next节点，head.next节点在任何时候 都有权利去争夺锁.
                    //条件二：tryAcquire(arg)
                    //成立：说明head对应的线程 已经释放锁了，head.next节点对应的线程，正好获取到锁了..
                    //不成立：说明head对应的线程  还没释放锁呢...head.next仍然需要被park。。
                    if (p == head && tryAcquire(arg)) {
                        //拿到锁之后需要做什么？
                        //设置自己为head节点。
                        setHead(node);
                        //将上个线程对应的node的next引用置为null。协助老的head出队..
                        p.next = null; // help GC
                        //当前线程 获取锁 过程中..没有异常
                        failed = false;
                        //返回当前线程的中断标记..
                        return interrupted;
                    }

                    //shouldParkAfterFailedAcquire  这个方法是干嘛的？ 当前线程获取锁资源失败后，是否需要挂起呢？
                    //返回值：true -> 当前线程需要 挂起    false -> 不需要..
                    //parkAndCheckInterrupt()  这个方法什么作用？ 挂起当前线程，并且唤醒之后 返回 当前线程的 中断标记
                    // （唤醒：1.正常唤醒 其它线程 unpark 2.其它线程给当前挂起的线程 一个中断信号..）
                    if (shouldParkAfterFailedAcquire(p, node) &&
                            parkAndCheckInterrupt())
                        //interrupted == true 表示当前node对应的线程是被 中断信号唤醒的...
                        interrupted = true;
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }

```

## parkAndCheckInterrupt方法源码解析

```java
 //park当前线程 将当前线程 挂起，唤醒后返回当前线程 是否为 中断信号 唤醒。
        private final boolean parkAndCheckInterrupt() {
            LockSupport.park(this);
            return Thread.interrupted();
        }
```

## shouldParkAfterFailedAcquire方法源码解析

```java
 /**
         * 总结：
         * 1.当前节点的前置节点是 取消状态 ，第一次来到这个方法时 会越过 取消状态的节点， 第二次 会返回true 然后park当前线程
         * 2.当前节点的前置节点状态是0，当前线程会设置前置节点的状态为 -1 ，第二次自旋来到这个方法时  会返回true 然后park当前线程.
         *
         * 参数一：pred 当前线程node的前置节点
         * 参数二：node 当前线程对应node
         * 返回值：boolean  true 表示当前线程需要挂起..
         */
        private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
            //获取前置节点的状态
            //waitStatus：0 默认状态 new Node() ； -1 Signal状态，表示当前节点释放锁之后会唤醒它的第一个后继节点； >0 表示当前节点是CANCELED状态
            int ws = pred.waitStatus;
            //条件成立：表示前置节点是个可以唤醒当前节点的节点，所以返回true ==> parkAndCheckInterrupt() park当前线程了..
            //普通情况下，第一次来到shouldPark。。。 ws 不会是 -1
            if (ws == Node.SIGNAL)
                return true;

            //条件成立： >0 表示前置节点是CANCELED状态
            if (ws > 0) {
                //找爸爸的过程，条件是什么呢？ 前置节点的 waitStatus <= 0 的情况。
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                //找到好爸爸后，退出循环
                //隐含着一种操作，CANCELED状态的节点会被出队。
                pred.next = node;

            } else {
                //当前node前置节点的状态就是 0 的这一种情况。
                //将当前线程node的前置node，状态强制设置为 SIGNAl，表示前置节点释放锁之后需要 喊醒我..
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
        }
```

## addWaiter方法源码解析

```java
 //最终返回当前线程包装出来的node
        private Node addWaiter(Node mode) {
            //Node.EXCLUSIVE
            //构建Node ，把当前线程封装到对象node中了
            Node node = new Node(Thread.currentThread(), mode);
            // Try the fast path of enq; backup to full enq on failure
            //快速入队
            //获取队尾节点 保存到pred变量中
            Node pred = tail;
            //条件成立：队列中已经有node了
            if (pred != null) {
                //当前节点的prev 指向 pred
                node.prev = pred;
                //cas成功，说明node入队成功
                if (compareAndSetTail(pred, node)) {
                    //前置节点指向当前node，完成 双向绑定。
                    pred.next = node;
                    return node;
                }
            }

            //什么时候会执行到这里呢？
            //1.当前队列是空队列  tail == null
            //2.CAS竞争入队失败..会来到这里..

            //完整入队..
            enq(node);

            return node;
        }

```

## enq方法源码解析

```java
  //返回值：返回当前节点的 前置节点。
        private Node enq(final Node node) {
            //自旋入队，只有当前node入队成功后，才会跳出循环。
            for (;;) {
                Node t = tail;
                //1.当前队列是空队列  tail == null
                //说明当前 锁被占用，且当前线程 有可能是第一个获取锁失败的线程（当前时刻可能存在一批获取锁失败的线程...）
                if (t == null) { // Must initialize
                    //作为当前持锁线程的 第一个 后继线程，需要做什么事？
                    //1.因为当前持锁的线程，它获取锁时，直接tryAcquire成功了，没有向 阻塞队列 中添加任何node，所以作为后继需要为它擦屁股..
                    //2.为自己追加node

                    //CAS成功，说明当前线程 成为head.next节点。
                    //线程需要为当前持锁的线程 创建head。
                    if (compareAndSetHead(new Node()))
                        tail = head;

                    //注意：这里没有return,会继续for。。
                } else {
                    //普通入队方式，只不过在for中，会保证一定入队成功！
                    node.prev = t;
                    if (compareAndSetTail(t, node)) {
                        t.next = node;
                        return t;
                    }
                }
            }
        }
```

## isSyncQueue源码解析

```java
 final boolean isOnSyncQueue(Node node) {
        //条件一：node.waitStatus == Node.CONDITION 条件成立，说明当前node一定是在
        //条件队列，因为signal方法迁移节点到 阻塞队列前，会将node的状态设置为 0
        //条件二：前置条件：node.waitStatus != Node.CONDITION   ===>
        // 1.node.waitStatus == 0 (表示当前节点已经被signal了)
        // 2.node.waitStatus == 1 （当前线程是未持有锁调用await方法..最终会将node的状态修改为 取消状态..）
        //node.waitStatus == 0 为什么还要判断 node.prev == null?
        //因为signal方法 是先修改状态，再迁移。
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;

        //执行到这里，会是哪种情况？
        //node.waitStatus != CONDITION 且 node.prev != null  ===> 可以排除掉 node.waitStatus == 1 取消状态..
        //为什么可以排除取消状态？ 因为signal方法是不会把 取消状态的node迁移走的
        //设置prev引用的逻辑 是 迁移 阻塞队列 逻辑的设置的（enq()）
        //入队的逻辑：1.设置node.prev = tail;   2. cas当前node为 阻塞队列的 tail 尾节点 成功才算是真正进入到 阻塞队列！ 3.pred.next = node;
        //可以推算出，就算prev不是null，也不能说明当前node 已经成功入队到 阻塞队列了。


        //条件成立：说明当前节点已经成功入队到阻塞队列，且当前节点后面已经有其它node了...
        if (node.next != null) // If has successor, it must be on queue
            return true;


        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        /**
         * 执行到这里，说明当前节点的状态为：node.prev != null 且 node.waitStatus == 0
         * findNodeFromTail 从阻塞队列的尾巴开始向前遍历查找node，如果查找到 返回true,查找不到返回false
         * 当前node有可能正在signal过程中，正在迁移中...还未完成...
         */
        return findNodeFromTail(node);
    }
```

## transferForSignal源码解析

```java
 final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        //cas修改当前节点的状态，修改为0，因为当前节点马上要迁移到 阻塞队列了
        //成功：当前节点在条件队列中状态正常。
        //失败：1.取消状态 （线程await时 未持有锁，最终线程对应的node会设置为 取消状态）
        //     2.node对应的线程 挂起期间，被其它线程使用 中断信号 唤醒过...（就会主队进入到 阻塞队列，这时也会修改状态为0）
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        //enq最终会将当前 node 入队到 阻塞队列，p 是当前节点在阻塞队列的 前驱节点.
        Node p = enq(node);

        //ws 前驱节点的状态..
        int ws = p.waitStatus;
        //条件一：ws > 0 成立：说明前驱节点的状态在阻塞队列中是 取消状态,唤醒当前节点。
        //条件二：前置条件(ws <= 0)，
        //compareAndSetWaitStatus(p, ws, Node.SIGNAL) 返回true 表示设置前驱节点状态为 SIGNAl状态成功
        //compareAndSetWaitStatus(p, ws, Node.SIGNAL) 返回false  ===> 什么时候会false?
        //当前驱node对应的线程 是 lockInterrupt入队的node时，是会响应中断的，外部线程给前驱线程中断信号之后，前驱node会将
        //状态修改为 取消状态，并且执行 出队逻辑..
        //前驱节点状态 只要不是 0 或者 -1 那么，就唤醒当前线程。
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            //唤醒当前node对应的线程...回头再说。
            LockSupport.unpark(node.thread);

        return true;
    }
```

## transferAfterCancelledWaiter源码解析

```java
   */
    final boolean transferAfterCancelledWait(Node node) {
        //条件成立：说明当前node一定是在 条件队列内，因为signal 迁移节点到阻塞队列时，会将节点的状态修改为0
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            //中断唤醒的node也会被加入到 阻塞队列中！！
            enq(node);
            //true：表示是在条件队列内被中断的.
            return true;
        }

        //执行到这里有几种情况？
        //1.当前node已经被外部线程调用 signal 方法将其迁移到 阻塞队列内了。
        //2.当前node正在被外部线程调用 signal 方法将其迁移至 阻塞队列中 进行中状态..

        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        while (!isOnSyncQueue(node))
            Thread.yield();

        //false:表示当前节点被中断唤醒时 不在 条件队列了..
        return false;
    }
```

## fulyRelease源码解析

```java
 final int fullyRelease(Node node) {
        //完全释放锁是否成功，当failed失败时，说明当前线程是未持有锁调用 await方法的线程..（错误写法..）
        //假设失败，在finally代码块中 会将刚刚加入到 条件队列的 当前线程对应的node状态 修改为 取消状态
        //后继线程就会将 取消状态的 节点 给清理出去了..
        boolean failed = true;
        try {
            //获取当前线程 所持有的 state值 总数！
            int savedState = getState();

            //绝大部分情况下：release 这里会返回true。
            if (release(savedState)) {
                //失败标记设置为false
                failed = false;
                //返回当前线程释放的state值
                //为什么要返回savedState？
                //因为在当你被迁移到“阻塞队列”后，再次被唤醒，且当前node在阻塞队列中是head.next 而且
                //当前lock状态是state == 0 的情况下，当前node可以获取到锁，此时需要将state 设置为savedState.
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```



# ReentrantLock源码

## tryRelease方法源码解析

```java
  protected final boolean tryRelease(int releases) {
            //减去释放的值..
            int c = getState() - releases;
            //条件成立：说明当前线程并未持锁..直接异常.,.
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();

            //当前线程持有锁..


            //是否已经完全释放锁..默认false
            boolean free = false;
            //条件成立：说明当前线程已经达到完全释放锁的条件。 c == 0
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            //更新AQS.state值
            setState(c);
            return free;
        }
```

## tryAcquire方法源码解析

```java
  /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         * 抢占成功：返回true  包含重入..
         * 抢占失败：返回false
         */
        protected final boolean tryAcquire(int acquires) {
            //current 当前线程
            final Thread current = Thread.currentThread();
            //AQS state 值
            int c = getState();
            //条件成立：c == 0 表示当前AQS处于无锁状态..
            if (c == 0) {
                //条件一：
                //因为fairSync是公平锁，任何时候都需要检查一下 队列中是否在当前线程之前有等待者..
                //hasQueuedPredecessors() 方法返回 true 表示当前线程前面有等待者，当前线程需要入队等待
                //hasQueuedPredecessors() 方法返回 false 表示当前线程前面无等待者，直接尝试获取锁..

                //条件二：compareAndSetState(0, acquires)
                //成功：说明当前线程抢占锁成功
                //失败：说明存在竞争，且当前线程竞争失败..
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    //成功之后需要做什么？
                    //设置当前线程为 独占者 线程。
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //执行到这里，有几种情况？
            //c != 0 大于0 的情况，这种情况就需要检查一下 当前线程是不是 独占锁的线程，因为ReentrantLock是可以重入的.

            //条件成立：说明当前线程就是独占锁线程..
            else if (current == getExclusiveOwnerThread()) {
                //锁重入的逻辑..

                //nextc 更新值..
                int nextc = c + acquires;
                //越界判断，当重入的深度很深时，会导致 nextc < 0 ，int值达到最大之后 再 + 1 ...变负数..
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //更新的操作
                setState(nextc);
                return true;
            }

            //执行到这里？
            //1.CAS失败  c == 0 时，CAS修改 state 时 未抢过其他线程...
            //2.c > 0 且 ownerThread != currentThread.
            return false;
        }
```

# 手写阻塞队列

```java
package com.xiaoliu.niubility;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * ClassName: MiniArrayBrokingQueue
 * Description:
 * date: 2020/4/6 20:56
 *
 * @author 小刘讲师，微信：vv517956494
 * 本课程属于 小刘讲师 VIP 源码特训班课程
 * 严禁非法盗用（如有发现非法盗取行为，必将追究法律责任）
 * <p>
 * 如有同学发现非 小刘讲源码 官方号传播本视频资源，请联系我！
 * @since 1.0.0
 */
public class MiniArrayBrokingQueue implements BrokingQueue {
    //线程并发控制
    private Lock lock = new ReentrantLock();
    /**
     * 当生产者线程生产数据时，它会先检查当前queues是否已经满了，如果已经满了，需要将当前生产者线程 调用notFull.await()
     * 进入到notFull条件队列挂起。等待消费者线程消费数据时唤醒。
     */
    private Condition notFull = lock.newCondition();

    /**
     * 当消费者线程消费数据时，它会先检查当前queues中是否有数据，如果没有数据,需要将当前消费者线程 调用notEmpty.await()
     * 进入到notEmpty条件队列挂起。等待生产者线程生产数据时唤醒。
     */
    private Condition notEmpty = lock.newCondition();


    //底层存储元素的数组
    private Object[] queues;
    //数组长度
    private int size;

    /**
     * count:当前队列中可以被消费的数据量
     * putptr:记录生产者存放数据的下一次位置。每个生产者生产完一个数据后，会将 putptr ++
     * takeptr:记录消费者消费数据的下一次的位置。每个消费者消费完一个数据后，将将takeptr ++
     */
    private int count,putptr, takeptr;


    public MiniArrayBrokingQueue(int size) {
        this.size = size;
        this.queues = new Object[size];
    }



    @Override
    public void put(Object element) throws InterruptedException {
        lock.lock();
        try {
            //第一件事？ 判断一下当前queues是否已经满了...
            if(count == this.size) {
                notFull.await();
            }

            //执行到这里，说明队列未满，可以向队列中存放数据了..
            this.queues[putptr] = element;

            putptr ++;

            if(putptr == this.size) putptr = 0;
            //生产数据 自增count
            count ++;

            //当向队列中成功放入一个元素之后，需要做什么呢？
            //需要给notEmpty一个唤醒信号
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            //第一件事？判断一下当前队列是否有数据可以被消费...
            if(count == 0) {
                notEmpty.await();
            }

            //执行到这里，说明队列有数据可以被消费了..
            Object element = this.queues[takeptr];

            takeptr ++;
            if(takeptr == this.size) takeptr = 0;
            //生产数据 自减count
            count --;

            //当向队列中成功消费一个元素之后，需要做什么呢？
            //需要给notFull一个唤醒信号
            notFull.signal();

            return element;
        }finally {
            lock.unlock();
        }
    }


    public static void main(String[] args) {
        BrokingQueue<Integer> queue = new MiniArrayBrokingQueue(10);

        Thread producer = new Thread(() -> {
            int i = 0;
            while(true) {
                i ++;
                if(i == 10) i = 0;

                try {
                    System.out.println("生产数据：" + i);
                    queue.put(Integer.valueOf(i));
                    TimeUnit.MILLISECONDS.sleep(200);
                } catch (InterruptedException e) {
                }
            }
        });
        producer.start();


        Thread consumer = new Thread(() -> {
            while(true) {
                try {
                    Integer result = queue.take();
                    System.out.println("消费者消费：" + result);
                    TimeUnit.MILLISECONDS.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        consumer.start();
    }
}

```

# CountDownLatch源码解析

## 原理解析

<img src="D:\学习笔记\image-20210409162054809.png" alt="image-20210409162054809" style="zoom:50%;" />



<img src="D:\学习笔记\image-20210409162112979.png" alt="image-20210409162112979" style="zoom:67%;" />

<img src="D:\学习笔记\image-20210409162127883.png" alt="image-20210409162127883" style="zoom:50%;" />

## tryReleaseShared源码解析

```java
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                //获取当前AQS.state
                int c = getState();
                //条件成立：说明前面已经有线程 触发 唤醒操作了，这里返回false
                if (c == 0)
                    return false;

                //执行到这里，说明 state > 0

                int nextc = c-1;

                //cas成功，说明当前线程执行 tryReleaseShared 方法 c-1之前，没有其它线程 修改过 state。
                if (compareAndSetState(c, nextc))
                    //nextc == 0 ：true ，说明当前调用 countDown() 方法的线程 就是需要触发 唤醒操作的线程.
                    return nextc == 0;
            }
        }
```

## await方法源码解析

```java
 public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

## countDown方法源码解析

```java
  public void countDown() {
        sync.releaseShared(1);
    }
```

# CyclicBarrier源码解析

## 成员变量源码解析

```java
 /**
     * Each use of the barrier is represented as a generation instance.
     * The generation changes whenever the barrier is tripped, or
     * is reset. There can be many generations associated with threads
     * using the barrier - due to the non-deterministic way the lock
     * may be allocated to waiting threads - but only one of these
     * can be active at a time (the one to which {@code count} applies)
     * and all the rest are either broken or tripped.
     * There need not be an active generation if there has been a break
     * but no subsequent reset.
     * 表示 “代”这个概念
     */
    private static class Generation {
        //表示当前“代”是否被打破，如果代被打破 ，那么再来到这一代的线程 就会直接抛出 BrokenException异常
        //且在这一代 挂起的线程 都会被唤醒，然后抛出 BrokerException异常。
        boolean broken = false;
    }

    /** The lock for guarding barrier entry */
    //因为barrier实现是依赖于Condition条件队列的，condition条件队列必须依赖lock才能使用。
    private final ReentrantLock lock = new ReentrantLock();

    /** Condition to wait on until tripped */
    //线程挂起实现使用的 condition 队列。条件：当前代所有线程到位，这个条件队列内的线程 才会被唤醒。
    private final Condition trip = lock.newCondition();

    /** The number of parties */
    //Barrier需要参与进来的线程数量
    private final int parties;

    /* The command to run when tripped */
    //当前代 最后一个到位的线程 需要执行的事件
    private final Runnable barrierCommand;

    /** The current generation */
    //表示barrier对象 当前 “代”
    private Generation generation = new Generation();

    /**
     * Number of parties still waiting. Counts down from parties to 0
     * on each generation.  It is reset to parties on each new
     * generation or when broken.
     */
    //表示当前“代”还有多少个线程 未到位。
    //初始值为parties
    private int count;
```

## nextGeneration方法源码解析

```java
  /**
     * Updates state on barrier trip and wakes up everyone.
     * Called only while holding lock.
     * 开启下一代，当这一代 所有线程到位后（假设barrierCommand不为空，还需要最后一个线程执行完事件），会调用nextGeneration()开启新的一代。
     */
    private void nextGeneration() {
        //将在trip条件队列内挂起的线程 全部唤醒
        // signal completion of last generation
        trip.signalAll();

        //重置count为parties
        // set up next generation
        count = parties;

        //开启新的一代..使用一个新的 generation对象，表示新的一代，新的一代和上一代 没有任何关系。
        
```

## breakBarrier源码解析

```java
   /**
     * Sets current barrier generation as broken and wakes up everyone.
     * Called only while holding lock.
     * 打破barrier屏障，在屏障内的线程 都会抛出异常..
     */
    private void breakBarrier() {
        //将代中的broken设置为true，表示这一代是被打破了的，再来到这一代的线程，直接抛出异常.
        generation.broken = true;
        //重置count为parties
        count = parties;
        //将在trip条件队列内挂起的线程 全部唤醒，唤醒后的线程 会检查当前代 是否是打破的，
        //如果是打破的话，接下来的逻辑和 开启下一代 唤醒的逻辑不一样.
        trip.signalAll();
    }
```

## dowait方法源码解析

<img src="D:\学习笔记\image-20210412105643221.png" alt="image-20210412105643221" style="zoom:50%;" />

```java
  /**
     * Main barrier code, covering the various policies.
     * timed：表示当前调用await方法的线程是否指定了 超时时长，如果true 表示 线程是响应超时的
     * nanos：线程等待超时时长 纳秒，如果timed == false ,nanos == 0
     */
    private int dowait(boolean timed, long nanos)
            throws InterruptedException, BrokenBarrierException,
            TimeoutException {
        //获取barrier全局锁对象
        final ReentrantLock lock = this.lock;
        //加锁
        //为什么要加锁呢？
        //因为 barrier的挂起 和 唤醒 依赖的组件是 condition。
        lock.lock();
        try {
            //获取barrier当前的 “代”
            final Generation g = generation;

            //如果当前代是已经被打破状态，则当前调用await方法的线程，直接抛出Broken异常
            if (g.broken)
                throw new BrokenBarrierException();

            //如果当前线程的中断标记位 为 true，则打破当前代，然后当前线程抛出 中断异常
            if (Thread.interrupted()) {
                //1.设置当前代的状态为broken状态  2.唤醒在trip 条件队列内的线程
                breakBarrier();
                throw new InterruptedException();
            }

            //执行到这里，说明 当前线程中断状态是正常的 false， 当前代的broken为 false（未打破状态）
            //正常逻辑...

            //假设 parties 给的是 5，那么index对应的值为 4,3,2,1,0
            int index = --count;
            //条件成立：说明当前线程是最后一个到达barrier的线程，此时需要做什么呢？
            if (index == 0) {  // tripped
                //标记：true表示 最后一个线程 执行cmd时未抛异常。  false，表示最后一个线程执行cmd时抛出异常了.
                //cmd就是创建 barrier对象时 指定的第二个 Runnable接口实现，这个可以为null
                boolean ranAction = false;
                try {


                    final Runnable command = barrierCommand;
                    //条件成立：说明创建barrier对象时 指定 Runnable接口了，这个时候最后一个到达的线程 就需要执行这个接口
                    if (command != null)
                        command.run();

                    //command.run()未抛出异常的话，那么线程会执行到这里。
                    ranAction = true;

                    //开启新的一代
                    //1.唤醒trip条件队列内挂起的线程，被唤醒的线程 会依次 获取到lock，然后依次退出await方法。
                    //2.重置count 为 parties
                    //3.创建一个新的generation对象，表示新的一代
                    nextGeneration();
                    //返回0，因为当前线程是此 代 最后一个到达的线程，所以Index == 0
                    return 0;
                } finally {
                    if (!ranAction)
                        //如果command.run()执行抛出异常的话，会进入到这里。
                        breakBarrier();
                }
            }

            //执行到这里，说明当前线程 并不是最后一个到达Barrier的线程..此时需要进入一个自旋中.

            // loop until tripped, broken, interrupted, or timed out
            //自旋，一直到 条件满足、当前代被打破、线程被中断，等待超时
            for (;;) {
                try {
                    //条件成立：说明当前线程是不指定超时时间的
                    if (!timed)
                        //当前线程 会 释放掉lock，然后进入到trip条件队列的尾部，然后挂起自己，等待被唤醒。
                        trip.await();
                    else if (nanos > 0L)
                        //说明当前线程调用await方法时 是指定了 超时时间的！
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //抛出中断异常，会进来这里。
                    //什么时候会抛出InterruptedException异常呢？
                    //Node节点在 条件队列内 时 收到中断信号时 会抛出中断异常！


                    //条件一：g == generation 成立，说明当前代并没有变化。
                    //条件二：! g.broken 当前代如果没有被打破，那么当前线程就去打破，并且抛出异常..
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        //执行到else有几种情况？
                        //1.代发生了变化，这个时候就不需要抛出中断异常了，因为 代已经更新了，这里唤醒后就走正常逻辑了..只不过设置下 中断标记。
                        //2.代没有发生变化，但是代被打破了，此时也不用返回中断异常，执行到下面的时候会抛出  brokenBarrier异常。也记录下中断标记位。

                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                //唤醒后，执行到这里，有几种情况？
                //1.正常情况，当前barrier开启了新的一代（trip.signalAll()）
                //2.当前Generation被打破，此时也会唤醒所有在trip上挂起的线程
                //3.当前线程trip中等待超时，然后主动转移到 阻塞队列 然后获取到锁 唤醒。

                //条件成立：当前代已经被打破
                if (g.broken)
                    //线程唤醒后依次抛出BrokenBarrier异常。
                    throw new BrokenBarrierException();

                //唤醒后，执行到这里，有几种情况？
                //1.正常情况，当前barrier开启了新的一代（trip.signalAll()）
                //3.当前线程trip中等待超时，然后主动转移到 阻塞队列 然后获取到锁 唤醒。

                //条件成立：说明当前线程挂起期间，最后一个线程到位了，然后触发了开启新的一代的逻辑，此时唤醒trip条件队列内的线程。
                if (g != generation)
                    //返回当前线程的index。
                    return index;

                //唤醒后，执行到这里，有几种情况？
                //3.当前线程trip中等待超时，然后主动转移到 阻塞队列 然后获取到锁 唤醒。

                if (timed && nanos <= 0L) {
                    //打破barrier
                    breakBarrier();
                    //抛出超时异常.
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

## 构造函数源码解析

```java
 public CyclicBarrier(int parties, Runnable barrierAction) {
        //因为小于等于0 的barrier没有任何意义..
        if (parties <= 0) throw new IllegalArgumentException();

        this.parties = parties;
        //count的初始值 就是parties，后面当前代每到位一个线程，count--
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

# Semaphore源码解析

## AQS的releaseShared源码解析

```java
 public final boolean releaseShared(int arg) {
            //条件成立：表示当前线程释放资源成功，释放资源成功后，去唤醒获取资源失败的线程..
            if (tryReleaseShared(arg)) {
                //唤醒获取资源失败的线程...
                doReleaseShared();
                return true;
            }
            return false;
        }
```

## acquireSharedInterruptibly源码解析

```java
  public final void acquireSharedInterruptibly(int arg)
                throws InterruptedException {
            //条件成立：说明当前调用acquire方法的线程 已经是 中断状态了,直接抛出异常..
            if (Thread.interrupted())
                throw new InterruptedException();


            //对应业务层面 执行任务的线程已经将latch打破了。然后其他再调用latch.await的线程，就不会在这里阻塞了
            if (tryAcquireShared(arg) < 0)
                doAcquireSharedInterruptibly(arg);
        }
```

## tryAcquireShared源码解析

```java
 protected int tryAcquireShared(int acquires) {
            for (;;) {
                //判断当前 AQS 阻塞队列内 是否有等待者线程，如果有直接返回-1，表示当前aquire操作的线程需要进入到队列等待..
                if (hasQueuedPredecessors())
                    return -1;
                //执行到这里，有哪几种情况？
                //1.调用aquire时 AQS阻塞队列内没有其它等待者
                //2.当前节点 在阻塞队列内是headNext节点

                //获取state ，state这里表示 通行证
                int available = getState();
                //remaining 表示当前线程 获取通行证完成之后，semaphore还剩余数量
                int remaining = available - acquires;

                //条件一：remaining < 0 成立，说明线程获取通行证失败..
                //条件二：前置条件，remaning >= 0, CAS更新state 成功，说明线程获取通行证成功，CAS失败，则自旋。
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

# ThreadLocal源码

![弱引用](D:\学习笔记\弱引用.png)

## 成员变量解析

```java
 /**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    //线程获取threadLocal.get()时 如果是第一次在某个 threadLocal对象上get时，会给当前线程分配一个value
    //这个value 和 当前的threadLocal对象 被包装成为一个 entry 其中 key是 threadLocal对象，value是threadLocal对象给当前线程生成的value
    //这个entry存放到 当前线程 threadLocals 这个map的哪个桶位？ 与当前 threadLocal对象的threadLocalHashCode 有关系。
    // 使用 threadLocalHashCode & (table.length - 1) 的到的位置 就是当前 entry需要存放的位置。
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     * 创建ThreadLocal对象时 会使用到，每创建一个threadLocal对象 就会使用nextHashCode 分配一个hash值给这个对象。
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     * 每创建一个ThreadLocal对象，这个ThreadLocal.nextHashCode 这个值就会增长 0x61c88647 。
     * 这个值 很特殊，它是 斐波那契数  也叫 黄金分割数。hash增量为 这个数字，带来的好处就是 hash分布非常均匀。
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     * 创建新的ThreadLocal对象时  会给当前对象分配一个hash，使用这个方法。
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```

## get方法源码解析

```java
     *
     * 返回当前线程与当前ThreadLocal对象相关联的 线程局部变量，这个变量只有当前线程能访问到。
     * 如果当前线程 没有分配，则给当前线程去分配（使用initialValue方法）
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取到当前线程Thread对象的 threadLocals map引用
        ThreadLocalMap map = getMap(t);
        //条件成立：说明当前线程已经拥有自己的 ThreadLocalMap 对象了
        if (map != null) {
            //key：当前threadLocal对象
            //调用map.getEntry() 方法 获取threadLocalMap 中该threadLocal关联的 entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            //条件成立：说明当前线程 初始化过 与当前threadLocal对象相关联的 线程局部变量
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                //返回value..
                return result;
            }
        }

        //执行到这里有几种情况？
        //1.当前线程对应的threadLocalMap是空
        //2.当前线程与当前threadLocal对象没有生成过相关联的 线程局部变量..

        //setInitialValue方法初始化当前线程与当前threadLocal对象 相关联的value。
        //且 当前线程如果没有threadLocalMap的话，还会初始化创建map。
        return setInitialValue();
    }
```

## setInitialValue源码解析

```java
     * setInitialValue方法初始化当前线程与当前threadLocal对象 相关联的value。
     * 且 当前线程如果没有threadLocalMap的话，还会初始化创建map。
     * @return the initial value
     */
    private T setInitialValue() {
        //调用的当前ThreadLocal对象的initialValue方法，这个方法 大部分情况下咱们都会重写。
        //value 就是当前ThreadLocal对象与当前线程相关联的 线程局部变量。
        T value = initialValue();
        //获取当前线程对象
        Thread t = Thread.currentThread();
        //获取当前线程内部的threadLocals    threadLocalMap对象。
        ThreadLocalMap map = getMap(t);
        //条件成立：说明当前线程内部已经初始化过 threadLocalMap对象了。 （线程的threadLocals 只会初始化一次。）
        if (map != null)
            //保存当前threadLocal与当前线程生成的 线程局部变量。
            //key: 当前threadLocal对象   value：线程与当前threadLocal相关的局部变量
            map.set(this, value);
        else
            //执行到这里，说明 当前线程内部还未初始化 threadLocalMap ，这里调用createMap 给当前线程创建map

            //参数1：当前线程   参数2：线程与当前threadLocal相关的局部变量
            createMap(t, value);

        //返回线程与当前threadLocal相关的局部变量
        return value;
    }
```

## set方法源码解析

```java
  public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取当前线程的threadLocalMap对象
        ThreadLocalMap map = getMap(t);
        //条件成立：说明当前线程的threadLocalMap已经初始化过了
        if (map != null)
            //调用threadLocalMap.set方法 进行重写 或者 添加。
            map.set(this, value);
        else
            //执行到这里，说明当前线程还未创建 threadLocalMap对象。

            //参数1：当前线程   参数2：线程与当前threadLocal相关的局部变量
            createMap(t, value);
    }
```

## remove方法源码解析

```java
     * 移除当前线程与当前threadLocal对象相关联的 线程局部变量。
     *
     * @since 1.5
     */
     public void remove() {
         //获取当前线程的 threadLocalMap对象
         ThreadLocalMap m = getMap(Thread.currentThread());
         //条件成立：说明当前线程已经初始化过 threadLocalMap对象了
         if (m != null)
             //调用threadLocalMap.remove( key = 当前threadLocal)
             m.remove(this);
     }

```

## getMap源码解析

```java
   ThreadLocalMap getMap(Thread t) {
        //返回当前线程的 threadLocals
        return t.threadLocals;
    }
```

## createMap源码解析

```java
   void createMap(Thread t, T firstValue) {
        //传递t 的意义就是 要访问 当前这个线程 t.threadLocals 字段，给这个字段初始化。


        //new ThreadLocalMap(this, firstValue)
        //创建一个ThreadLocalMap对象 初始 k-v 为 ： this <当前threadLocal对象> ，线程与当前threadLocal相关的局部变量
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

# ThreadLoal内部类ThreadLocalMap源码解析

## Entry内部类源码解析

```java
   * 什么是弱引用呢？
         * A a = new A();     //强引用
         * WeakReference weakA = new WeakReference(a);  //弱引用
         *
         * a = null;
         * 下一次GC 时 对象a就被回收了，别管有没有 弱引用 是否在关联这个对象。
         *
         * key 使用的是弱引用保留，key保存的是threadLocal对象。
         * value 使用的是强引用，value保存的是 threadLocal对象与当前线程相关联的 value。
         *
         * entry#key 这样设计有什么好处呢？
         * 当threadLocal对象失去强引用且对象GC回收后，散列表中的与 threadLocal对象相关联的 entry#key 再次去key.get() 时，拿到的是null。
         * 站在map角度就可以区分出哪些entry是过期的，哪些entry是非过期的。
         *
         *
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

## ThreadLocalMap成员变量解析

负载因子0.625 采用的是线程探测法定位桶

threshold = length * 0.625

然后rehash的时候再进行一次判断

size > threshold - shreshold/4  比如16 * 0.625  = 10   10 - 10/4 = 8 那么扩容阈值就是8

```java
 /**
         * The initial capacity -- MUST be a power of two.
         * 初始化当前map内部 散列表数组的初始长度 16
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         * threadLocalMap 内部散列表数组引用，数组的长度 必须是 2的次方数
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         * 当前散列表数组 占用情况，存放多少个entry。
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         * 扩容触发阈值，初始值为： len * 2/3
         * 触发后调用 rehash() 方法。
         * rehash() 方法先做一次全量检查全局 过期数据，把散列表中所有过期的entry移除。
         * 如果移除之后 当前 散列表中的entry 个数仍然达到  threshold - threshold/4  就进行扩容。
         */
        private int threshold; // Default to 0

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         * 将阈值设置为 （当前数组长度 * 2）/ 3。
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        /**
         * Increment i modulo len.
         * 参数1：当前下标
         * 参数2：当前散列表数组长度
         */
        private static int nextIndex(int i, int len) {
            //当前下标+1 小于散列表数组的话，返回 +1后的值
            //否则 情况就是 下标+1 == len ，返回0
            //实际形成一个环绕式的访问。
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.
         * 参数1：当前下标
         * 参数2：当前散列表数组长度
         */
        private static int prevIndex(int i, int len) {
            //当前下标-1 大于等于0 返回 -1后的值就ok。
            //否则 说明 当前下标-1 == -1. 此时 返回散列表最大下标。
            //实际形成一个环绕式的访问。
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

```

## 构造函数

```java
/**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         *
         * 因为Thread.threadLocals字段是延迟初始化的，只有线程第一次存储 threadLocal-value 时 才会创建 threadLocalMap对象。
         *
         * firstKey :threadLocal对象
         * firstValue: 当前线程与threadLocal对象关联的value。
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            //创建entry数组长度为16，表示threadLocalMap内部的散列表。
            table = new Entry[INITIAL_CAPACITY];
            //寻址算法：key.threadLocalHashCode & (table.length - 1)
            //table数组的长度一定是 2 的次方数。
            //2的次方数-1 有什么特征呢？  转化为2进制后都是1.    16==> 1 0000 - 1 => 1111
            //1111 与任何数值进行&运算后 得到的数值 一定是 <= 1111

            //i 计算出来的结果 一定是 <= B1111
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            //创建entry对象 存放到 指定位置的slot中。
            table[i] = new Entry(firstKey, firstValue);
            //设置size=1
            size = 1;
            //设置扩容阈值 （当前数组长度 * 2）/ 3  => 16 * 2 / 3 => 10
            setThreshold(INITIAL_CAPACITY);
        }

```

## getEntry源码解析

```java
         * ThreadLocal对象 get() 操作 实际上是由 ThreadLocalMap.getEntry() 代理完成的。
         *
         * key:某个 ThreadLocal对象，因为 散列表中存储的entry.key 类型是 ThreadLocal。
         *
         * @param  key the thread local object
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntry(ThreadLocal<?> key) {
            //路由规则： ThreadLocal.threadLocalHashCode & (table.length - 1) ==》 index
            int i = key.threadLocalHashCode & (table.length - 1);
            //访问散列表中 指定指定位置的 slot
            Entry e = table[i];
            //条件一：成立 说明slot有值
            //条件二：成立 说明 entry#key 与当前查询的key一致，返回当前entry 给上层就可以了。
            if (e != null && e.get() == key)
                return e;
            else
                //有几种情况会执行到这里？
                //1.e == null
                //2.e.key != key


            //getEntryAfterMiss 方法 会继续向当前桶位后面继续搜索 e.key == key 的entry.

            //为什么这样做呢？？
            //因为 存储时  发生hash冲突后，并没有在entry层面形成 链表.. 存储时的处理 就是线性的向后找到一个可以使用的slot，并且存放进去。
                return getEntryAfterMiss(key, i, e);
        }
```

## getEntryAfterMiss源码解析

```java
     *
         * @param  key the thread local object           threadLocal对象 表示key
         * @param  i the table index for key's hash code  key计算出来的index
         * @param  e the entry at table[i]                table[index] 中的 entry
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            //获取当前threadLocalMap中的散列表 table
            Entry[] tab = table;
            //获取table长度
            int len = tab.length;

            //条件：e != null 说明 向后查找的范围是有限的，碰到 slot == null 的情况，搜索结束。
            //e:循环处理的当前元素
            while (e != null) {
                //获取当前slot 中entry对象的key
                ThreadLocal<?> k = e.get();
                //条件成立：说明向后查询过程中找到合适的entry了，返回entry就ok了。
                if (k == key)
                    //找到的情况下，就从这里返回了。
                    return e;
                //条件成立：说明当前slot中的entry#key 关联的 ThreadLocal对象已经被GC回收了.. 因为key 是弱引用， key = e.get() == null.
                if (k == null)
                    //做一次 探测式过期数据回收。
                    expungeStaleEntry(i);
                else
                    //更新index，继续向后搜索。
                    i = nextIndex(i, len);
                //获取下一个slot中的entry。
                e = tab[i];
            }

            //执行到这里，说明关联区段内都没找到相应数据。
            return null;
        }
```

## set方法源码解析

```java
        /**
         * Set the value associated with key.
         *
         * ThreadLocal 使用set方法 给当前线程添加 threadLocal-value   键值对。
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {
            //获取散列表
            Entry[] tab = table;
            //获取散列表数组长度
            int len = tab.length;
            //计算当前key 在 散列表中的对应的位置
            int i = key.threadLocalHashCode & (len-1);


            //以当前key对应的slot位置 向后查询，找到可以使用的slot。
            //什么slot可以使用呢？？
            //1.k == key 说明是替换
            //2.碰到一个过期的 slot ，这个时候 咱们可以强行占用呗。
            //3.查找过程中 碰到 slot == null 了。
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {

                //获取当前元素key
                ThreadLocal<?> k = e.get();

                //条件成立：说明当前set操作是一个替换操作。
                if (k == key) {
                    //做替换逻辑。
                    e.value = value;
                    return;
                }

                //条件成立：说明 向下寻找过程中 碰到entry#key == null 的情况了，说明当前entry 是过期数据。
                if (k == null) {
                    //碰到一个过期的 slot ，这个时候 咱们可以强行占用呗。
                    //替换过期数据的逻辑。
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }



            //执行到这里，说明for循环碰到了 slot == null 的情况。
            //在合适的slot中 创建一个新的entry对象。
            tab[i] = new Entry(key, value);
            //因为是新添加 所以++size.
            int sz = ++size;

            //做一次启发式清理
            //条件一：!cleanSomeSlots(i, sz) 成立，说明启发式清理工作 未清理到任何数据..
            //条件二：sz >= threshold 成立，说明当前table内的entry已经达到扩容阈值了..会触发rehash操作。
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

## replaceStaleEntry源码解析

![replaceStaleEntry工作原理](D:\学习笔记\replaceStaleEntry工作原理.png)

```java
         * @param  key the key
         * @param  value the value to be associated with key
         * @param  staleSlot index of the first stale entry encountered while
         *         searching for key.
         * key: 键 threadLocal对象
         * value: val
         * staleSlot: 上层方法 set方法，迭代查找时 发现的当前这个slot是一个过期的 entry。
         */
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            //获取散列表
            Entry[] tab = table;
            //获取散列表数组长度
            int len = tab.length;
            //临时变量
            Entry e;

            //表示 开始探测式清理过期数据的 开始下标。默认从当前 staleSlot开始。
            int slotToExpunge = staleSlot;


            //以当前staleSlot开始 向前迭代查找，找有没有过期的数据。for循环一直到碰到null结束。
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len)){
                //条件成立：说明向前找到了过期数据，更新 探测清理过期数据的开始下标为 i
                if (e.get() == null){
                    slotToExpunge = i;
                }
            }

            //以当前staleSlot向后去查找，直到碰到null为止。
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                //获取当前元素 key
                ThreadLocal<?> k = e.get();

                //条件成立：说明咱们是一个 替换逻辑。
                if (k == key) {
                    //替换新数据。
                    e.value = value;

                    //交换位置的逻辑..
                    //将table[staleSlot]这个过期数据 放到 当前循环到的 table[i] 这个位置。
                    tab[i] = tab[staleSlot];
                    //将tab[staleSlot] 中保存为 当前entry。 这样的话，咱们这个数据位置就被优化了..
                    tab[staleSlot] = e;

                    //条件成立：
                    // 1.说明replaceStaleEntry 一开始时 的向前查找过期数据 并未找到过期的entry.
                    // 2.向后检查过程中也未发现过期数据..
                    if (slotToExpunge == staleSlot)
                        //开始探测式清理过期数据的下标 修改为 当前循环的index。
                        slotToExpunge = i;


                    //cleanSomeSlots ：启发式清理
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                //条件1：k == null 成立，说明当前遍历的entry是一个过期数据..
                //条件2：slotToExpunge == staleSlot 成立，一开始时 的向前查找过期数据 并未找到过期的entry.
                if (k == null && slotToExpunge == staleSlot)
                    //因为向后查询过程中查找到一个过期数据了，更新slotToExpunge 为 当前位置。
                    //前提条件是 前驱扫描时 未发现 过期数据..
                    slotToExpunge = i;
            }

            //什么时候执行到这里呢？
            //向后查找过程中 并未发现 k == key 的entry，说明当前set操作 是一个添加逻辑..

            //直接将新数据添加到 table[staleSlot] 对应的slot中。
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);


            //条件成立：除了当前staleSlot 以外 ，还发现其它的过期slot了.. 所以要开启 清理数据的逻辑..
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

```

## expungeStaleEntry源码解析

![expungeStaleEntry 探测式清理过期数据](D:\学习笔记\expungeStaleEntry 探测式清理过期数据.png)

```java
        * 参数 staleSlot   table[staleSlot] 就是一个过期数据，以这个位置开始 继续向后查找过期数据，直到碰到 slot == null 的情况结束。
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         */
        private int expungeStaleEntry(int staleSlot) {
            //获取散列表
            Entry[] tab = table;
            //获取散列表当前长度
            int len = tab.length;

            // expunge entry at staleSlot
            //help gc
            tab[staleSlot].value = null;
            //因为staleSlot位置的entry 是过期的 这里直接置为Null
            tab[staleSlot] = null;
            //因为上面干掉一个元素，所以 -1.
            size--;

            // Rehash until we encounter null
            //e：表示当前遍历节点
            Entry e;
            //i：表示当前遍历的index
            int i;

            //for循环从 staleSlot + 1的位置开始搜索过期数据，直到碰到 slot == null 结束。
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                //进入到for循环里面 当前entry一定不为null


                //获取当前遍历节点 entry 的key.
                ThreadLocal<?> k = e.get();

                //条件成立：说明k表示的threadLocal对象 已经被GC回收了... 当前entry属于脏数据了...
                if (k == null) {
                    //help gc
                    e.value = null;
                    //脏数据对应的slot置为null
                    tab[i] = null;
                    //因为上面干掉一个元素，所以 -1.
                    size--;
                } else {
                    //执行到这里，说明当前遍历的slot中对应的entry 是非过期数据
                    //因为前面有可能清理掉了几个过期数据。
                    //且当前entry 存储时有可能碰到hash冲突了，往后偏移存储了，这个时候 应该去优化位置，让这个位置更靠近 正确位置。
                    //这样的话，查询的时候 效率才会更高！

                    //重新计算当前entry对应的 index
                    int h = k.threadLocalHashCode & (len - 1);
                    //条件成立：说明当前entry存储时 就是发生过hash冲突，然后向后偏移过了...
                    if (h != i) {
                        //将entry当前位置 设置为null
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.


                        //h 是正确位置。

                        //以正确位置h 开始，向后查找第一个 可以存放entry的位置。
                        while (tab[h] != null)
                            h = nextIndex(h, len);

                        //将当前元素放入到 距离正确位置 更近的位置（有可能就是正确位置）。
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

## cleanSomeSlots源码解析

```java
      * 参数 i 启发式清理工作开始位置
         * 参数 n 一般传递的是 table.length ，这里n 也表示结束条件。
         * @return true if any stale entries have been removed.
         */
        private boolean cleanSomeSlots(int i, int n) {
            //表示启发式清理工作 是否清楚过过期数据
            boolean removed = false;
            //获取当前map的散列表引用
            Entry[] tab = table;
            //获取当前散列表数组长度
            int len = tab.length;

            do {
                //这里为什么不是从i就检查呢？
                //因为cleanSomeSlots(i = expungeStaleEntry(???), n)  expungeStaleEntry(???) 返回值一定是null。

                //获取当前i的下一个 下标
                i = nextIndex(i, len);
                //获取table中当前下标为i的元素
                Entry e = tab[i];
                //条件一：e != null 成立
                //条件二：e.get() == null 成立，说明当前slot中保存的entry 是一个过期的数据..
                if (e != null && e.get() == null) {
                    //重新更新n为 table数组长度
                    n = len;
                    //表示清理过数据.
                    removed = true;
                    //以当前过期的slot为开始节点 做一次 探测式清理工作
                    i = expungeStaleEntry(i);
                }


                // 假设table长度为16
                // 16 >>> 1 ==> 8
                // 8 >>> 1 ==> 4
                // 4 >>> 1 ==> 2
                // 2 >>> 1 ==> 1
                // 1 >>> 1 ==> 0
            } while ( (n >>>= 1) != 0);

            return removed;
        }
```

![cleanSomeSlots 启发式清理工作](D:\学习笔记\cleanSomeSlots 启发式清理工作.png)

## rehash源码解析

```java
        private void rehash() {
            //这个方法执行完后，当前散列表内的所有过期的数据，都会被干掉。
            expungeStaleEntries();


            // Use lower threshold for doubling to avoid hysteresis
            //条件成立：说明清理完 过期数据后，当前散列表内的entry数量仍然达到了 threshold * 3/4，真正触发 扩容！
            if (size >= threshold - threshold / 4)
                //扩容。
                resize();
        }
```

## resize源码解析

```java
       private void resize() {
            //获取当前散列表
            Entry[] oldTab = table;
            //获取当前散列表长度
            int oldLen = oldTab.length;
            //计算出扩容后的表大小  oldLen * 2
            int newLen = oldLen * 2;
            //创建一个新的散列表
            Entry[] newTab = new Entry[newLen];
            //表示新table中的entry数量。
            int count = 0;

            //遍历老表 迁移数据到新表。
            for (int j = 0; j < oldLen; ++j) {
                //访问老表的指定位置的slot
                Entry e = oldTab[j];
                //条件成立：说明老表中的指定位置 有数据
                if (e != null) {
                    //获取entry#key
                    ThreadLocal<?> k = e.get();
                    //条件成立：说明老表中的当前位置的entry 是一个过期数据..
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        //执行到这里，说明老表的当前位置的元素是非过期数据 正常数据，需要迁移到扩容后的新表。。

                        //计算出当前entry在扩容后的新表的 存储位置。
                        int h = k.threadLocalHashCode & (newLen - 1);
                        //while循环 就是拿到一个距离h最近的一个可以使用的slot。
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);

                        //将数据存放到 新表的 合适的slot中。
                        newTab[h] = e;
                        //数量+1
                        count++;
                    }
                }
            }


            //设置下一次触发扩容的指标。
            setThreshold(newLen);
            size = count;
            //将扩容后的新表 的引用保存到 threadLocalMap 对象的 table这里。。
            table = newTab;
        }
```

# SynchronousQueue源码解析

## 内核核心抽象类Transfer

```java
   abstract static class Transferer<E> {
        /**
         * Performs a put or take.
         *
         * @param e if non-null, the item to be handed to a consumer;
         *          if null, requests that transfer return an item
         *          offered by producer.
         * @param timed if this operation should timeout
         * @param nanos the timeout, in nanoseconds
         * @return if non-null, the item provided or received; if null,
         *         the operation failed due to timeout or interrupt --
         *         the caller can distinguish which of these occurred
         *         by checking Thread.interrupted.
         *
         *
         *
         *
         * @param e 可以为null，null时表示这个请求是一个 REQUEST 类型的请求
         *          如果不是null，说明这个请求是一个 DATA 类型的请求。
         *
         * @param timed 如果为true 表示指定了超时时间 ,如果为false 表示不支持超时，表示当前请求一直等待到匹配为止，或者被中断。
         * @param nanos 超时时间限制 单位 纳秒
         *
         *
         * @return E 如果当前请求是一个 REQUEST类型的请求，返回值如果不为null 表示 匹配成功，如果返回null，表示REQUEST类型的请求超时 或 被中断。
         *           如果当前请求是一个 DATA 类型的请求，返回值如果不为null 表示 匹配成功，返回当前线程put的数据。
         *           如果返回值为null 表示，DATA类型的请求超时 或者 被中断..都会返回Null。
         *
         */
        abstract E transfer(E e, boolean timed, long nanos);
    }
```

## 成员变量解析

```java
 //为什么需要自旋这个操作？
    //因为线程 挂起 唤醒站在cpu角度去看的话，是非常耗费资源的，涉及到用户态和内核态的切换...
    //自旋的好处，自旋期间线程会一直检查自己的状态是否被匹配到，如果自旋期间被匹配到，那么直接就返回了
    //如果自旋期间未被匹配到，自旋次数达到某个指标后，还是会将当前线程挂起的...
    //NCPUS：当一个平台只有一个CPU时，你觉得还需要自旋么？
    //答：肯定不需要自旋了，因为一个cpu同一时刻只能执行一个线程，自旋没有意义了...而且你还站着cpu 其它线程没办法执行..这个
    //栈的状态更不会改变了.. 当只有一个cpu时 会直接选择 LockSupport.park() 挂起等待者线程。


    /** The number of CPUs, for spin control */
    //表示运行当前程序的平台，所拥有的CPU数量
    static final int NCPUS = Runtime.getRuntime().availableProcessors();

    /**
     * The number of times to spin before blocking in timed waits.
     * The value is empirically derived -- it works well across a
     * variety of processors and OSes. Empirically, the best value
     * seems not to vary with number of CPUs (beyond 2) so is just
     * a constant.
     */
    //表示指定超时时间的话，当前线程最大自旋次数。
    //只有一个cpu 自旋次数为0
    //当cpu大于1时，说明当前平台是多核平台，那么指定超时时间的请求的最大自旋次数是 32 次。
    //32是一个经验值。
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

    /**
     * The number of times to spin before blocking in untimed waits.
     * This is greater than timed value because untimed waits spin
     * faster since they don't need to check times on each spin.
     */
    //表示未指定超时限制的话，线程等待匹配时，自旋次数。
    //是指定超时限制的请求的自旋次数的16倍.
    static final int maxUntimedSpins = maxTimedSpins * 16;

    /**
     * The number of nanoseconds for which it is faster to spin
     * rather than to use timed park. A rough estimate suffices.
     */
    //如果请求是指定超时限制的话，如果超时nanos参数是< 1000 纳秒时，
    //禁止挂起。挂起再唤醒的成本太高了..还不如选择自旋空转呢...
    static final long spinForTimeoutThreshold = 1000L;

```

## 非公平模式TransferStack源码解析

### 成员变量解析

```java
   /* Modes for SNodes, ORed together in node fields */
          /** The head (top) of the stack */
        //表示栈顶指针
        volatile SNode head;  
        /** Node represents an unfulfilled consumer */
        /** 表示Node类型为 请求类型 */
        static final int REQUEST    = 0;
        /** Node represents an unfulfilled producer */
        /** 表示Node类型为 数据类型 */
        static final int DATA       = 1;
        /** Node is fulfilling another unfulfilled DATA or REQUEST */
        /** 表示Node类型为 匹配中类型
         * 假设栈顶元素为 REQUEST-NODE，当前请求类型为 DATA的话，入栈会修改类型为 FULFILLING 【栈顶 & 栈顶之下的一个node】。
         * 假设栈顶元素为 DATA-NODE，当前请求类型为 REQUEST的话，入栈会修改类型为 FULFILLING 【栈顶 & 栈顶之下的一个node】。
         */
        static final int FULFILLING = 2;

        /** Returns true if m has fulfilling bit set. */
        //判断当前模式是否为 匹配中状态。
        static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }
```

### 内部类SNode源码以及变量解析

```java
  /** Node class for TransferStacks. */
        static final class SNode {
            //指向下一个栈帧
            volatile SNode next;        // next node in stack
            //与当前node匹配的节点
            volatile SNode match;       // the node matched to this
            //假设当前node对应的线程 自旋期间未被匹配成功，那么node对应的线程需要挂起，挂起前 waiter 保存对应的线程引用，
            //方便 匹配成功后，被唤醒。
            volatile Thread waiter;     // to control park/unpark
            //数据域，data不为空 表示当前Node对应的请求类型为 DATA类型。 反之则表示Node为 REQUEST类型。
            Object item;                // data; or null for REQUESTs
            //表示当前Node的模式 【DATA/REQUEST/FULFILLING】
            int mode;
            // Note: item and mode fields don't need to be volatile
            // since they are always written before, and read after,
            // other volatile/atomic operations.

            SNode(Object item) {
                this.item = item;
            }



            //CAS方式设置Node对象的next字段。
            boolean casNext(SNode cmp, SNode val) {
                //优化：cmp == next  为什么要判断？
                //因为cas指令 在平台执行时，同一时刻只能有一个cas指令被执行。
                //有了java层面的这一次判断，可以提升一部分性能。 cmp == next 不相等，就没必要走 cas指令。
                return cmp == next &&
                    UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
            }


            /**
             * Tries to match node s to this node, if so, waking up thread.
             * Fulfillers call tryMatch to identify their waiters.
             * Waiters block until they have been matched.
             *
             * @param s the node to match
             * @return true if successfully matched to s
             *
             *
             * 尝试匹配
             * 调用tryMatch的对象是 栈顶节点的下一个节点，与栈顶匹配的节点。
             *
             * @return ture 匹配成功。 否则匹配失败..
             */
            boolean tryMatch(SNode s) {
                //条件一：match == null 成立，说明当前Node尚未与任何节点发生过匹配...
                //条件二 成立：使用CAS方式 设置match字段，表示当前Node已经被匹配了
                if (match == null &&
                    UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                    //当前Node如果自旋结束，那么会使用LockSupport.park 方法挂起，挂起之前会将Node对应的Thread 保留到 waiter字段。
                    Thread w = waiter;
                    //条件成立：说明Node对应的Thread已经挂起了...
                    if (w != null) {    // waiters need at most one unpark
                        waiter = null;
                        //使用unpark方式唤醒。
                        LockSupport.unpark(w);
                    }
                    return true;
                }
                return match == s;
            }

            /**
             * Tries to cancel a wait by matching node to itself.
             * 尝试取消..
             */
            void tryCancel() {
                //match字段 保留当前Node对象本身，表示这个Node是取消状态，取消状态的Node，最终会被 强制 移除出栈。
                UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
            }

            //如果match保留的是当前Node本身，那表示当前Node是取消状态，反之 则 非取消状态。
            boolean isCancelled() {
                return match == this;
            }

            // Unsafe mechanics
            private static final sun.misc.Unsafe UNSAFE;
            private static final long matchOffset;
            private static final long nextOffset;

            static {
                try {
                    UNSAFE = sun.misc.Unsafe.getUnsafe();
                    Class<?> k = SNode.class;
                    matchOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("match"));
                    nextOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("next"));
                } catch (Exception e) {
                    throw new Error(e);
                }
            }
        }
```

### transfer方法源码解析

```java
  @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
            /*
             * Basic algorithm is to loop trying one of three actions:
             *
             * 1. If apparently empty or already containing nodes of same
             *    mode, try to push node on stack and wait for a match,
             *    returning it, or null if cancelled.
             *
             * 2. If apparently containing node of complementary mode,
             *    try to push a fulfilling node on to stack, match
             *    with corresponding waiting node, pop both from
             *    stack, and return matched item. The matching or
             *    unlinking might not actually be necessary because of
             *    other threads performing action 3:
             *
             * 3. If top of stack already holds another fulfilling node,
             *    help it out by doing its match and/or pop
             *    operations, and then continue. The code for helping
             *    is essentially the same as for fulfilling, except
             *    that it doesn't return the item.
             */

            //包装当前线程的Node
            SNode s = null; // constructed/reused as needed
            //e == null 条件成立：当前线程是一个REQUEST线程。
            //否则 e!=null 说明 当前线程是一个DATA线程，提交数据的线程。
            int mode = (e == null) ? REQUEST : DATA;

            //自旋
            for (;;) {
                //h 表示栈顶指针
                SNode h = head;

                //CASE1：当前栈内为空 或者 栈顶Node模式与当前请求模式一致，都是需要做入栈操作。
                if (h == null || h.mode == mode) {  // empty or same-mode

                    //条件一：成立，说明当前请求是指定了 超时限制的
                    //条件二：nanos <= 0 , nanos == 0. 表示这个请求 不支持 “阻塞等待”。 queue.offer();
                    if (timed && nanos <= 0) {      // can't wait
                        //条件成立：说明栈顶已经取消状态了，协助栈顶出栈。
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     // pop cancelled node
                        else
                            //大部分情况从这里返回。
                            return null;

                    }

                    //什么时候执行else if 呢？
                    //当前栈顶为空 或者 模式与当前请求一致，且当前请求允许 阻塞等待。
                    //casHead(h, s = snode(s, e, h, mode))  入栈操作。
                    else if (casHead(h, s = snode(s, e, h, mode))) {
                        //执行到这里，说明 当前请求入栈成功。
                        //入栈成功之后要做什么呢？
                        //在栈内等待一个好消息，等待被匹配！

                        //awaitFulfill 等待被匹配的逻辑...
                        //1.正常情况：返回匹配的节点
                        //2.取消情况：返回当前节点  s节点进去，返回s节点...
                        SNode m = awaitFulfill(s, timed, nanos);


                        //条件成立：说明当前Node状态是 取消状态...
                        if (m == s) {               // wait was cancelled
                            //将取消状态的节点 出栈...
                            clean(s);
                            //取消状态 最终返回null
                            return null;
                        }
                        //执行到这里 说明当前Node已经被匹配了...

                        //条件一：成立，说明栈顶是有Node
                        //条件二：成立，说明 Fulfill 和 当前Node 还未出栈，需要协助出栈。
                        if ((h = head) != null && h.next == s)
                            //将fulfill 和 当前Node 结对 出栈
                            casHead(h, s.next);     // help s's fulfiller

                        //当前NODE模式为REQUEST类型：返回匹配节点的m.item 数据域
                        //当前NODE模式为DATA类型：返回Node.item 数据域，当前请求提交的 数据e
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    }
                }
                //什么时候来到这？？
                //栈顶Node的模式与当前请求的模式不一致，会执行else if 的条件。
                //栈顶是 (DATA  Reqeust)    (Request   DATA)   (FULFILLING  REQUEST/DATA)
                //CASE2：当前栈顶模式与请求模式不一致，且栈顶不是FULFILLING
                else if (!isFulfilling(h.mode)) { // try to fulfill
                    //条件成立：说明当前栈顶状态为 取消状态，当前线程协助它出栈。
                    if (h.isCancelled())            // already cancelled
                        //协助 取消状态节点 出栈。
                        casHead(h, h.next);         // pop and retry



                    //条件成立：说明压栈节点成功，入栈一个 FULFILLING | mode  NODE
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        //当前请求入栈成功

                        //自旋，fulfill 节点 和 fulfill.next 节点进行匹配工作...
                        for (;;) { // loop until matched or waiters disappear
                            //m 与当前s 匹配节点。
                            SNode m = s.next;       // m is s's match

                            //m == null 什么时候可能成立呢？
                            //当s.next节点 超时或者被外部线程中断唤醒后，会执行 clean 操作 将 自己清理出栈，此时
                            //站在匹配者线程 来看，真有可能拿到一个null。
                            if (m == null) {        // all waiters are gone
                                //将整个栈清空。
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                //回到外层大的 自旋中，再重新选择路径执行，此时有可能 插入一个节点。
                                break;              // restart main loop
                            }

                            //什么时候会执行到这里呢？
                            //fulfilling 匹配节点不为null，进行真正的匹配工作。

                            //获取 匹配节点的 下一个节点。
                            SNode mn = m.next;
                            //尝试匹配，匹配成功，则将fulfilling 和 m 一起出栈。
                            if (m.tryMatch(s)) {
                                //结对出栈
                                casHead(s, mn);     // pop both s and m

                                //当前NODE模式为REQUEST类型：返回匹配节点的m.item 数据域
                                //当前NODE模式为DATA类型：返回Node.item 数据域，当前请求提交的 数据e
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match
                                //强制出栈
                                s.casNext(m, mn);   // help unlink
                        }

                    }
                }

                //CASE3：什么时候会执行？
                //栈顶模式为 FULFILLING模式，表示栈顶和栈顶下面的栈帧正在发生匹配...
                //当前请求需要做 协助 工作。
                else {                            // help a fulfiller
                    //h 表示的是 fulfilling节点,m fulfilling匹配的节点。
                    SNode m = h.next;               // m is h's match
                    //m == null 什么时候可能成立呢？
                    //当s.next节点 超时或者被外部线程中断唤醒后，会执行 clean 操作 将 自己清理出栈，此时
                    //站在匹配者线程 来看，真有可能拿到一个null。
                    if (m == null)                  // waiter is gone
                        //清空栈
                        casHead(h, null);           // pop fulfilling node

                    //大部分情况：走else分支。
                    else {
                        //获取栈顶匹配节点的 下一个节点
                        SNode mn = m.next;
                        //条件成立：说明 m 和 栈顶 匹配成功
                        if (m.tryMatch(h))          // help match
                            //双双出栈，让栈顶指针指向 匹配节点的下一个节点。
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            //强制出栈
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }
```

### awaitFulfill源码解析

```java
     * @param s 当前请求Node
         * @param timed 当前请求是否支持 超时限制
         * @param nanos 如果请求支持超时限制，nanos 表示超时等待时长。
         */
        SNode awaitFulfill(SNode s, boolean timed, long nanos) {
            /*
             * When a node/thread is about to block, it sets its waiter
             * field and then rechecks state at least one more time
             * before actually parking, thus covering race vs
             * fulfiller noticing that waiter is non-null so should be
             * woken.
             *
             * When invoked by nodes that appear at the point of call
             * to be at the head of the stack, calls to park are
             * preceded by spins to avoid blocking when producers and
             * consumers are arriving very close in time.  This can
             * happen enough to bother only on multiprocessors.
             *
             * The order of checks for returning out of main loop
             * reflects fact that interrupts have precedence over
             * normal returns, which have precedence over
             * timeouts. (So, on timeout, one last check for match is
             * done before giving up.) Except that calls from untimed
             * SynchronousQueue.{poll/offer} don't check interrupts
             * and don't wait at all, so are trapped in transfer
             * method rather than calling awaitFulfill.
             */


            //等待的截止时间。  timed == true  =>  System.nanoTime() + nanos
            final long deadline = timed ? System.nanoTime() + nanos : 0L;

            //获取当前请求线程..
            Thread w = Thread.currentThread();

            //spins 表示当前请求线程 在 下面的 for(;;) 自旋检查中，自旋次数。 如果达到spins自旋次数时，当前线程对应的Node 仍然未被匹配成功，
            //那么再选择 挂起 当前请求线程。
            int spins = (shouldSpin(s) ?
                          //timed == true 指定了超时限制的，这个时候采用 maxTimedSpins == 32 ,否则采用 32 * 16
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);

            //自旋检查逻辑：1.是否匹配  2.是否超时  3.是否被中断..
            for (;;) {


                //条件成立：说明当前线程收到中断信号，需要设置Node状态为 取消状态。
                if (w.isInterrupted())
                    //Node对象的 match 指向 当前Node 说明该Node状态就是 取消状态。
                    s.tryCancel();


                //m 表示与当前Node匹配的节点。
                //1.正常情况：有一个请求 与 当前Node 匹配成功，这个时候 s.match 指向 匹配节点。
                //2.取消情况：当前match 指向 当前Node...
                SNode m = s.match;

                if (m != null)
                    //可能正常 也可能是 取消...
                    return m;



                //条件成立：说明指定了超时限制..
                if (timed) {
                    //nanos 表示距离超时 还有多少纳秒..
                    nanos = deadline - System.nanoTime();
                    //条件成立：说明已经超时了...
                    if (nanos <= 0L) {
                        //设置当前Node状态为 取消状态.. match-->当前Node
                        s.tryCancel();
                        continue;
                    }
                }





                //条件成立：说明当前线程还可以进行自旋检查...
                if (spins > 0)
                    //自旋次数 累积 递减。。。
                    spins = shouldSpin(s) ? (spins-1) : 0;


                //spins == 0 ，已经不允许再进行自旋检查了
                else if (s.waiter == null)
                    //把当前Node对应的Thread 保存到 Node.waiter字段中..
                    s.waiter = w; // establish waiter so can park next iter

                //条件成立：说明当前Node对应的请求  未指定超时限制。
                else if (!timed)
                    //使用不指定超时限制的park方法 挂起当前线程，直到 当前线程被外部线程 使用unpark唤醒。
                    LockSupport.park(this);

                //什么时候执行到这里？ timed == true 设置了 超时限制..
                //条件成立：nanos > 1000 纳秒的值，只有这种情况下，才允许挂起当前线程..否则 说明 超时给的太少了...挂起和唤醒的成本 远大于 空转自旋...
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }
```

### shouldSpin方法源码解析

```java
   boolean shouldSpin(SNode s) {
            //获取栈顶
            SNode h = head;
            //条件一 h == s ：条件成立 说明当前s 就是栈顶，允许自旋检查...
            //条件二 h == null : 什么时候成立？ 当前s节点 自旋检查期间，又来了一个 与当前s 节点匹配的请求，双双出栈了...条件会成立。
            //条件三 isFulfilling(h.mode) ： 前提 当前 s 不是 栈顶元素。并且当前栈顶正在匹配中，这种状态 栈顶下面的元素，都允许自旋检查。
            return (h == s || h == null || isFulfilling(h.mode));
        }
```

### clean方法源码解析

```java
     void clean(SNode s) {
            //清空数据域
            s.item = null;   // forget item
            //释放线程引用..
            s.waiter = null; // forget thread
            /*
             * At worst we may need to traverse entire stack to unlink
             * s. If there are multiple concurrent calls to clean, we
             * might not see s if another thread has already removed
             * it. But we can stop when we see any node known to
             * follow s. We use s.next unless it too is cancelled, in
             * which case we try the node one past. We don't check any
             * further because we don't want to doubly traverse just to
             * find sentinel.
             */
            //检查取消节点的截止位置
            SNode past = s.next;

            if (past != null && past.isCancelled())
                past = past.next;

            // Absorb cancelled nodes at head
            //当前循环检查节点
            SNode p;
            //从栈顶开始向下检查，将栈顶开始向下连续的 取消状态的节点 全部清理出去，直到碰到past为止。
            while ((p = head) != null && p != past && p.isCancelled())
                casHead(p, p.next);

            // Unsplice embedded nodes
            while (p != null && p != past) {
                //获取p.next
                SNode n = p.next;
                if (n != null && n.isCancelled())
                    p.casNext(n, n.next);
                else
                    p = n;
            }
        }
```

## 公平模式TransferQueue源码解析

### QNode内部类源码解析

```java
   /** Node class for TransferQueue. */
        static final class QNode {
            //指向当前节点的下一个节点，组装链表使用的。
            volatile QNode next;          // next node in queue
            //数据域  Node代表的是DATA类型，item表示数据   否则 Node代表的REQUEST类型，item == null
            volatile Object item;         // CAS'ed to or from null
            //当Node对应的线程 未匹配到节点时，对应的线程 最终会挂起，挂起之前会保留 线程引用到waiter ，
            //方法 其它Node匹配当前节点时 唤醒 当前线程..
            volatile Thread waiter;       // to control park/unpark
            //true 当前Node是一个DATA类型   false表示当前Node是一个REQUEST类型。
            final boolean isData;


            QNode(Object item, boolean isData) {
                this.item = item;
                this.isData = isData;
            }

            //修改当前节点next引用
            boolean casNext(QNode cmp, QNode val) {
                return next == cmp &&
                    UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
            }

            //修改当前节点数据域item
            boolean casItem(Object cmp, Object val) {
                return item == cmp &&
                    UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
            }

            /**
             * Tries to cancel by CAS'ing ref to this as item.
             * 尝试取消当前node
             * 取消状态的Node，它的item域，指向自己Node。
             */
            void tryCancel(Object cmp) {
                UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
            }

            //判断当前Node是否为取消状态
            boolean isCancelled() {
                return item == this;
            }

            /**
             * Returns true if this node is known to be off the queue
             * because its next pointer has been forgotten due to
             * an advanceHead operation.
             *
             * 判断当前节点是否 “不在” 队列内，当next指向自己时，说明节点已经出队。
             */
            boolean isOffList() {
                return next == this;
            }

            // Unsafe mechanics
            private static final sun.misc.Unsafe UNSAFE;
            private static final long itemOffset;
            private static final long nextOffset;

            static {
                try {
                    UNSAFE = sun.misc.Unsafe.getUnsafe();
                    Class<?> k = QNode.class;
                    itemOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("item"));
                    nextOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("next"));
                } catch (Exception e) {
                    throw new Error(e);
                }
            }
        }
```

### 成员变量源码解析

```java
  /** Head of queue */
        //指向队列的dummy节点
        transient volatile QNode head;
        /** Tail of queue */
        //指向队列的尾节点。
        transient volatile QNode tail;
        /**
         * Reference to a cancelled node that might not yet have been
         * unlinked from queue because it was the last inserted node
         * when it was cancelled.
         * 表示被清理节点的前驱节点。因为入队操作是 两步完成的，
         * 第一步：t.next = newNode
         * 第二步：tail = newNode
         * 所以，队尾节点出队，是一种非常特殊的情况，需要特殊处理，回头讲！
         */
        transient volatile QNode cleanMe;
```



### advanceHead方法源码解析

```java
     /**
         * Tries to cas nh as new head; if successful, unlink
         * old head's next node to avoid garbage retention.
         * 设置头指针指向新的节点，蕴含操作：老的头节点出队。
         */
        void advanceHead(QNode h, QNode nh) {
            if (h == head &&
                UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
                h.next = h; // forget old next
        }

```

### advanceTail方法源码解析

```java
     /**
         * Tries to cas nt as new tail.
         * 更新队尾节点 为新的队尾。
         * @param t 老的队尾
         * @param nt 新的队尾
         */
        void advanceTail(QNode t, QNode nt) {
            if (tail == t)
                UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
        }
```

### transfer方法源码解析

```java
   @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
            /* Basic algorithm is to loop trying to take either of
             * two actions:
             *
             * 1. If queue apparently empty or holding same-mode nodes,
             *    try to add node to queue of waiters, wait to be
             *    fulfilled (or cancelled) and return matching item.
             *
             * 2. If queue apparently contains waiting items, and this
             *    call is of complementary mode, try to fulfill by CAS'ing
             *    item field of waiting node and dequeuing it, and then
             *    returning matching item.
             *
             * In each case, along the way, check for and try to help
             * advance head and tail on behalf of other stalled/slow
             * threads.
             *
             * The loop starts off with a null check guarding against
             * seeing uninitialized head or tail values. This never
             * happens in current SynchronousQueue, but could if
             * callers held non-volatile/final ref to the
             * transferer. The check is here anyway because it places
             * null checks at top of loop, which is usually faster
             * than having them implicitly interspersed.
             */
            //s 指向当前请求 对应Node
            QNode s = null; // constructed/reused as needed
            //isData == true 表示 当前请求是一个写数据操作（DATA）   否则isData == false 表示当前请求是一个 REQUEST操作。
            boolean isData = (e != null);


            //自旋..
            for (;;) {
                QNode t = tail;
                QNode h = head;
                if (t == null || h == null)         // saw uninitialized value
                    continue;                       // spin


                //CASE1：入队
                //条件一：成立，说明head和tail同时指向dummy节点，当前队列实际情况 就是 空队列。此时当前请求需要做入队操作，因为没有任何节点 可以去匹配。
                //条件二：队列不是空，队尾节点与当前请求类型是一致的情况。说明也是无法完成匹配操作的情况，此时当前节点只能入队...
                if (h == t || t.isData == isData) { // empty or same-mode
                    //获取当前队尾t 的next节点 tn - t.next
                    QNode tn = t.next;
                    //因为多线程环境，当前线程在入队之前，其它线程有可能已经入队过了..改变了 tail 引用。
                    if (t != tail)                    // inconsistent read
                        //线程回到自旋...再选择路径执行。
                        continue;

                    //条件成立：说明已经有线程 入队了，且只完成了 入队的 第一步：设置t.next = newNode， 第二步可能尚未完成..
                    if (tn != null) {               // lagging tail
                        //协助更新tail 指向新的　尾结点。
                        advanceTail(t, tn);
                        //线程回到自旋...再选择路径执行。
                        continue;
                    }

                    //条件成立：说明当前调用transfer方法的 上层方法 可能是 offer() 无参的这种方法进来的，这种方法不支持 阻塞等待...
                    if (timed && nanos <= 0)        // can't wait
                        //检查未匹配到,直接返回null。
                        return null;



                    //条件成立：说明当前请求尚未 创建对应的node
                    if (s == null)
                        //创建node过程...
                        s = new QNode(e, isData);



                    //条件 不成立：!t.casNext(null, s)  说明当前t仍然是tail，当前线程对应的Node入队的第一步 完成！
                    if (!t.casNext(null, s))        // failed to link in
                        continue;


                    //更新队尾 为咱们请求节点。
                    advanceTail(t, s);              // swing tail and wait

                    //当前节点 等待匹配....
                    //当前请求为DATA模式时：e 请求带来的数据
                    //x == this 当前SNode对应的线程 取消状态
                    //x == null 表示已经有匹配节点了，并且匹配节点拿走了item数据。

                    //当前请求为REQUEST模式时：e == null
                    //x == this 当前SNode对应的线程 取消状态
                    //x != null 且 item != this  表示当前REQUEST类型的Node已经匹配到一个DATA类型的Node了。
                    Object x = awaitFulfill(s, e, timed, nanos);


                    //说明当前Node状态为 取消状态，需要做 出队逻辑。
                    if (x == s) {                   // wait was cancelled
                        //清理出队逻辑，最后讲。
                        clean(t, s);
                        return null;
                    }

                    //执行到这里说明 当前Node 匹配成功了...
                    //1.当前线程在awaitFulfill方法内，已经挂起了...此时运行到这里时是被 匹配节点的线程使用LockSupport.unpark() 唤醒的..
                    //被唤醒：当前请求对应的节点，肯定已经出队了，因为匹配者线程 是先让当前Node出队的，再唤醒当前Node对应线程的。

                    //2.当前线程在awaitFulfill方法内，处于自旋状态...此时匹配节点 匹配后，它检查发现了，然后返回到上层transfer方法的。
                    //自旋状态返回时：当前请求对应的节点，不一定就出队了...


                    //被唤醒时：s.isOffList() 条件会成立。  !s.isOffList() 不会成立。
                    //条件成立：说明当前Node仍然在队列内，需要做 匹配成功后 出队逻辑。
                    if (!s.isOffList()) {           // not already unlinked
                        //其实这里面做的事情，就是防止当前Node是自旋检查状态时发现 被匹配了，然后当前线程 需要将
                        //当前线程对应的Node做出队逻辑.

                        //t 当前s节点的前驱节点，更新dummy节点为 s节点。表示head.next节点已经出队了...
                        advanceHead(t, s);          // unlink if head

                        //x != null 且 item != this  表示当前REQUEST类型的Node已经匹配到一个DATA类型的Node了。
                        //因为s节点已经出队了，所以需要把它的item域 给设置为它自己，表示它是个取消出队状态。
                        if (x != null)              // and forget fields
                            s.item = s;
                        //因为s已经出队，所以waiter一定要保证是null。
                        s.waiter = null;
                    }

                    //x != null 成立，说明当前请求是REQUEST类型，返回匹配到的数据x
                    //x != null 不成立，说明当前请求是DATA类型，返回DATA请求时的e。
                    return (x != null) ? (E)x : e;
                }

                //CASE2：队尾节点 与 当前请求节点 互补 （队尾->DATA，请求类型->REQUEST）  (队尾->REQUEST, 请求类型->DATA)
                else {                            // complementary-mode
                    //h.next节点 其实是真正的队头，请求节点 与队尾模式不同，需要与队头 发生匹配。因为TransferQueue是一个 公平模式
                    QNode m = h.next;               // node to fulfill

                    //条件一：t != tail 什么时候成立呢？ 肯定是并发导致的，其它线程已经修改过tail了，有其它线程入队过了..当前线程看到的是过期数据，需要重新循环
                    //条件二：m == null 什么时候成立呢？ 肯定是其它请求先当前请求一步，匹配走了head.next节点。
                    //条件三：条件成立，说明已经有其它请求匹配走head.next了。。。当前线程看到的是过期数据。。。重新循环...
                    if (t != tail || m == null || h != head)
                        continue;                   // inconsistent read


                    //执行到这里，说明t m h 不是过期数据,是准确数据。目前来看是准确的！
                    //获取匹配节点的数据域 保存到x
                    Object x = m.item;


                    //条件一：isData == (x != null)
                    //isData 表示当前请求是什么类型  isData == true：当前请求是DATA类型  isData == false：当前请求是REQUEST类型。
                    //1.假设isData == true   DATA类型
                    //m其实表示的是 REQUEST 类型的NODE，它的数据域是 null  => x==null
                    //true == (null != null)  => true == false => false

                    //2.假设isData == false REQUEST类型
                    //m其实表示的是 DATA 类型的NODE，它的数据域是 提交是的e ，并且e != null。
                    //false == (obj != null) => false == true => false

                    //总结：正常情况下，条件一不会成立。

                    //条件二：条件成立，说明m节点已经是 取消状态了...不能完成匹配，当前请求需要continue，再重新选择路径执行了..

                    //条件三：!m.casItem(x, e)，前提条件 m 非取消状态。
                    //1.假设当前请求为REQUEST类型   e == null
                    //m 是 DATA类型了...
                    //相当于将匹配的DATA Node的数据域清空了，相当于REQUEST 拿走了 它的数据。

                    //2.假设当前请求为DATA类型    e != null
                    //m 是 REQUEST类型了...
                    //相当于将匹配的REQUEST Node的数据域 填充了，填充了 当前DATA 的 数据。相当于传递给REQUEST请求数据了...
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }


                    //执行到这里，说明匹配已经完成了，匹配完成后，需要做什么？
                    //1.将真正的头节点 出队。让这个真正的头结点成为dummy节点
                    advanceHead(h, m);              // successfully fulfilled
                    //2.唤醒匹配节点的线程..
                    LockSupport.unpark(m.waiter);

                    //x != null 成立，说明当前请求是REQUEST类型，返回匹配到的数据x
                    //x != null 不成立，说明当前请求是DATA类型，返回DATA请求时的e。
                    return (x != null) ? (E)x : e;
                }
            }
        }
```

### awaitFulfill方法源码解析

```java
   @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
            /* Basic algorithm is to loop trying to take either of
             * two actions:
             *
             * 1. If queue apparently empty or holding same-mode nodes,
             *    try to add node to queue of waiters, wait to be
             *    fulfilled (or cancelled) and return matching item.
             *
             * 2. If queue apparently contains waiting items, and this
             *    call is of complementary mode, try to fulfill by CAS'ing
             *    item field of waiting node and dequeuing it, and then
             *    returning matching item.
             *
             * In each case, along the way, check for and try to help
             * advance head and tail on behalf of other stalled/slow
             * threads.
             *
             * The loop starts off with a null check guarding against
             * seeing uninitialized head or tail values. This never
             * happens in current SynchronousQueue, but could if
             * callers held non-volatile/final ref to the
             * transferer. The check is here anyway because it places
             * null checks at top of loop, which is usually faster
             * than having them implicitly interspersed.
             */
            //s 指向当前请求 对应Node
            QNode s = null; // constructed/reused as needed
            //isData == true 表示 当前请求是一个写数据操作（DATA）   否则isData == false 表示当前请求是一个 REQUEST操作。
            boolean isData = (e != null);


            //自旋..
            for (;;) {
                QNode t = tail;
                QNode h = head;
                if (t == null || h == null)         // saw uninitialized value
                    continue;                       // spin


                //CASE1：入队
                //条件一：成立，说明head和tail同时指向dummy节点，当前队列实际情况 就是 空队列。此时当前请求需要做入队操作，因为没有任何节点 可以去匹配。
                //条件二：队列不是空，队尾节点与当前请求类型是一致的情况。说明也是无法完成匹配操作的情况，此时当前节点只能入队...
                if (h == t || t.isData == isData) { // empty or same-mode
                    //获取当前队尾t 的next节点 tn - t.next
                    QNode tn = t.next;
                    //因为多线程环境，当前线程在入队之前，其它线程有可能已经入队过了..改变了 tail 引用。
                    if (t != tail)                    // inconsistent read
                        //线程回到自旋...再选择路径执行。
                        continue;

                    //条件成立：说明已经有线程 入队了，且只完成了 入队的 第一步：设置t.next = newNode， 第二步可能尚未完成..
                    if (tn != null) {               // lagging tail
                        //协助更新tail 指向新的　尾结点。
                        advanceTail(t, tn);
                        //线程回到自旋...再选择路径执行。
                        continue;
                    }

                    //条件成立：说明当前调用transfer方法的 上层方法 可能是 offer() 无参的这种方法进来的，这种方法不支持 阻塞等待...
                    if (timed && nanos <= 0)        // can't wait
                        //检查未匹配到,直接返回null。
                        return null;



                    //条件成立：说明当前请求尚未 创建对应的node
                    if (s == null)
                        //创建node过程...
                        s = new QNode(e, isData);



                    //条件 不成立：!t.casNext(null, s)  说明当前t仍然是tail，当前线程对应的Node入队的第一步 完成！
                    if (!t.casNext(null, s))        // failed to link in
                        continue;


                    //更新队尾 为咱们请求节点。
                    advanceTail(t, s);              // swing tail and wait

                    //当前节点 等待匹配....
                    //当前请求为DATA模式时：e 请求带来的数据
                    //x == this 当前SNode对应的线程 取消状态
                    //x == null 表示已经有匹配节点了，并且匹配节点拿走了item数据。

                    //当前请求为REQUEST模式时：e == null
                    //x == this 当前SNode对应的线程 取消状态
                    //x != null 且 item != this  表示当前REQUEST类型的Node已经匹配到一个DATA类型的Node了。
                    Object x = awaitFulfill(s, e, timed, nanos);


                    //说明当前Node状态为 取消状态，需要做 出队逻辑。
                    if (x == s) {                   // wait was cancelled
                        //清理出队逻辑，最后讲。
                        clean(t, s);
                        return null;
                    }

                    //执行到这里说明 当前Node 匹配成功了...
                    //1.当前线程在awaitFulfill方法内，已经挂起了...此时运行到这里时是被 匹配节点的线程使用LockSupport.unpark() 唤醒的..
                    //被唤醒：当前请求对应的节点，肯定已经出队了，因为匹配者线程 是先让当前Node出队的，再唤醒当前Node对应线程的。

                    //2.当前线程在awaitFulfill方法内，处于自旋状态...此时匹配节点 匹配后，它检查发现了，然后返回到上层transfer方法的。
                    //自旋状态返回时：当前请求对应的节点，不一定就出队了...


                    //被唤醒时：s.isOffList() 条件会成立。  !s.isOffList() 不会成立。
                    //条件成立：说明当前Node仍然在队列内，需要做 匹配成功后 出队逻辑。
                    if (!s.isOffList()) {           // not already unlinked
                        //其实这里面做的事情，就是防止当前Node是自旋检查状态时发现 被匹配了，然后当前线程 需要将
                        //当前线程对应的Node做出队逻辑.

                        //t 当前s节点的前驱节点，更新dummy节点为 s节点。表示head.next节点已经出队了...
                        advanceHead(t, s);          // unlink if head

                        //x != null 且 item != this  表示当前REQUEST类型的Node已经匹配到一个DATA类型的Node了。
                        //因为s节点已经出队了，所以需要把它的item域 给设置为它自己，表示它是个取消出队状态。
                        if (x != null)              // and forget fields
                            s.item = s;
                        //因为s已经出队，所以waiter一定要保证是null。
                        s.waiter = null;
                    }

                    //x != null 成立，说明当前请求是REQUEST类型，返回匹配到的数据x
                    //x != null 不成立，说明当前请求是DATA类型，返回DATA请求时的e。
                    return (x != null) ? (E)x : e;
                }

                //CASE2：队尾节点 与 当前请求节点 互补 （队尾->DATA，请求类型->REQUEST）  (队尾->REQUEST, 请求类型->DATA)
                else {                            // complementary-mode
                    //h.next节点 其实是真正的队头，请求节点 与队尾模式不同，需要与队头 发生匹配。因为TransferQueue是一个 公平模式
                    QNode m = h.next;               // node to fulfill

                    //条件一：t != tail 什么时候成立呢？ 肯定是并发导致的，其它线程已经修改过tail了，有其它线程入队过了..当前线程看到的是过期数据，需要重新循环
                    //条件二：m == null 什么时候成立呢？ 肯定是其它请求先当前请求一步，匹配走了head.next节点。
                    //条件三：条件成立，说明已经有其它请求匹配走head.next了。。。当前线程看到的是过期数据。。。重新循环...
                    if (t != tail || m == null || h != head)
                        continue;                   // inconsistent read


                    //执行到这里，说明t m h 不是过期数据,是准确数据。目前来看是准确的！
                    //获取匹配节点的数据域 保存到x
                    Object x = m.item;


                    //条件一：isData == (x != null)
                    //isData 表示当前请求是什么类型  isData == true：当前请求是DATA类型  isData == false：当前请求是REQUEST类型。
                    //1.假设isData == true   DATA类型
                    //m其实表示的是 REQUEST 类型的NODE，它的数据域是 null  => x==null
                    //true == (null != null)  => true == false => false

                    //2.假设isData == false REQUEST类型
                    //m其实表示的是 DATA 类型的NODE，它的数据域是 提交是的e ，并且e != null。
                    //false == (obj != null) => false == true => false

                    //总结：正常情况下，条件一不会成立。

                    //条件二：条件成立，说明m节点已经是 取消状态了...不能完成匹配，当前请求需要continue，再重新选择路径执行了..

                    //条件三：!m.casItem(x, e)，前提条件 m 非取消状态。
                    //1.假设当前请求为REQUEST类型   e == null
                    //m 是 DATA类型了...
                    //相当于将匹配的DATA Node的数据域清空了，相当于REQUEST 拿走了 它的数据。

                    //2.假设当前请求为DATA类型    e != null
                    //m 是 REQUEST类型了...
                    //相当于将匹配的REQUEST Node的数据域 填充了，填充了 当前DATA 的 数据。相当于传递给REQUEST请求数据了...
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }


                    //执行到这里，说明匹配已经完成了，匹配完成后，需要做什么？
                    //1.将真正的头节点 出队。让这个真正的头结点成为dummy节点
                    advanceHead(h, m);              // successfully fulfilled
                    //2.唤醒匹配节点的线程..
                    LockSupport.unpark(m.waiter);

                    //x != null 成立，说明当前请求是REQUEST类型，返回匹配到的数据x
                    //x != null 不成立，说明当前请求是DATA类型，返回DATA请求时的e。
                    return (x != null) ? (E)x : e;
                }
            }
        }
```

### clean方法源码解析

```java
    */
        void clean(QNode pred, QNode s) {
            s.waiter = null; // forget thread
            /*
             * At any given time, exactly one node on list cannot be
             * deleted -- the last inserted node. To accommodate this,
             * if we cannot delete s, we save its predecessor as
             * "cleanMe", deleting the previously saved version
             * first. At least one of node s or the node previously
             * saved can always be deleted, so this always terminates.
             */
            while (pred.next == s) { // Return early if already unlinked
                QNode h = head;
                QNode hn = h.next;   // Absorb cancelled first node as head
                if (hn != null && hn.isCancelled()) {
                    advanceHead(h, hn);
                    continue;
                }
                QNode t = tail;      // Ensure consistent read for tail
                if (t == h)
                    return;
                QNode tn = t.next;
                if (t != tail)
                    continue;
                if (tn != null) {
                    advanceTail(t, tn);
                    continue;
                }
                if (s != t) {        // If not tail, try to unsplice
                    QNode sn = s.next;
                    if (sn == s || pred.casNext(s, sn))
                        return;
                }




                QNode dp = cleanMe;
                if (dp != null) {    // Try unlinking previous cancelled node
                    QNode d = dp.next;
                    QNode dn;
                    if (d == null ||               // d is gone or
                        d == dp ||                 // d is off list or
                        !d.isCancelled() ||        // d not cancelled or
                        (d != t &&                 // d not tail and
                         (dn = d.next) != null &&  //   has successor
                         dn != d &&                //   that is on list
                         dp.casNext(d, dn)))       // d unspliced
                        casCleanMe(dp, null);
                    if (dp == pred)
                        return;      // s is already saved node
                } else if (casCleanMe(null, pred))
                    return;          // Postpone cleaning s
            }
        }
```

