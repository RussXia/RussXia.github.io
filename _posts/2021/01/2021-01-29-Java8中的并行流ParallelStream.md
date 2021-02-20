---
layout: blog
title: "Java8中的并行流ParallelStream"
catalog: true
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

上面是一个非常简单的`ParallelStream`使用的demo，ParallelStream就是一个并行的流，它和Stream唯一的区别就是Stream是串行的，而ParallelStream是并行的。ParallelStream使用的`ForkJoinPool.commonPool`(默认为当前核心数-1)。

commonPool的并发度设置:
+ `java.util.concurrent.ForkJoinPool.common.parallelism`
+ `parallelism = Runtime.getRuntime().availableProcessors() - 1`

```java
public static void main(String[] args) {
        Set<Thread> threadSet = new HashSet<>();
        IntStream.range(1, 10).parallel()
                .forEach(record -> {
                    System.out.println(Thread.currentThread().getName() + "--" + record);
                    threadSet.add(Thread.currentThread());
                });
        System.out.println("thread size-" + threadSet.size());
    }
```

ParallelStream使用了main线程作为执行线程和ForkJoinPool.commonPool创建的线程。

```text
ForkJoinPool.commonPool-worker-1--3
main--6
ForkJoinPool.commonPool-worker-1--4
ForkJoinPool.commonPool-worker-3--2
ForkJoinPool.commonPool-worker-2--8
ForkJoinPool.commonPool-worker-3--1
ForkJoinPool.commonPool-worker-1--7
main--5
ForkJoinPool.commonPool-worker-2--9
thread size-4
```

![3igzmhuynj](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/3igzmhuynj.png)

ForkJoinPool采用分治的思想，即将一个大任务，拆分成多个"小任务"并行计算，再把多个小任务的计算的结果合并成总的计算结果。Fork对应图上的切分部分，Join对应其中的合并结果。(可以理解为一个单机版的map-reduce)类似于ThreadPoolExecutor 和 Runnable ，ForkJoinPool会接受一类特殊的task--`ForkJoinTask`。

ForkJoinPool和其他`ExecutorService`实现类的最大区别就是`work-stealing`(工作窃取)算法。work-stealing算法，是指某个线程，从其他线程的工作队列(workQueue)里窃取任务来执行。

`workQueue`主要是用来保存向ForkJoinPool提交的任务，具体任务的执行是由`ForkJoinWorkerThreadFactory`生成的`ForkJoinWorkerThread`执行的。

Fork/Join中最核心的就是fork()方法和join()方法。
+ fork()方法会将当前产生的新任务(新任务调用fork()方法)push到工作队列的队尾。(LIFO)
+ join()方法当任务结束时，返回结果；如果需要的任务尚未完成，阻塞当前线程等待子任务的执行结果。


## ParallelStream有什么问题

通过前面的介绍，我们知道了，ParallelStream是用`ForkJoinPool.commonPool`进行多线程计算的。那么这里面存在什么问题呢？

#### commonPool的并发度

commonPool的默认并发度是按照当前系统cpu的核心数-1设置的。你可以通过设置`java.util.concurrent.ForkJoinPool.common.parallelism`参数调整commonPool默认并发度。但是这是一个全局配置，因此它也会影响所有的ParallelStream。

对于CPU密集型的操作，并发度为系统的cpu核心数-1是非常合适的。但是如果是IO密集型的，且任务数相较于线程数大很多，那么ParallelStream并不是很好的选择。

#### 线程安全

ParallelStream是多线程的，所以也会存在线程安全的问题。
```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
List<Calendar> list = new ArrayList<>();
for (int i = 0; i < 5; i++) {
    Calendar startDay = new GregorianCalendar();
    Calendar checkDay = new GregorianCalendar();
    checkDay.setTime(startDay.getTime());
    checkDay.add(Calendar.DATE,i);
    list.add(checkDay);
}

list.forEach(day ->  System.out.println(sdf.format(day.getTime())));
System.out.println("-----------------------");
list.parallelStream().forEach(day ->  System.out.println(sdf.format(day.getTime())));
System.out.println("-----------------------");
```
输出
```text
2021-02-03
2021-02-04
2021-02-05
2021-02-06
2021-02-07
-----------------------
2021-02-03
2021-02-03
2021-02-03
2021-02-06
2021-02-05
-----------------------
```

在这段Java代码中，我们以当天的日期作为开始，向后加了5天。在单线程输出时，输出正常。但是换成多线程格式化输出时，输出异常。因为`SimpleDateFormat`不是线程安全的。

## 总结

1. ParallelStream适合CPU密集型的计算，不适合IO密集型
2. ParallelStream是多线程的，注意线程安全。
3. ParallelStream不是顺序执行的。
4. `commonPool`是共享的，避免在其中执行阻塞任务，导致其他任务无法立即执行
5. 不要在同一线程池混合运行阻塞任务和非阻塞任务。(commonPool不要执行阻塞任务)


