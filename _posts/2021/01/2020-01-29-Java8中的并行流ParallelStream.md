---
layout: blog
title: "Java8中的并行流ParallelStream"
catalog: false
tag: [Java,2021,Java基础]
---
# Java8中的并行流ParallelStream

Java8中加入了Stream流操作，极大地提高了编程效率和程序的可读性。同时它又提供了串行和并行两种模式，来适应不同的业务场景。其中并行就是我们今天要说到的`ParallelStream`。

## ParallelStream的工作原理是什么

#### 什么是Stream

首先回顾下什么是Stream，Stream不是集合元素，也不保存数据，它更像是一个高级的Iterator，如同水流一样，单向，不可重复。
对于一个流来说，我们的操作一般分为两种类型：

+ Intermediate(中间)：一个流可以进行零次/多次的Intermediate操作，每次Intermediate操作返回一个新的流，交给下一个操作使用。这类操作是lazy的，也就是说没有Terminal操作不会被执行。
+ Terminal(终止)：一个流只能有一次Terminal操作，所以Terminal操作必定是一个流的最后一个操作。

除了这一类操作，还有一类Short-circuiting操作。主要指：
+ 对于一个 intermediate 操作，如果它接受的是一个无限大（infinite/unbounded）的 Stream，但返回一个有限的新 Stream。(e.g.:limit)
+ 对于一个 terminal 操作，如果它接受的是一个无限大的 Stream，但能在有限的时间计算出结果。(e.g.:findAny、findFirst)

对于这三种操作类型的分类：

- Intermediate：

  map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered

- Terminal：

  forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator

- Short-circuiting：

  anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 limit

#### ParallelStream简介


```java
IntStream.range(1, 20).parallel()
                .forEach((value) -> {
                    String name = Thread.currentThread().getName();
                    System.out.println(name + "-----" + value);
                });
```

上面是一个非常简单的`ParallelStream`使用的demo，ParallelStream就是一个并行的流，它和Stream唯一的区别就是Stream是并行的，而ParallelStream是并行的。

![3igzmhuynj](/Users/dasouche/Downloads/3igzmhuynj.png)

+ ForkJoinPool.commonPool
    + 分治思想
+ ForkJoinPool的并发度设置
    + `java.util.concurrent.ForkJoinPool.common.parallelism`
    + `parallelism = Runtime.getRuntime().availableProcessors() - 1`




## ParallelStream有什么问题
+ I/O密集型&CPU密集型

+ 线程安全


## 总结


