---
title: Java8中HashMap和ConcurrentHashMap解析
date: 2019-01-17 13:57:10
tags: 集合
categories: Java
---

HashMap是我们平时开发中用的比较多的集合容器，但是它不是线程安全的。当需要在多线程时保证安全我们会选择使用ConcurrentHashMap。下面对这两个集合介绍（基于Java8的源码，同时会与Java7的简单对比）。

<!--more-->

------------

### HashMap
Java8的HashMap底层结构是基于数组+链表/红黑树，相比Java7多了一个红黑树。红黑树主要解决了当冲突过多，链表太长，查询时间复杂度会变成O(N)的问题，Java8中当链表存储元素大于8之后将转成红黑树的存储方式，这样最坏时间复杂度由O(N)变成了O(logN)。

下图是HashMap的简单结构示意图：
![images](/images/hashmap-struct.png)
Java7 中使用 Entry 来代表每个 HashMap 中的数据节点，Java8 中使用 Node，基本没有区别，都是 key，value，hash 和 next 这四个属性，不过，Node 只能用于链表的情况，红黑树的情况需要使用 TreeNode。我们根据数组元素中，第一个节点数据类型是 Node 还是 TreeNode 来判断该位置下是链表还是红黑树的。

####  常量介绍
``` java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //默认容量16

static final int MAXIMUM_CAPACITY = 1 << 30; //最大容量

static final float DEFAULT_LOAD_FACTOR = 0.75f; //负载因子

static final int TREEIFY_THRESHOLD = 8;  //链表转换为红黑树的阈值

static final int UNTREEIFY_THRESHOLD = 6;  //红黑树转换为链表的阈值

static final int MIN_TREEIFY_CAPACITY = 64; //链表转换为红黑树需满足的最小容量
```

#### put方法分析


```  java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

// 第四个参数 onlyIfAbsent 如果是 true，那么只有在不存在该 key 时才会进行 put 操作
// 第五个参数 evict 我们这里不关心
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
         //第一次put的时候table还没初始化，在resize方法中初始化table (第一次 resize 和后续的扩容有些不一样)
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
            
         //根据hash值找到数组下标的位置，如果此位置没有值，则初始化一个node放入该位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
           //此位置已经有值
            Node<K,V> e; K k;
            //首先判断该位置的第一个数据和我们要插入的数据，key 是不是"相等"，如果是，取出这个节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
             //如果该结点是红黑树的结点，调用红黑树的方法put进去。
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
               //到这里说明该位置上的node是一个链表
                for (int binCount = 0; ; ++binCount) {
                   // 插入到链表的最后面(Java7 是插入到链表的最前面)
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //链表数量达到8个转成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果找到相同key，则直接返回该结点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //e!=null说明map已经存在key相同的，下面判断是否需要覆盖。
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 如果 HashMap 由于新插入这个值导致 size 已经超过了阈值，需要进行扩容
        //Java7中是先扩容再插入，这里是先插入再扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

#### 扩容resize方法分析

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
             // 将数组大小扩大一倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 将阈值扩大一倍
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold(设置了初始容量的时候)
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults（默认初始容量）
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        
        //创建一个新的数组，赋值给table
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;//第一次put调用resize到这就结束了，下面是数据迁移 
        
        if (oldTab != null) {
            // 开始遍历原数组，进行数据迁移。
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果原数组某个位置的node只有一个元素，则直接迁移到新的数组就行
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果是红黑树，具体我们就不展开了
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                    
                    // 这块是处理链表的情况，
                    // 需要将此链表拆成两个链表，放到新的数组中，并且保留原来的先后顺序
                    // loHead、loTail 对应一条链表，hiHead、hiTail 对应另一条链表
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
                            // 第一条链表
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 第二条链表的新的位置是 j + oldCap
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
    

#### get方法分析
HashMap的get操作相当简单，根据key的hash值去数组中找对应的位置的node，如果该位置第一个元素就是要找的就直接返回，否则判断node是红黑树还是链表，遍历红黑树或者链表找到对应的value返回。

```
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
 }
    
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
              //头结点是的话直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
              //遍历红黑树查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //遍历链表查找
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

