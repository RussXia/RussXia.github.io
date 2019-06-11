---
layout: blog
title: "HashMap中的Hash冲突解决和扩容机制"
tag: [Java,Java面试]
---
# 关于HashMap
HashMap根据key的hash值来存储数据，HashMap最多只允许一个key为null的记录。HashMap是线程不安全的。HashMap的数据结构是:数组+链表+红黑树（JDK1.8增加了红黑树部分）。

HashMap中的几个关键属性如下:
```java
    //hash表
    transient Node<K,V>[] table;

    transient Set<Map.Entry<K,V>> entrySet;
    
    //map中包含的key-value个数
    transient int size;

    //记录当前HashMap内部结构发生变化的次数，主要用于fail-fast的判断
    transient int modCount;

    //扩容阈值，capacity * load factor
    //默认容量:DEFAULT_INITIAL_CAPACITY = 1 << 4
    int threshold;

    //负载因子
    final float loadFactor;

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; //用来定位数组索引位置
        final K key;
        V value;
        Node<K,V> next; //链表的下一个node
    }
```
HashMap的默认初始容量时1<<4(16),HashMap的容量一定时2的n次幂。(这是为了方便hash计算,具体可以看[https://russxia.github.io/2019/04/29/TreeSet-LinkedHashSet-HashSet的区别.html]中关于hash函数的描述)。

HashMap使用哈希表来存储数据。Hash算法能将任意数据散列映射到有限的空间上，因此在快速查找和加密中使用广泛。为了解决哈希冲突，可以采用"开放地址法"或者"链地址法"等来解决问题。HashMap中采用的是`链地址法`。

# 关于Hash算法和Hash冲突
<B>Hash算法</B>:就是根据设定的Hash函数H(key)和处理冲突方法，将一组关键字映射到一个有限的地址区间上的算法。所以Hash算法也被称为散列算法、杂凑算法。

<B>Hash表</B>:通过Hash算法后得到的有限地址区间上的集合。数据存放的位置和key之前存在一定的关系(`H(key)=stored_value_hash(数据存放位置)`),可以实现快速查询。与之相对的，如果数据存放位置和key之间不存在任何关联关系的集合，称之为`非Hash表`。

<B>Hash冲突</B>:由于用于计算的数据是无限的`H(key),key属于(-∞,+∞)`,而映射到区间是有限的，所以肯定会存在两个key:key1,key2，H(key1)=H(key2)，这就是hash冲突。一般的解决Hash冲突方法有:开放定址法、再哈希法、链地址法（拉链法）、建立公共溢出区。

## 开放地址法
开放定址法也称为`再散列法`，基本思想就是，如果`p=H(key)`出现冲突时，则以`p`为基础，再次hash，`p1=H(p)`,如果p1再次出现冲突，则以p1为基础，以此类推，直到找到一个不冲突的哈希地址`pi`。 因此开放定址法所需要的hash表的长度要大于等于所需要存放的元素，而且因为存在再次hash，所以`只能在删除的节点上做标记，而不能真正删除节点。`

缺点:容易产生堆积问题;不适合大规模的数据存储;插入时会发生多次冲突的情况;删除时要考虑与要删除元素互相冲突的另一个元素，比较复杂。


## 再哈希法(双重散列，多重散列)
提供多个不同的hash函数，当`R1=H1(key1)`发生冲突时，再计算`R2=H2(key1)`，直到没有冲突为止。
这样做虽然不易产生堆集，但增加了计算的时间。

## 链地址法(拉链法)
链地址法:将哈希值相同的元素构成一个同义词的单链表,并将单链表的头指针存放在哈希表的第i个单元中，查找、插入和删除主要在同义词链表中进行。链表法适用于经常进行插入和删除的情况。HashMap采用的就是链地址法来解决hash冲突。(链表长度大于等于8时转为红黑树)

## 建立公共溢出区
将哈希表分为公共表和溢出表，当溢出发生时，将所有溢出数据统一放到溢出区。

# HashMap中的处理冲突
下面是HashMap的put方法:
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) //如果hash数组为空，初始化一下
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)  //计算落在hash桶的位置，如果当前桶为空，直接新增节点
        tab[i] = newNode(hash, key, value, null);
    else {  //当前桶存在元素
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) 
            //如果key已经存在，替换元素
            e = p;
        else if (p instanceof TreeNode) //如果当前是树结构了(不是链表了),向树上添加元素
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {  //当前结构依然时链表，遍历链表，直到末尾或者找到key相同的元素替换
            for (int binCount = 0; ; ++binCount) {
                //到达末尾，新增元素，如果链表长度达到8，转为红黑树
                if ((e = p.next) == null) { 
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //遍历链表的过程中，发现了有key相同的元素，直接替换，然后break
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { //如果是已经存在的元素，判断是否替换(onlyIfAbsent)
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //如果容量超过阈值，扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

# HashMap中的扩容机制
当HashMap中元素的个数达到了阈值，(capacity * load factor)，HashMap中，元素存放的位置与hash数组的个数是有关系的(`tab[i = (n - 1) & hash]`)，所以当发生扩容是，hash数组的个数发生了变化，这个时候，元素也需要重新进行hash计算。
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //老的容量和阈值
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    //计算新的容量和阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    //构建新的hash桶
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        //遍历原由的hash桶，copy元素到新的里面
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;   //原来的hash桶置空
                if (e.next == null) //原来位置上只有一个元素(没有链表和树),直接放到新的位置
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) //如果是树状结构，单独处理
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order 这里的是链表元素
                    Node<K,V> loHead = null, loTail = null; //原来index位置
                    Node<K,V> hiHead = null, hiTail = null; //新的index位置
                    Node<K,V> next;
                    //循环处理链表元素，判断元素是放在原index位置还是放在新index位置
                    do {
                        next = e.next;
                        //放在原index的位置的元素，loHead拿到链表头，loTail拿到链表尾
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //放在新index位置的元素，hiHead拿到链表头，hiTail拿到链表尾
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                     //放入原index处
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //放入新index处
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
其中比较有意思的是，由于put时计算hash数组角标是通过`i = (n - 1) & hash`计算的，其中n是的2的x次幂，扩容时，容量变为原来的两倍,`n-1 == (n-1)<<1 & 1`，所以hash桶中的元素，要么还在原来的index位置，要么在`oldIndex+oldCap`的位置。所以放入新的index时，元素下表:`newTab[j + oldCap]`。

关于treenode，HashMap在扩容时，如果当前节点是棵树，先还是和链表的一样，将所有元素分为两批(一批还在当前index，另一批在新的index位置，此处传入的bit=oldCap，所以和链表的还是蛮类似的)。分成的两组，分别判断当前元素个数是否小于阈值，如果是，还原成链表。
```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
        TreeNode<K,V> b = this;
        // Relink into lo and hi lists, preserving order
        TreeNode<K,V> loHead = null, loTail = null;
        TreeNode<K,V> hiHead = null, hiTail = null;
        int lc = 0, hc = 0;
        for (TreeNode<K,V> e = b, next; e != null; e = next) {
            next = (TreeNode<K,V>)e.next;
            e.next = null;
            if ((e.hash & bit) == 0) {
                if ((e.prev = loTail) == null)
                    loHead = e;
                else
                    loTail.next = e;
                loTail = e;
                ++lc;
            }
            else {
                if ((e.prev = hiTail) == null)
                    hiHead = e;
                else
                    hiTail.next = e;
                hiTail = e;
                ++hc;
            }
        }

        if (loHead != null) {
            if (lc <= UNTREEIFY_THRESHOLD)
                tab[index] = loHead.untreeify(map);
            else {
                tab[index] = loHead;
                if (hiHead != null) // (else is already treeified)
                    loHead.treeify(tab);
            }
        }
        if (hiHead != null) {
            if (hc <= UNTREEIFY_THRESHOLD)
                tab[index + bit] = hiHead.untreeify(map);
            else {
                tab[index + bit] = hiHead;
                if (loHead != null)
                    hiHead.treeify(tab);
            }
        }
}
```
