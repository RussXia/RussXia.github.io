---
layout: blog
title: "CountDownLatch、CyclicBarrier与Semaphore"
tag: [Java,Java面试]
---
# CountDownLatch
CountDownLatch可以实现多线程之间的计数功能，并实现了阻塞功能。其内部通过实现了AQS的`java.util.concurrent.CountDownLatch.Sync`保证线程之间同步修改count值。

在构造CountDownLatch的时候，传入计数值，并在构造函数内完成sync的初始化。
```java
 public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
CountDownLatch提供了两个核心的方法，`countDown`和`await()`。每次调用countDown()，count值都会减1，当count值减为0时，所有await的线程从等待中恢复。
```java
private static class Printer implements Runnable {
    private final CountDownLatch count;

    private Printer(CountDownLatch count) {
        this.count = count;
    }

    public void run() {
        //do something
        System.out.println(Thread.currentThread().getName() + " ready");
        count.countDown();
    }
}

public static void main(String[] args) throws InterruptedException {
    CountDownLatch count = new CountDownLatch(10);
    for (int i = 0; i < 10; i++) {
        //开启10个工作线程，执行完任务后调用countdown方法
        new Thread(new Printer(count), "work_thread-" + i).start();
    }
    new Thread(() -> {
        try {
             //阻塞线程，等待结束信号变为0时进行工作
            count.await();
            System.out.println(Thread.currentThread().getName() + ":-- all ready");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }, "personal-thread").start();
    //阻塞线程，等待结束信号变为0时进行工作
    count.await();
    System.out.println(Thread.currentThread().getName() + ":所有线程准备开始!");
    Thread.sleep(2000);
}
```

# CyclicBarrier
CyclicBarrier可以实现类似栅栏的效果：让一组线程共同等待，触发栅栏预设的值parties后，同时执行。
CyclicBarrier内部使用ReentrantLock保证线程之间的同步修改栅栏值parties。
```java
/**
  * parties是指定的栅栏值，就是必须调用await()方法的线程的数目
  * barrierAction是触发了parties阈值后会执行的任务(由最后到达的线程执行)  
  */
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```
下面是CyclicBarrier中await源码
```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //一个分代的屏障标示，内部只有一个broken字段，如果为true表示屏障被突破
        //不同代的CyclicBarrier的Gereration对象不同(每次都会重新new)
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();
        //如果当前线程已被中断，破坏屏障，唤醒所有await的线程，当前线程抛出InterruptedException
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        //需要等待进入屏障的线程数量-1
        int index = --count;
        if (index == 0) {  // tripped，减少到0，达到阈值
            boolean ranAction = false;
            try {
                //如果构造时有指定触发的任务，执行
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                //触发所有await的线程，然后重置CyclicBarrier(可复用)
                nextGeneration();
                return 0;
            } finally {
                //如果ranAction异常的没有为true，突破屏障
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed) //如果没有设置超时时间，直接await
                    trip.await();
                else if (nanos > 0L)    //等待至超时时间
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {    //等于当前代，且屏障没有被破坏，破坏屏障
                    breakBarrier();
                    throw ie;
                } else {    //不等于当前代，或者屏障已经被破坏
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt(); //中断当前线程
                }
            }
            //屏障被破坏，抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
            //已经不是当前代了，返回索引
            if (g != generation)
                return index;
            //设置超时且超时了，破坏屏障，抛出超时异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}
```
CyclicBarrier使用示例:
```java
public static void main(String[] args) {
    CyclicBarrier cyclicBarrier = new CyclicBarrier(2);
    ExecutorService executorService = Executors.newFixedThreadPool(2);
    //任务A
    executorService.submit(() -> {
        try {
            System.out.println("Task A step 1");
            cyclicBarrier.await();
            System.out.println("Task A step 2");
            cyclicBarrier.await();
            System.out.println("Task A step 3");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    });
    //任务B
    executorService.submit(() -> {
        try {
            System.out.println("Task B step 1");
            cyclicBarrier.await();
            System.out.println("Task B step 2");
            cyclicBarrier.await();
            System.out.println("Task B step 3");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    });
    executorService.shutdown();
}   
```
输出:
```text
Task A step 1
Task B step 1
Task B step 2
Task A step 2
Task A step 3
Task B step 3
```

# Semaphore
Semaphore的意思是信号量，我们可以用Semaphore实现对特定资源的允许线程同时访问数量的控制。

Semaphore类似于令牌，信号量有限，信号量被获取光了，只能等待其他线程释放信号量。Semaphore有两个关键的方法`acquire`获取信号量许可，`release`释放信号量许可。其内部还是使用的基于AQS实现的同步控制。

```java
public static void main(String[] args) {
    ExecutorService executorService = Executors.newFixedThreadPool(5);
    final Semaphore semaphore = new Semaphore(2);
    for (int i = 0; i < 5; i++) {
        executorService.submit(() -> {
            try {
                semaphore.acquire();
                System.out.println(Thread.currentThread().getName() + " acquire permit");
                System.out.println(Thread.currentThread().getName() + ":permit remains " + semaphore.availablePermits());
                Thread.sleep(1000);
                semaphore.release();
                System.out.println(Thread.currentThread().getName() + " release permit");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
    executorService.shutdown();
}
```
控制台输出:
```text
pool-1-thread-2 acquire permit
pool-1-thread-2:permit remains 0        //这里是pool-1-thread-2和pool-1-thread-1并发了
pool-1-thread-1 acquire permit
pool-1-thread-1:permit remains 0
pool-1-thread-2 release permit
pool-1-thread-1 release permit          //释放完，此时还有两个信号量，
pool-1-thread-3 acquire permit
pool-1-thread-3:permit remains 1
pool-1-thread-4 acquire permit
pool-1-thread-4:permit remains 0        //线程3，4获取完信号量
pool-1-thread-3 release permit
pool-1-thread-5 acquire permit          //线程5获取到刚刚线程3释放的信号量
pool-1-thread-4 release permit          //线程4释放信号量
pool-1-thread-5:permit remains 1        //此时还有一个线程4释放的信号量
pool-1-thread-5 release permit
```

# 限流的三种常见算法
常见的三种限流算法:
+ 令牌桶限流：令牌痛容量固定，按照固定速度往里面添加令牌(满了就丢弃),如果令牌桶重令牌数量充足，请求可以被处理，否则拒绝新请求。
+ 漏斗限流：漏斗按固定速度流出，当流入的请求数量达到漏斗的上限时，请求被拒绝。
+ 计数器限流：有时我们也可以用计数器来控制一定时间内的总并发数。