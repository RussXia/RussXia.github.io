---
layout: blog
title: "MySQL与B-Tree和B+Tree"
catalog: true
tag: [MySQL,数据结构,算法,2019]
---

# MySQL与B-Tree和B+Tree

MySQL支持多种存储引擎，如InnoDB，MyISAM，Memory等等。其中InnoDB支持事物安全(ACID)，支持行锁定和外键，因而成为了MySQL默认的存储引擎。(MyISAM是在MySQL 5.5.5 之前的默认引擎)

+ InnoDB是支持事物安全的存储引擎，支持行锁定和外键，支持 B-Tree/Hash/FullText 索引类型。因为行锁，所以相比较MyISAM更能支持高并发的读写，但是存储空间也比MyISAM高。
+ MyISAM不支持事物，也不支持外键，锁级别是表锁，这个引擎适合查询为主的业务。支持 B-Tree/FullText/R-tree 索引类型，强调快速读取。
+ Memory是内存级别的存储引擎，所以存储的数据量较小，对数据的一致性支持较差。锁级别为表锁，不支持事务。但访问速度非常快，并且默认使用 hash 索引。

其中，索引(index)是为了提高MySQL检索数据效率的一种数据结构。索引中的B-Tree索引使用的数据结构是多叉平衡树。其实InnoDB中使用的是B+Tree，是B-Tree的变种，至于为什么网上以及MySQL中查询所使用的索引得到的是B-Tree，可以参考：[https://dba.stackexchange.com/questions/204561/does-mysql-use-b-tree-btree-or-both](https://dba.stackexchange.com/questions/204561/does-mysql-use-b-tree-btree-or-both) 。

相较而言，B-Tree和B+Tree是有区别的，但是总的来说，在作为存储引擎的索引方面，还是B+Tree更合适。

+ B+Tree排序更快(非叶子结点不存储数据，底层链表)。
+ B-Tree在中间结点(非叶子结点)上的插入速度更快。

## 二叉树和平衡二叉树

关于二叉树以及其遍历，在我之前的博客有讲到过，可以参考：[https://russxia.github.io/2018/03/08/%E4%BA%8C%E5%8F%89%E6%A0%91%E5%8F%8A%E5%85%B6%E9%81%8D%E5%8E%86/](https://russxia.github.io/2018/03/08/二叉树及其遍历/)。那么平衡二叉树又是什么呢，和普通的二叉树相比，区别又是什么呢。

#### 概念

我们知道，二叉查找树的查询效率并不是严格的O(logN)的，当二叉树退化成链表的情况下，其最差的查找效率是O(N)，等同于顺序查找。所以为了提升查询效率，如果我们保证了每棵树都是一颗完全二叉树(除最后一层外，其余层都是满的)，那么查询的效率就可以保证更接近于O(longN)，因此引出了平衡二叉树的概念。

关于平衡二叉树的设计思路，可以参考百度百科的平衡二叉树：

> 对一棵查找树（search tree）进行查询/新增/删除 等动作, 所花的时间与树的高度h 成比例, 并不与树的容量 n 成比例。如果可以让树维持矮矮胖胖的好身材, 也就是让h维持在O(lg n)左右, 完成上述工作就很省时间。能够一直维持好身材, 不因新增删除而长歪的搜寻树, 叫做balanced search tree（平衡树）。

所以为了满足平衡二叉树的特点，在插入/删除新元素的时候，如果新插入/删除的元素使得当前的二叉树不再是一颗平衡二叉树，我们需要对树结点进行"旋转"，使其重新满足平衡二叉树，这就是树的左旋和右旋。

#### 总结

更多关于二叉平衡树常见的定义和实现，可以参考wiki的[平衡二叉搜索树]([https://zh.wikipedia.org/zh-cn/%E5%B9%B3%E8%A1%A1%E4%BA%8C%E5%85%83%E6%90%9C%E5%B0%8B%E6%A8%B9](https://zh.wikipedia.org/zh-cn/平衡二元搜尋樹))。

> **平衡二叉搜索树**(Balanced Binary Tree)是一种结构平衡的[二叉搜索树](https://zh.wikipedia.org/wiki/二叉搜索树)，是一种每个节点的左右两子[树](https://zh.wikipedia.org/wiki/树_(数据结构))高度差都不超过一的[二叉树](https://zh.wikipedia.org/wiki/二元樹)。能在O(logN)内完成插入、查找和删除操作，最早被发明的平衡二叉搜索树为[AVL树](https://zh.wikipedia.org/wiki/AVL树)。
>
> 常见的平衡二叉搜索树有：
>
> + [AVL树](https://zh.wikipedia.org/wiki/AVL树)
> + [红黑树](https://zh.wikipedia.org/wiki/紅黑樹)
> + [Treap](https://zh.wikipedia.org/wiki/Treap)
> + [节点大小平衡树(SBT)](https://zh.wikipedia.org/w/index.php?title=节点大小平衡树&action=edit&redlink=1)

## B-Tree

B-Tree是一颗多路平衡查找树，在百度百科的平衡二叉树中有说到：`对一棵查找树（search tree）进行查询/新增/删除 等动作, 所花的时间与树的高度h 成比例, 并不与树的容量 n 成比例。` 树的深度过大，意味着频繁的磁盘I/O读写，低下的查询效率。那么如何减少树的深度呢，一个基本的思路就是：采用多叉树的结构。

<b>一棵m阶的B-树 (m叉树)的特性如下</b>

>1. 树中每个结点最多含有m个孩子（m>=2）
>
>2. 除根结点和叶子结点外，其它每个结点至少有[ceil(m / 2)]个孩子(最多有m个孩子)(其中ceil(x)是一个向正无穷取整的函数)
>
>3. 若根结点不是叶子结点，则至少有2个孩子（特殊情况：没有孩子的根结点，即根结点为叶子结点，整棵树只有一个根节点）
>
>4. 所有叶子结点都出现在同一层，叶子结点不包含任何关键字信息(可以看做是外部接点或查询失败的接点，实际上这些结点不存在，指向这些结点的指针都为null)
>
>5. 每个非终端结点中包含有n个关键字信息： (n，P0，K1，P1，K2，P2，……，Kn，Pn)。其中：
>
>   ​    a)  Ki (i=1…n)为关键字，且关键字按顺序升序排序K(i-1)< Ki。 
>
>   ​    b)  Pi为指向子树根的接点，且指针P(i-1)指向子树种所有结点的关键字均小于Ki，但都大于K(i-1)。 
>
>   ​    c)  关键字的个数n必须满足： [ceil(m / 2)-1]<= n <= m-1。

<b>一颗4阶的B-Tree的插入和删除</b>

假设我们先插入了100、200、300三个元素，因为我们是一颗4阶树，所以根节点可以存放3个元素(key)。当插入第四个元素400时，200成为根节点，100成为左结点，300、400成为右结点。(特性3)

当连续插入元素500、600时，由于右子节点的key>3了，所以其中间结点400被移动到根节点，`|500|600|`构成新的右子节点。`300` 是和 `100` 、`|500|600|`同一级别的新节点(特性5)

删除元素的过程中，有可能会破坏了B-Tree的特性，所以需要考虑的情况会比插入的情况要复杂的多。

关于删除的过程，可以参考wiki：

+ [https://zh.wikipedia.org/wiki/B树#删除](https://zh.wikipedia.org/wiki/B树#删除)

其主要思路是先删除元素，删除的元素又需要去区分叶子结点和非叶子结点这种情况。删除元素后重新平衡，重新平衡从叶子结点开始向根节点进发，直到重新平衡为止。

## B+Tree

B+Tree是B-Tree的变形，是一种更适应数据存储和索引的数据结构。它和B-Tree的主要不同在于：

+ 所有关键字存储在叶子节点，非叶子节点不存储真正的data
+ 所有的非叶子结点，都可以看成是索引。
+ 所有的叶子结点依顺序增加了顺序指针，形成了一个链表。

为什么这样的改动，就能使B+Tree比B-Tree更适合数据存储和索引呢？原因主要有以下几点：

+ B+Tree的磁盘读写代价更低



参考内容:

> + [http://neoremind.com/2012/02/b树、b树与b树简介/](http://neoremind.com/2012/02/b树、b树与b树简介/)
>
> + [https://dba.stackexchange.com/questions/204561/does-mysql-use-b-tree-btree-or-both](https://dba.stackexchange.com/questions/204561/does-mysql-use-b-tree-btree-or-both)
> + [[https://zh.wikipedia.org/wiki/B%E6%A0%91](https://zh.wikipedia.org/wiki/B树)]([https://zh.wikipedia.org/wiki/B%E6%A0%91](https://zh.wikipedia.org/wiki/B树))
> + [http://data.biancheng.net/view/60.html](http://data.biancheng.net/view/60.html)
> + [https://zh.wikipedia.org/wiki/B树#删除](https://zh.wikipedia.org/wiki/B树#删除)

