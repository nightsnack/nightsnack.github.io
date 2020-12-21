---
title: Java HashMap 源码学习笔记
date: 2018-02-07 16:16:49
tags: 
    - java源码
    - java
---

##  Java HashMap 源码学习笔记
<!-- more -->

map子类中的hashmap，源码学习记录。部分是翻译还有一些是自己的记录。

写起来才发现好坑啊，网上很多是1.7的，我看的是1.8的，没想到1.7根1.8差的还挺多的。1.7源码都没看过的我…🙁

### 1.常量初始化 

The default initial capacity - MUST be a power of two.\

最小初始容量，必须是2的指数次幂

```java 
     static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16 
```



The maximum capacity, used if a higher value is implicitly specified by either of the constructors with arguments. MUST be a power of two <= 1<<30.

最大容量，必须是2的指数次幂，**为什么呢？**见hash函数

```java
     static final int MAXIMUM_CAPACITY = 1 << 30;
```


The load factor used when none specified in constructor.

负载平衡，是时间复杂度和空间复杂度的平衡，oracle官方给出的是0.75是最佳。过高的因子会降低存储空间但是查找（lookup，包括HashMap中的put与get方法）的时间就会增加

```java
     static final float DEFAULT_LOAD_FACTOR = 0.75f;
```


The bin count threshold for using a tree rather than list for a bin. Bins are converted to trees when adding an element to a bin with at least this many nodes. The value must be greater than 2 and should be at least 8 to mesh with assumptions in tree removal about conversion back to plain bins upon shrinkage.

当节点个数>= TREEIFY_THRESHOLD - 1时，HashMap将采用红黑树存储。为什么这么做呢？当发生Hash冲突时，HashMap首先是采用链表将重复的值串起来，并将最后放入的值置于链首，java8对HashMap进行了优化。当节点个数多了之后使用红黑树存储。这样做的好处是，最坏的情况下即所有的key都Hash冲突，采用链表的话查找时间为O(n),而采用红黑树为O(logn)，这也是Java8中HashMap性能提升的奥秘

```java
     static final int TREEIFY_THRESHOLD = 8; 
```



The bin count threshold for untreeifying a (split) bin during a resize operation. Should be less than TREEIFY_THRESHOLD, and at most 6 to mesh with shrinkage detection under removal.

```java
     static final int UNTREEIFY_THRESHOLD = 6;
```

The smallest table capacity for which bins may be treeified. (Otherwise the table is resized if too many nodes in a bin.) Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts between resizing and treeification thresholds.

重新分配大小

```java
     static final int MIN_TREEIFY_CAPACITY = 64;
```


HashMap性能的有两个参数：初始容量(initialCapacity) 和加载因子(loadFactor)。容量 是哈希表中桶的数量，初始容量只是哈希表在创建时的容量。加载因子 是哈希表在其容量自动增加之前可以达到多满的一种尺度。当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 rehash 操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数。

### 2.Hash函数