#### HashMap线程不安全的体现
*  多个线程同时插入时，如果两个键的hash冲突了，有可能同时操作同一个结点，会造成数据覆盖的问题。
*  扩容时，会新建一个table，然后把数据迁移到新的table，如果在迁移时有线程来获取数据，就会造成获取数据为空。

#### hashCode的计算
```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
HashMap的hash值都是通过将key的hashCode()高16位与低16位进行异或运算，这样可以保留高位的特征，避免一些key的hashCode高位不同，低位相同，造成hash冲突。

#### HashMap扩容

##### 扩容过程
HashMap有一个负载因子，默认0.75，当容量达到最大容量*0.75时，发生扩容，新的容量为原来的两倍。然后遍历老的数组，当数组结点只有一个元素时，直接通过hash计算放入新的数组；当数组结点是红黑树时，进行红黑树的处理；当数组结点时一个链表时，遍历链表，根据hash&(2\*length)是0还是1判断放入oldIndex还是oldIndex+length。

##### 扩容是否需要rehash

在Java8以后，不再需要rehash了，元素存储在数组中的位置是根据hash%length来决定的，扩容之后length变成2\*length，由于length是2的N次幂，hash&(2\*length)的最后结果只有0或者1，当是0时，元素在数组中的位置还是之前的oldIndex，当是1时，元素在数组中的位置变成了oldIndex+length。（这个在上面的源码中也体现了）。

HashMap的容量必须是2的N次幂，因为：

   *  计算数组位置时，因为hash&(length-1)=hash%length，可以通过hash&(length-1)来计算，位运算的效率更高。
   *  方便扩容时不用rehash，直接根据hash&(2*length)的结果是0还是1来判断新的位置。

-------

### ConcurrentHashMap

Java8的ConcurrentHashMap底层结构和HashMap是一致的，不过由于要保证线程安全，使用了cas和synchronized锁。Java7的ConcurrentHashMap使用了分段锁，引入了Segment，一个ConcurrentHashMap由Segment数组组成，Segment继承了ReentrantLock来进行加锁，所以每次操作时只需要对每个Segment进行加锁，其他Segment不受影响，这样并发度就高了。

#### put方法分析
```java
     public V put(K key, V value) {
        return putVal(key, value, false);
     }

     final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        //计算hash值
        int hash = spread(key.hashCode());
        int binCount = 0;
        //无限循环，直到put成功
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //tab==null或者长度为0，初始化table
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
              //根据hash值找到tab结点，并得到首结点f
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //当f为null，则使用cas将该f结点设置为需要添加的结点，成功跳出循环，失败继续循环
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
            //f结点的hash值为-1，则说明正在扩容，需要去帮助扩容
                tab = helpTransfer(tab, f);
            else {
                //到这里说明f结点不为null，
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                       //头结点的hash值大于0说明是链表
                        if (fh >= 0) {
                            binCount = 1; // 用于记录链表长度
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //如果找到了相等的key，则判断是否需要覆盖
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //将新结点插入链表最后
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                           //红黑树
                            Node<K,V> p;
                            binCount = 2;
                            //调用红黑树的插值方法插入新节点
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
                    // 判断是否要将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }

```

####初始化table

```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
          //sizeCtl<0说明有其他线程正在初始化table
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
                //使用cas将sizeCtl设置为-1，代表初始化table。
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        //初始化数组，长度为默认的16或者初始化时提供的长度
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                         // 如果 n 为 16 的话，那么这里 sc = 12
                        // 其实就是 0.75 * n
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```


-----

### 参考资料：

[Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析
](https://javadoop.com/post/hashmap)