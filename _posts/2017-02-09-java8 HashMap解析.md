---
layout: post
title:  "java HashMap解析"
date:   2017-02-09 10:02:00 +0800
categories: java HashMap
tags:
 - java
 - Map
---

# java HashMap解析

HashMap是java中常用且相对重要的类之一。了解此类的数据结构及储存原理对我们写程序有莫大帮助。java8中又对此类底层实现进行了优化，比如引入了红黑树的结构以解决哈希碰撞。今天我们就从底层解析一下HashMap，希望对大家有所帮助。


## HashMap的数据结构

### 1. HashMap整体结构

Map是java中的储存键（key）、值(value)对数据结构。而HashMap即是通过key的hash值确定value的储存位置。在理想情况下，仅需要O(1)的时间就可以通过key定位到value值。不过，这里一个显著的问题是，不同的key也可能有相同的哈希值，HashMap采用**数组+链表**解决。


![这里写图片描述](http://img.blog.csdn.net/20170209082753880?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3RoZmVuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


如图，HashMap的主结构类似于一个数组，添加值时通过key确定储存位置。每个位置是一个Node<V>（图中黑点）的数据结构，该结构可组成链表。当发生冲突时，相同hash值的键值对会组成链表。

这种**数组+链表**的组合形式大部分情况下都能有不错的性能效果，java6、7就是这样设计的。然而，在极端情况下，一组（比如经过精心设计的）键值对都发生了冲突，这时的哈希结构就会退化成一个链表，使HashMap性能急剧下降。



所以在java8中，HashMap的结构实现变为**数组+链表+红黑树**。如图：

![这里写图片描述](http://img.blog.csdn.net/20170209082825542?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3RoZmVuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当链表达到一定长度，会将链表转为红黑树。我们知道链表的查询时间为O(n)，而红黑树的查询时间为O(logN)。当长度大到一定程度时，红黑树的优势会更加明显。

### 2. 类概览

在具体实现上，HashMap有许多内部类、方法及字段。下面列举一些比较重要的。

```java

//默认Map容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//链表转为红黑树的临界值
static final int TREEIFY_THRESHOLD = 8;

//数组，HashMap的主要储存结构
transient Node<K,V>[] table;

//节点，即HashMap的键值对的储存结构
static class Node<K,V> implements Map.Entry<K,V> 

//红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>

//用于计算key的哈希值
static final int hash(Object key)

//添加新键值对
public V put(K key, V value)

//删除键值对
public boolean remove(Object key, Object value)

```

### 3. Node<K,V>结构

Node<K,V>是HashMap的内部类，也是其键值对的底层实现。类声明如下：

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;  //指向该链表的下一个node

        Node(int hash, K key, V value, Node<K,V> next) {}
        public final K getKey()        {}
        public final V getValue()      {}
        public final String toString() {}
        public final int hashCode() {}
        public final V setValue(V newValue) {}  
        public final boolean equals(Object o) {}
           
    }
```
如此，HashMap的数组+链表结构就大致成形了，Node[]为数组，而Node又可连成链表。

### 4. TreeNode<K,V> 红黑树结构

TreeNode<K,V> 是红黑树的结构实现，类声明如下：

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {}
        
        //以下省略其他方法
 }
        
```

红黑树结构包含前、后、左、右节点，以及标志是否为红黑树的字段。此结构是java8新加的。




## HashMap的实现

### Put的实现

某一键值对<K,V>,添加到map中。

工作流程可概括为以下几点：

1. 根据K的哈希算法确定该键值对在数组（HashMap）中的索引位置x。
2. 若索引位置x为空，将<K,V>添加于此，结束。若x不为空，转向3
3. 判断x处的值是否等于V,若等于，用V覆盖原值。结束。否则，转向4
4. 在x处遍历链表，并在尾部插入<K,V>。判断链表长度是否大于`TREEIFY_THRESHOLD`,若小于，结束。若大于，将该链表转为红黑树结构,结束。

下面我们结合代码详细分析一下此过程。

### 1. 通过hash值定位元素位置

对于通过hash定位储存的Map,哈希算法对其性能有很大影响。好的哈希算法可以尽可能避免冲突的发生，使读取效率保持在O(1)，下面是HashMap的哈希过程。

为表述方面,键值对设为("hello","world")。put方法源码为

```java

  public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
```
由此可见，先对`hello`进行哈希操作。hash()源码为

```java
  static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
随后，put()过程中有一步异或操作。

>  i = (n - 1) & hash

n是HashMap底层数组的长度，当n为2的次方时，`(n-1)&hash`等价于`hash%n`,可确保得到的值落在数组索引范围内。

例如，对`hello`进行哈希计算为`99163451`。进行索引计算为`11`,即(`hello`,`world`)会落在数组索引为11的位置。

### 2. put过程

废话不说，先上代码

```java
 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;  //若底层数组还没有元素，先扩容
        if ((p = tab[i = (n - 1) & hash]) == null)  //这就是前面提到的索引的计算，判断此位置是否有值。
            tab[i] = newNode(hash, key, value, null); //若此位置无值，添加节点，对应步骤2
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;  //若此位置有值，且与要添加的值相等，覆盖，对应步骤3
            else if (p instanceof TreeNode) //这里查看节点类型，若是TreeNode,说明已经是红黑树，调用红黑树添加节点即可。
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else { //仍是链表，遍历，若发现有值相同的，覆盖，否则直接将节点加在链表最后。
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) { //若其后无值了，在后面添加要添加的节点
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash); //判断链表长度是否足够转为红黑树
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;  //若遍历过程中发现有与添加的值相同，覆盖
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
        if (++size > threshold)
            resize();  //若长度超过扩容阈值，进行扩容。
        afterNodeInsertion(evict);
        return null;
    }


```

### 3. 扩容

当初始化数组或数组大小到达一定程度时，都会引发扩容机制。

```java
 final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //根据情况判断新数组大小
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) { //若容量已超过最大值，已无法扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; //否则，扩大为原来2倍
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // oldCap、oldThr为0时默认为初始值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; //构建新数组
        table = newTab;
        if (oldTab != null) { //将旧的值移到新数组中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null) //若该位置有值且只有一个（不是链表或红黑树）
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode) //若是红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 若是链表
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

有关红黑树及链表重新扩容的算法在下篇文章中会有介绍，HashMap扩容的大致流程如上面注解那样，需考虑当前容量及数据结构。

### 4.java8的性能优化

HashMap经java8的优化后，解决了哈希碰撞的问题。在哈希均匀分布的情况下，java7和java8对HashMap的性能测试中表现类似，而在哈希极端分布的情况下，java8的HashMap具有明显的性能优势。所以，如果可以的话，应选用java8的HashMap。



--------------全文完------------------


参考文章

1. [Java 8系列之重新认识HashMap](http://tech.meituan.com/java-hashmap.html)
2. [Java 8：HashMap的性能提升](http://www.importnew.com/14417.html)

 