Computes key.hashCode() and spreads (XORs) higher bits of hash to lower. Because the table uses power-of-two masking, sets of hashes that vary only in bits above the current mask will always collide. (Among known examples are sets of Float keys holding consecutive whole numbers in small tables.) So we apply a transform that spreads the impact of higher bits downward. There is a tradeoff between speed, utility, and quality of bit-spreading. Because many common sets of hashes are already reasonably distributed (so don't benefit from spreading), and because we use trees to handle large sets of collisions in bins, we just XOR some shifted bits in the cheapest possible way to reduce systematic lossage, as well as to incorporate impact of the highest bits that would otherwise never be used in index calculations because of table bounds.

```java

static final int hash(Object key) {   //jdk1.8 & jdk1.7
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
     return h & (length-1);  //第三步 取模运算
}

```

在哈希表容量（也就是buckets或slots大小）为length的情况下，为了使每个key都能在冲突最小的情况下映射到`[0,length)`（注意是左闭右开区间）的索引（index）内，一般有两种做法：

1. 让length为素数，然后用`hashCode(key) mod length`的方法得到索引
2. 让length为2的指数倍，然后用`hashCode(key) & (length-1)`的方法得到索引

[HashTable](http://docs.oracle.com/javase/7/docs/api/index.html?java/util/Hashtable.html)用的是方法1，`HashMap`用的是方法2。

下图，n为哈希列表长度

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fo6yr3pfc8j30xa0j0q5u.jpg)

也就是高16位与低16位XOR（其实是原值和高16位右移16位后的值XOR）后再和 length-1 &运算，length是2 的指数次幂，最小是16，也就是说length-1的末4为全1，32就是末8位全1，以此类推。



为什么要这么做呢？

> 如果`hashCode(key)`的大于`length`的值，而且`hashCode(key)`的二进制位的低位变化不大，那么冲突就会很多，举个例子：
>
>  Java中对象的哈希值都32位整数，而HashMap默认大小为16，那么有两个对象那么的哈希值分别为：`0xABAB0000`与`0xBABA0000`，它们的后几位都是一样，那么与16异或后得到结果应该也是一样的，也就是产生了冲突。
>
> 造成冲突的原因关键在于16限制了只能用低位来计算，高位直接舍弃了，所以我们需要额外的哈希函数而不只是简单的对象的`hashCode`方法了。
>  具体来说，就是HashMap中`hash`函数干的事了.



### 3. List式的bucket Node

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

bucket是一个list，每个元素是一个节点，用于开散列法解决冲突（如果冲突就在这个节点后面链式存储）

所以每个元素有一个next节点，也是一个Node类。

### 4.get

```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * k to a value v such that  (key==null ? k==null :
     * key.equals(k)), then this method returns v; otherwise
     * it returns null.  (There can be at most one such mapping.)
     *
     * <p>A return value of null does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to null.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     * 这段话意思就是说有的key是空的，但是不代表这个map没有建立关联的关系
     
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

	/**
	* The table, initialized on first use, and resized as necessary. When allocated, 		length is always a power of two. (We also tolerate length zero in some operations to 	 allow bootstrapping mechanics that are currently not needed.)
	*/
	transient Node<K,V>[] table;
	

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key 哈希值
     * @param key the key 原值
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {    //数组非空，把数组赋给tab，长度大于0，而且头结点(用哈希值和数组长度异或)非空
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;		//看first的key是否等于传入的key
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)		//如果是红黑树方式存储的
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);	//调用红黑树寻找treenode
                do {				//链式存储的话链式寻找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }		//找不到或者长度为空什么的返回空
        return null;
    }
```

### 5.put

![hashMap put方法执行流程图](https://ws4.sinaimg.cn/large/006tKfTcgy1fo7ooddqq6j31bo11s7cj.jpg)



①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

```java
/**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;	//如果tab为空，先创建tab
        if ((p = tab[i = (n - 1) & hash]) == null)		//如果tab中当前hash值不存在先创建该节点
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))		//对于链表的头节点或者树根，如果该key存在，直接覆盖value
                e = p;
            else if (p instanceof TreeNode)		//如果p是红黑树节点，用红黑树的添加方式添加
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {		//链表，顺序添加
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st //链表长度大于8转换为红黑树进行处理
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))		//链表中存在该key
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)		//超过最大容量，扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```

### 5.resize 扩容



使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中**hash1是key1对应的哈希与高位运算结果**。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fo7q9z8yzsj319c0ce761.jpg)



元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![b2cb057773e3d67976c535d6ef547d51_hd](https://ws3.sinaimg.cn/large/006tKfTcly1g0l59m14g8j30k003taao.jpg)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：



![544caeb82a329fa49cc99842818ed1ba_hd](https://ws4.sinaimg.cn/large/006tKfTcly1g0l58u0adjj30k00bjjtk.jpg)</u>

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
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length; //旧容量，旧Thr(Resize上限)，超过这个threshold就要开始扩容了
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {	//超过最大值直接把threshold置为integer最大值，不再扩容了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }		//向左移一位，扩成原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
          						//如果oldThreshold初始化了，那新的容量初始化为threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults否则用默认值初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
      // 计算新的resize上限newThr
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;	//初始化新的table
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) { 
                              //这里为什么是 & oldCap，图上画的是表长-1，表长16的时候扩容32应该和31 &，是因为要判断是否为0，比如说5和21，&31的时候是 一个是5，一个是21，不好判断，但是如果 &16，5&16是0，21&16是 16，这样就可以判断了。
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }			//如果不需要移位，找个链表lo链起来
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }			//如果需要移位，找个链表hi链起来
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }			//lo表的不动
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }			//hi表的放在加上旧表容量的位置
                    }
                }
            }
        }
        return newTab;
    }

    
