---
layout: blog
title: "TreeSet、HashSet、LinkedHashSet的区别"
tag: Java面试
---
# Set接口
TreeSet、HashSet、LinkedHashSet都实现了Set接口。Set接口和List接口一样继承自Collection接口。

## List和Set的区别
关于list，jdk中的描述是这样的:`An ordered collection (also known as a <i>sequence</i>).Unlike sets, lists typically allow duplicate elements`。即list是有序的集合,和set不同的是list是允许重复的元素(因为list以index获取元素)。

关于set，jdk中的描述是这样的:`A collection that contains no duplicate elements. More formally, sets
 contain no pair of elements <code>e1</code> and <code>e2</code> such that
 <code>e1.equals(e2)</code>, and at most one null element.`，set不允许包含重复的元素，确切地说，set集合不允许存在两个元素e1,e2，e1.equals(e2)；至多允许一个null值。
```
public interface Set<E> extends Collection<E> 
```

## HashSet
HashSet是依靠hash table实现的(内部实现实际上是一个HashMap)。
不保证顺序(hash无法保证顺序)。
允许null值。
因为其实现借助于hash表，所以两个元素,`e1.equals(e2)`,必须也要保证 `e1.hashCode() == e2.hashCode()`

## LinkedHashSet
LinkedHashSet继承自HashSet，不过和HashSet不同的是，它是借助于LinkedHashMap实现的(LinkedHashMap其实也继承自HashMap,但是它依靠双向链表，实现了插入顺序有序)。所以它的特性和HashSet类似，但是LinkedHashSet是有序的，因为有了链表，所以在遍历元素的时候LinkedHashSet要优于HashSet，但是在插入时，增加了维护链表的开销，所以插入性能略差于HashSet。
```java
/**
 * 依靠Hash table和linked list实现的set集合，支持顺序。
 * 和HashSet的区别是内部维护了一个双向链表，链表的顺序就是插入的顺序。
 * 当一个元素重新插入set时，其内部顺序不受影响
 */
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```
## TreeSet
不同与HashSet的无序，LinkedHashSet的插入有序，TreeSet是有序的集合，其实现了SortedSet接口。TreeSet借助于TreeMap实现。Set内的元素按照 `Comparable` 或者 传入构造器的`Comparator`的顺序排序。TreeSet不支持null元素(因为要排序嘛)。

# HashMap、LinkedHashMap、TreeMap的数据结构
## HashMap
HashMap是基于hash table实现的map，允许空值空键。HashMap除了允许为空以及非同步，其他和HashTable完全一致。HashMap是不保证顺序不变的。
在jdk1.8中，HashMap的数据结构是数组(hash划分)+链表+二叉树。当链表的长度大于等于8时，链表转换成红黑树。
![](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/hashmap_struct.jpg)

## LinkedHashMap
LinkedHashMap继承自HashMap，在HashMap的基础上维护了一条双向链表，解决了HashMap不能保持插入顺序的问题。
![](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/linkedhashmap_struct.jpg)

## TreeMap
TreeMap基于红黑树的数据结构实现。红黑树是一颗二叉树，同时红黑树更是一颗自平衡的排序二叉树。
+ 1.每个节点都只能是红色或者黑色
+ 2.根节点是黑色
+ 3.每个叶节点（NIL节点，空节点）是黑色的。
+ 4.如果一个结点是红的，则它两个子节点都是黑的。也就是说在一条路径上不能出现相邻的两个红色结点。
+ 5.从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。
```java
public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        //已经存在的节点，替换旧值
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        //新增节点
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        fixAfterInsertion(e);   //红黑树平衡调整
        size++;
        modCount++;
        return null;
    }
```


## 关于hash函数
HashMap的散列是依靠hash实现的，下面是HashMap对key进行hash的代码。
做一次16位右位移异或混合。(扰动函数)
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
`key.hashCode()`是key键自带的hash函数，返回一个int型的散列值。int类型的取值范围有2^32个，大概有40亿的映射空间。但是一个大约40亿的数组内存是不适合存放的。HashMap是对数组的长度做取模运算，得到的余数用来访问数组下标。
```java
Node<K,V>[] tab; Node<K,V> p; int n, i; //n就是数组的长度
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
if ((p = tab[i = (n - 1) & hash]) == null)  //i = (n - 1) & hash,用散列值和数组长度做一个"与"操作
    tab[i] = newNode(hash, key, value, null);
```
因为HashMap的数组长度是2的n次幂(2,4,8,16),所以"与"操作，散列值的高位全部归零，只保留低位用作数组的下标访问。这个是长度为16时，ll
```
        10100101 11000100 00100101
&       00000000 00000000 00001111
----------------------------------
        00000000 00000000 00000101    //高位全部归零，只保留末四位
```