```

### 6.Remove

Remove的方法思路和get差不多，只是最后是--size

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

    /**
     * Implements Map.remove and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
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

### 7.线程安全

#### Java 1.7

在多线程使用场景中，应该尽量避免使用线程不安全的HashMap，而使用线程安全的ConcurrentHashMap。那么为什么说HashMap是线程不安全的，下面举例子说明在并发的多线程使用场景中使用HashMap可能造成死循环。代码例子如下(便于理解，仍然使用JDK1.7的环境)：

```java
public class HashMapInfiniteLoop {  

    private static HashMap<Integer,String> map = new HashMap<Integer,String>(2，0.75f);  
    public static void main(String[] args) {  
        map.put(5， "C");  

        new Thread("Thread1") {  
            public void run() {  
                map.put(7, "B");  
                System.out.println(map);  
            };  
        }.start();  
        new Thread("Thread2") {  
            public void run() {  
                map.put(3, "A);  
                System.out.println(map);  
            };  
        }.start();        
    }  
}
```

其中，map初始化为一个长度为2的数组，loadFactor=0.75，threshold=2*0.75=1，也就是说当put第二个key的时候，map就需要进行resize。

通过设置断点让线程1和线程2同时debug到transfer方法(3.3小节代码块)的首行。注意此时两个线程已经成功添加数据。放开thread1的断点至transfer方法的“Entry next = e.next;” 这一行；然后放开线程2的的断点，让线程2进行resize。结果如下图。
![jdk1.7 hashMap死循环例图1](https://ws2.sinaimg.cn/large/006tKfTcly1g0l53xl5zbj30k0092757.jpg)


注意，Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。

线程一被调度回来执行，先是执行 newTalbe[i] = e， 然后是e = next，导致了e指向了key(7)，而下一次循环的next = e.next导致了next指向了key(3)。

![jdk1.7 hashMap死循环例图2](https://ws2.sinaimg.cn/large/006tKfTcly1g0l54uk4i4j30k007k0t2.jpg)

e.next = newTable[i] 导致 key(3).next 指向了 key(7)。注意：此时的key(7).next 已经指向了key(3)， 环形链表就这样出现了。
![5f3cf5300f041c771a736b40590fd7b1_hd](https://ws4.sinaimg.cn/large/006tKfTcly1g0l57eeta1j30k006zwf4.jpg)


于是，当我们用线程一调用map.get(11)时，悲剧就出现了——Infinite Loop。



#### Java1.8

java1.8在resize的时候不再使用重新计算每个Hash值的方法，那么而且，它是把新元素插在链表的尾部，尾部的next指向null，那么久不会出现所谓无线循环。但依然是线程不安全的。

详见：*http://blog.csdn.net/u010412719/article/details/52049347*

但是除了Infinite Loop，HashMap依然存在另一个问题：

**fail-fast**

如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。

这个异常意在提醒开发者及早意识到线程安全问题，具体原因请查看ConcurrentModificationException的原因以及解决措施



### 8. 红黑树

红黑树部分等我学了红黑树再说吧。