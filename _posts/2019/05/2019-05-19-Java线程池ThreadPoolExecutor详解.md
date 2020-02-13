---
layout: blog
title: "Java线程池ThreadPoolExecutor详解"
catalog: true
tag: [Java,Java基础,2019]
---
# 为什么要使用线程池
线程是系统资源，多线程技术主要是为了合理利用cpu的并行处理能力(cpu的快速切换以及多核心)，但是创建和销毁线程的开销还是比较耗费时间的，如果系统频繁地创建和销毁线程，那么会造成资源的大量浪费(线程也是一种计算机资源)。

线程池技术就是为了减少线程的创建和销毁，降低资源消耗，提高响应速度。而且线程池的引入，也为管理线程提供了入口，我们可以为线程池指定上限(当然也有无界的线程池,其实无界的就是maxSize为`Integer.MAX_VALUE`)，而不是无限的创建新线程。

# 构建线程池
在构建线程池之前，我们需要了解几个概念,Worker和workQueue。

Worker是线程的抽象，是用于控制和中断执行task的线程。Worker继承自AQS，以实现在task执行过程中获取和释放锁。Worker在创建时,需要传入一个`firstTask`,Worker会先执行传入的task，之后会从任务队列中取。
```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
{
    /**
        * This class will never be serialized, but we provide a
        * serialVersionUID to suppress a javac warning.
        */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
      * Creates with given first task and thread from ThreadFactory.
      * @param firstTask the first task (null if none)
      */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

}
```
线程池用一个 HashSet 管理这些线程。
```java
/**
  * Set containing all worker threads in pool. Accessed only when
  * holding mainLock.
  */
private final HashSet<Worker> workers = new HashSet<Worker>();
```

workQueue,一个阻塞队列，用来存放等待执行任务，主要是为了线程池做缓冲。阻塞队列相较于普通队列来说，支持两个附加操作：1.在队列为空时，获取元素的线程会阻塞等待队列变为非空;2.当队列满时，存储元素的线程会等待队列可用。
```java
/**
    * The queue used for holding tasks and handing off to worker
    * threads.  We do not require that workQueue.poll() returning
    * null necessarily means that workQueue.isEmpty(), so rely
    * solely on isEmpty to see if the queue is empty (which we must
    * do for example when deciding whether to transition from
    * SHUTDOWN to TIDYING).  This accommodates special-purpose
    * queues such as DelayQueues for which poll() is allowed to
    * return null even if it may later return non-null when delays
    * expire.
    */
private final BlockingQueue<Runnable> workQueue;
```

## 线程池的工作流程
从整体来看，线程池的整个工作过程大体分为以下几步：
+ 初始化线程池，指定线程池的大小。
+ * 向线程池中放入需要执行的任务
+ * 如果未达到线程池的核心线程数(corePoolSize)，创建worker，放入线程池集合并执行任务。worker执行完任务后，会一直执行完了后该线程会一直监听工作队列workQueue
+ * 如果线程池达到corePoolSize，但是workQueue未满，将任务放入workQueue
+ * 如果workQueue已满，判断线程池的线程数是否达到线程池上限(maximumPoolSize),如果没有，创建worker,执行任务
+ * 如果线程池已经达到上限(maximumPoolSize)，拒绝任务。
+ 关闭线程池
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
        * Proceed in 3 steps:
        *
        * 1. If fewer than corePoolSize threads are running, try to
        * start a new thread with the given command as its first
        * task.  The call to addWorker atomically checks runState and
        * workerCount, and so prevents false alarms that would add
        * threads when it shouldn't, by returning false.
        *
        * 2. If a task can be successfully queued, then we still need
        * to double-check whether we should have added a thread
        * (because existing ones died since last checking) or that
        * the pool shut down since entry into this method. So we
        * recheck state and if necessary roll back the enqueuing if
        * stopped, or start a new thread if there are none.
        *
        * 3. If we cannot queue task, then we try to add a new
        * thread.  If it fails, we know we are shut down or saturated
        * and so reject the task.
        */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

![线程池处理流程](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/java_thread_pool_process.png)

除此之外，如果线程池创建了超出corePoolSize大小的线程，且线程空闲了一定时间(keepAliveTime),线程池会回收该线程。一般来说，线程池会维护corePoolSize个核心线程存活。如果核心线程被设置为允许超时退出(allowCoreThreadTimeOut)，那么超时后，核心线程也会被回收。

## 用ThreadPoolExecutor构建线程池的几个参数
```java
public ThreadPoolExecutor(int corePoolSize,
                        int maximumPoolSize,
                        long keepAliveTime,
                        TimeUnit unit,
                        BlockingQueue<Runnable> workQueue,
                        ThreadFactory threadFactory,
                        RejectedExecutionHandler handler) {}
```
+ corePoolSize：核心线程数量，如果运行的线程少于corePoolSize，创建新的线程(addWorker,把这个任务作为firstTask)来处理任务。即使线程池中的其他线程是空闲的。
+ maximumPoolSize：线程池能支持的最大线程数，大于corePoolSize小于等于maximumPoolSize的线程，可以被回收。如果超出了maximumPoolSize，会触发RejectedExecutionHandler。
+ keepAliveTime和unit：线程池中空闲线程允许存活的时间，如果超出了corePoolSize，当达到keepAliveTime上限时，那部分线程就会被回收。
+ workQueue:保存等待执行的任务的阻塞队列，如果线程池达到了corePoolSize，那么任务会先放到workQueue中，直到workQueue满了，才会去尝试在不达到maximumPoolSize的前提下新建线程。
+ threadFactory:帮助executor创建线程
+ rejectedExecutionHandler:如果阻塞队列已满，线程数达到了maximumPoolSize上限且没有空闲的线程，触发拒绝策略。

```java
/**
  * Returns true if this pool allows core threads to time out and
  * terminate if no tasks arrive within the keepAlive time, being
  * replaced if needed when new tasks arrive. When true, the same
  * keep-alive policy applying to non-core threads applies also to
  * core threads. When false (the default), core threads are never
  * terminated due to lack of incoming tasks.
  *
  * @return {@code true} if core threads are allowed to time out,
  *         else {@code false}
  *
  * @since 1.6
  */
public boolean allowsCoreThreadTimeOut() {
    return allowCoreThreadTimeOut;
}
```

+ allowCoreThreadTimeOut：如果返回true，表明当前线程是允许corePoolSize内的线程在空闲了keepAlive时间后退出。(核心线程和非核心线程策略一样了)

## 常见的几种阻塞队列
在线程池中，我们常用到的阻塞队列，主要有以下几个:
### SynchronousQueue
这是一个很特别的阻塞队列，他没有容量的概念，内部也没有数据缓存空间。对于每个put/offer操作，等必需等待一个take/poll操作。(生产者和消费者相互等待，<B>组队</B>后put/take结束)

SynchronousQueue支持两种竞争机制:公平和非公平两种
非公平模式下，生成者消费者后进先出栈(LIFO stack)，公平模式先进先出队列(FIFO queue)。代码内部是采用双向链表实现的栈和队列。
### LinkedBlockingQueue
基于单向链表实现的FIFO队列，默认缺省大小是`Integer#MAX_VALUE`，构造LinkedBlockingQueue时可以指定其容量。

LinkedBlockingQueue对入队和出队操作采用了不同的锁，所以LinkedBlockingQueue的入队和出队操作可以并发进行。LinkedBlockingQueue内部采用的是可重入独占的非公平锁。
```java
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

### ArrayBlockingQueue
ArrayBlockingQueue是一个用数组实现的有界阻塞队列。内部按照FIFO的原则对元素进行排序。在初始化ArrayBlockingQueue时，必须指定队列长度，且指定长度后无法进行修改。ArrayBlockingQueue只使用了一把可重入锁，所以不可以像LinkedBlockingQueue那样，支持入队和出队并发执行。
```java
    /** 存储队列元素的数组 */
    final Object[] items;

    /** 取下一个元素的index */
    int takeIndex;

    /** 放下一个元素的index */
    int putIndex;

    /** 队列中元素的个数 */
    int count;

    public void put(E e) throws InterruptedException {
        checkNotNull(e);    //非空
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();   //加锁
        try {
            while (count == items.length)   //队列已满，阻塞
                notFull.await();
            enqueue(e); //入队
        } finally {
            lock.unlock();  //解锁
        }
    }
```
### PriorityBlockingQueue
PriorityBlockingQueue是一个有优先级的无界队列，通过compare方法比较元素的优先级，所以存放的元素必须实现Comparable接口。每次出队返回优先级最高的元素。队列内部使用数组来存放队列元素，用数组存放二叉树的堆排序结果。使用一个独占锁控制入队和出队。

由于PriorityBlockingQueue是无界的，但是内部使用数组存放队列元素，所以当数组满了以后，需要对进行扩容。PriorityBlockingQueue的put操作是不会被阻塞的，当 当前数组元素个数>=数组最大容量时，进行扩容。在扩容之前，PriorityBlockingQueue会释放锁，保证take操作不会被阻塞，使用cas，保证多个put操作并发时，只会有一个put操作进行扩容。
```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();    //加锁
    int n, cap;
    Object[] array;
    //如果当前元素个数>=数组容量，扩容
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        //比较元素，放入队列
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        //数组元素个数+1
        size = n + 1;
        //唤醒notEmpty条件队列上的一个元素
        notEmpty.signal();
    } finally {
        lock.unlock();  //释放锁
    }
    return true;
}
```
关于扩容:
```java
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); //扩容前，先释放锁，不阻塞take操作
    Object[] newArray = null;
    //cas保证只会有一个线程进行扩容
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                    0, 1)) {
        try {
            //计算新容量，上限是Integer.MAX_VALUE - 8
            int newCap = oldCap + ((oldCap < 64) ?
                                    (oldCap + 2) : // grow faster if small
                                    (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            //创建新的数组
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }
    //并发时，只会有一个线程能通过cas，其他线程会调用Thread.yield()，让出cpu
    //但是还是有可能在newArray=null(未完成扩容)的情况下，其他的获取了锁(概率比较小而已)
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

### DelayedWorkQueue
DelayedWorkQueue是ScheduledThreadPoolExecutor的一个内部类，是一个无界的阻塞队列。DelayedWorkQueue和PriorityQueue一样，使用数组存放二叉树堆排序。

DelayedWorkQueue使用一个可重入的独立锁控制入队出队。
```java
 public boolean offer(Runnable x) {
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        //扩容
        if (i >= queue.length)
            grow();
        size = i + 1;
        //如果原来队列为空，存放到数组第一个位置
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
            //按二叉树堆的存放顺序存放到数组中
            siftUp(i, e);
        }
        //如果之前队列为空，添加了元素，通知在等待的线程
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
```

```java
 public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            //获取第一个元素，如果为空，则说明队列为空，等待唤醒
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                //获取延迟时间
                long delay = first.getDelay(NANOSECONDS);
                //如果任务不用等待，从堆上拿走返回给线程
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                //如果任务需要等待，且已经有一个线程在等待执行任务了(在等待的肯定先执行)，当前线程等待唤醒
                if (leader != null)
                    available.await();
                else {
                    //设置leader为当前线程，等待delay时间
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        //重置leader
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        //拿到任务，唤醒在等待的线程，释放锁。
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

## 常见的几种拒绝策略
+ AbortPolicy：丢弃任务，抛出 `RejectedExecutionException` 异常。
+ CallerRunsPolicy：用调用者所在的线程执行任务。
+ DiscardOldestPolicy：丢弃阻塞队列最前面的任务，执行当前任务(调用线程池的execute()方法)。
+ DiscardPolicy：直接丢弃任务，不抛异常，也不做什么处理

## Java线程池的结构设计
+ Executor接口:最顶层的接口，提供execute方法，在未来的某个时间执行任务。当一个Runnable的任务被提交给Executor后，如何执行就要看其具体实现类。
```java
public interface Executor {
    void execute(Runnable command);
}
```
+ ExecutorService接口：继承自Executor，并声明了submit、invokeAll、invokeAny以及shutDown等方法。其中submit方法，支持返回task运行结果(传递的task必须是Callable<T> task)。

+ ScheduledExecutorService接口：继承自ExecutorService接口，增加了定时执行的功能。

+ AbstractExecutorService抽象类：ExecutorService接口的默认实现,基本实现了基本实现了ExecutorService中声明的方法(实现了submit和invoke等等方法，execute和shutdown等方法未实现)。

+ ThreadPoolExecutor实现类：实现了AbstractExecutorService抽象类，提供了一个拥有完整的基础功能的线程池。后面提到的Executors工具类方法提供的创建线程池，有一部分就是用法的ThreadPoolExecutor实现的。

+ ScheduledThreadPoolExecutor实现类：继承自ThreadPoolExecutor，实现了ScheduledExecutorService接口，提供拥有调度功能的线程池。

![Java线程池的结构设计](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/java_thread_pool_struct.png)

## Executors提供的创建线程池方法
+ newCachedThreadPool：创建一个可缓存的线程池。线程数量不定，如果有空闲的线程则复用；如果没有，则创建新的线程。超过一定的空闲时间则回收(60s)线程。
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```
+ newFixedThreadPool:创建一个固定大小的线程池，该线程池的核心线程数和最大线程数相等。
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}
```
+ newScheduledThreadPool:返回一个指定了核心线程数的支持定时地周期执行任务的线程池。
```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```
+ newSingleThreadExecutor:创建一个只有一个线程的线程池，他和newFixedThreadPool(1)的区别是，当这个生成的变量不在被引用时(reconfigurable),被gc时会调用FinalizableDelegatedExecutorService的finalize()方法，它在finalize()方法里，调用了持有的线程池对象ExecutorService的shutdown()方法。(所以才会当reconfigurable的时候，会释放之前线程占用的空间。) newSingleThreadExecutor
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
+ newSingleThreadScheduledExecutor:创建一个单例的，定期或延时执行任务的线程池。
```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
    return new DelegatedScheduledExecutorService
        (new ScheduledThreadPoolExecutor(1));
}
```
+ newWorkStealingPool:创建一个线程池:基于work-stealing算法实现的ForkJoinPool。其并发级别就是机器的核心数。
```
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool
        (Runtime.getRuntime().availableProcessors(),
            ForkJoinPool.defaultForkJoinWorkerThreadFactory,
            null, true);
}
```
![Executors提供的创建线程池方法](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/thread_pool_executors.jpg)

# 其他
## 关于Callable、Runnable、Future、FutureTask
+ Runnable:Runnable只有一个无返回值的run方法，执行完任务后无任何返回值
```java
public interface Runnable {
    public abstract void run();
}
```
+ Callable:Callable是jdk1.5引入的J.U.C包下的接口，它只有一个带返回值的call()方法。一般是配合ExecutorService使用，`<T> Future<T> submit(Callable<T> task);`,调用submit方法直接获得一个Future类型的返回值。通过Future的get方法，可以获得任务的执行结果。
```java
public interface Callable<V> {
    V call() throws Exception;
}
```
+ Future:Future提供取消执行Runnable/Callable任务，查询Runnable/Callabler任务状态(是否取消，是否完成等)，获取执行结果(get方法,阻塞获得执行结果)等功能。
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
+ FutureTask:FutureTask是一个类，实现了RunnableFuture接口(继承自Runnable, Future接口)。所以FutureTask可以看作是Callable和Runnable两个的包装类，它既可以接手Runnable类型任务，也可以接受Runnable类型的任务。由于Runnable没有返回值，所以用Runnable构造FutureTask时，要先指定返回值
```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

FutureTask的使用示例:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    //testCallable();
    testRunnable();
}

public static void testCallable() throws ExecutionException, InterruptedException {
    Callable<String> callable = () -> {
        System.out.println("Callable started");
        Thread.sleep(3000L);
        System.out.println("Callable over");
        return "callable result";
    };
    FutureTask<String> futureTask = new FutureTask<>(callable);
    new Thread(futureTask).start();
    while (true) {
        if (!futureTask.isDone()) {
            System.out.println("callable task is done!");
            System.out.println(futureTask.get());
            break;
        }
    }
    System.out.println("main thread over");
}

public static void testRunnable() throws ExecutionException, InterruptedException {
    Runnable runnable = () -> {
        System.out.println("Runnable started");
        try {
            Thread.sleep(3000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Runnable over");
    };
    FutureTask<String> futureTask = new FutureTask<>(runnable, "runnable result");
    new Thread(futureTask).start();
    while (true) {
        if (!futureTask.isDone()) {
            System.out.println("Runnable task is done!");
            System.out.println(futureTask.get());
            break;
        }
    }
    System.out.println("main thread over");
}
```

## 关于线程池的shutDown和finalize
我们知道，在对像被gc之前，会被调用finalize方法。我们可以在finalize方法中去释放一些资源(其实并不推荐这么做，因为finalize方法调用的时间不确定。) 在查看源码的过程中，我发现`java.util.concurrent.Executors#newFixedThreadPool(int nThreads)`方法上有一段注释是这样写的:
```text
If any thread terminates due to a failure during execution prior to shutdown, a new one will take its place if needed to execute subsequent tasks.
```
也就是说，`java.util.concurrent.Executors#newFixedThreadPool(int nThreads)`的线程在执行过程中因为失败而停止了，会新起一个线程代替执行后续任务。

而在`java.util.concurrent.Executors#newSingleThreadExecutor()`中，有一段注释是这样写的:
```text
Note however that if this single thread terminates due to a failure during execution prior to shutdown, a new one will take its place if needed to execute subsequent tasks.
Unlike the otherwise equivalent {@code newFixedThreadPool(1)} the returned executor is guaranteed not to be reconfigurable to use additional threads.
```
`java.util.concurrent.Executors#newSingleThreadExecutor()`在线程执行任务失败时，新的线程会替代它执行后续任务。但是和`newFixedThreadPool(1)`不同的是，在reconfigurable的时候，不能调整它的`core pool size`。
[参见相关stackoverflow问题](https://stackoverflow.com/questions/3911100/any-difference-among-executors-newsinglethreadexecutor-and-executors-newfixedt)

查看源码的时候，发现`java.util.concurrent.Executors#newSingleThreadExecutor()`返回的并不是一个`ThreadPoolExecutor`对象，而是`FinalizableDelegatedExecutorService`，`FinalizableDelegatedExecutorService`的父类`DelegatedExecutorService`基本就是个装饰器模式，调用通过构造函数传入进来的`ExecutorService e`实例的方法。而`FinalizableDelegatedExecutorService`多了一个`finalize`方法。所以说，`java.util.concurrent.Executors#newSingleThreadExecutor()`是无法调整他的`core pool size`。
```java
static class DelegatedExecutorService extends AbstractExecutorService {
    private final ExecutorService e;
    DelegatedExecutorService(ExecutorService executor) { e = executor; }
    public void execute(Runnable command) { e.execute(command); }
    public void shutdown() { e.shutdown(); }
    public List<Runnable> shutdownNow() { return e.shutdownNow(); }
    public boolean isShutdown() { return e.isShutdown(); }
    public boolean isTerminated() { return e.isTerminated(); }
    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        return e.awaitTermination(timeout, unit);
    }
    public Future<?> submit(Runnable task) {
        return e.submit(task);
    }
    public <T> Future<T> submit(Callable<T> task) {
        return e.submit(task);
    }
    public <T> Future<T> submit(Runnable task, T result) {
        return e.submit(task, result);
    }
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        return e.invokeAll(tasks);
    }
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                            long timeout, TimeUnit unit)
        throws InterruptedException {
        return e.invokeAll(tasks, timeout, unit);
    }
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        return e.invokeAny(tasks);
    }
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                            long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return e.invokeAny(tasks, timeout, unit);
    }
}
```
除了继承自装饰器模式的父类`DelegatedExecutorService`,`FinalizableDelegatedExecutorService`对象还复写了finalize方法，并在finalize方法，调用了装饰器ExecutorService的shutdown方法，在发生gc时，如果`FinalizableDelegatedExecutorService`不存在引用，即使不调用shutdown方法，创建的线程是可以被gc掉的。

这个时候，我发现，`java.util.concurrent.ThreadPoolExecutor#finalize`也调用了shutdown方法，这个时候矛盾的地方就来了。

我们知道，finalize方法，是在gc决定回收一个不再引用的对象时被调用，也就是说，`java.util.concurrent.ThreadPoolExecutor#finalize`方法被调用的前提，这个ThreadPoolExecutor不再被引用(gcroot无法关联到)。但是，Thread是作为GCRoot一部分的，所以即使一个`ThreadPoolExecutor`对象不在被引用，只要它内部还有alive的thread，仍然可以通过GCRoot找到，finalize方法就永远不会被调用。

+ [finalize方法参考资料](https://javaconceptoftheday.com/garbage-collection-finalize-method-java/)
+ [GCRoot](https://www.zhihu.com/question/53613423/answer/135743258)

为了验证猜想，以`-Xms128m -Xmx128m -Xss160k -XX:+PrintGCDetails`相同的JVM参数，运行两个demo做对比。
```java
public static void main(String[] args) throws ClassNotFoundException {
    while (true) {
        byte[] dd = new byte[1024 * 1024 * 10];
        AtomicBoolean atomicBoolean = new AtomicBoolean(true) {
            @Override
            protected void finalize() throws Throwable {
                System.out.println("java.util.concurrent.atomic.AtomicBoolean#finalize");
                super.finalize();
            }
        };

        Class<?> finalizable = Class.forName("java.util.concurrent.Executors.FinalizableDelegatedExecutorService");

        ExecutorService single = Executors.newSingleThreadExecutor();

        int i = 0;
        while (i++ <= 10) {
            single.submit(() -> {});
        }
        single = null;
        System.gc();
//            Thread.sleep(1000);
    }
}
```
一直正常运行，不会触发OOM或者stackoverflow，说明gc正常，线程空间得到了释放。debug发现确实调用到finalize方法。部分控制台输出如下:
```
22562
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 12197K->96K(38400K)] 23530K->21669K(125952K), 0.0026554 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 96K->0K(38400K)] [ParOldGen: 21573K->11333K(87552K)] 21669K->11333K(125952K), [Metaspace: 4959K->4959K(1056768K)], 0.0127000 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
22563
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 11753K->128K(38400K)] 23086K->21701K(125952K), 0.0022073 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 128K->0K(38400K)] [ParOldGen: 21573K->11333K(87552K)] 21701K->11333K(125952K), [Metaspace: 4959K->4959K(1056768K)], 0.0109530 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
22564
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 11975K->128K(38400K)] 23309K->21701K(125952K), 0.0024081 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 128K->0K(38400K)] [ParOldGen: 21573K->11331K(87552K)] 21701K->11331K(125952K), [Metaspace: 4959K->4959K(1056768K)], 0.0111041 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
22565
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 11753K->160K(38400K)] 23085K->21731K(125952K), 0.0027836 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 160K->0K(38400K)] [ParOldGen: 21571K->11325K(87552K)] 21731K->11325K(125952K), [Metaspace: 4959K->4959K(1056768K)], 0.0063579 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
22566
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 12640K->128K(38400K)] 23966K->21693K(125952K), 0.0189186 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [PSYoungGen: 128K->0K(38400K)] [ParOldGen: 21565K->11251K(87552K)] 21693K->11251K(125952K), [Metaspace: 4961K->4961K(1056768K)], 0.0372619 secs] [Times: user=0.03 sys=0.00, real=0.04 secs] 
java.util.concurrent.atomic.AtomicBoolean#finalize
```

```java
public static void main(String[] args) {
    int j = 0;
    while (true) {
        System.out.println(j++);
        byte[] dd = new byte[1024 * 1024 * 10];
        AtomicBoolean atomicBoolean = new AtomicBoolean(true) {
            @Override
            protected void finalize() throws Throwable {
                System.out.println("java.util.concurrent.atomic.AtomicBoolean#finalize");
                super.finalize();
            }
        };
        ThreadPoolExecutor threadPoolExecutor =
                new ThreadPoolExecutor(50, 100, 1000L, TimeUnit.MICROSECONDS, new ArrayBlockingQueue<>(1000)) {
                    @Override
                    protected void finalize() {
                        System.out.println("java.util.concurrent.ThreadPoolExecutor.finalize");
                        super.finalize();
                    }
                };
        int i = 0;
        while (i++ <= 10) {
            threadPoolExecutor.submit(() -> {});
        }
//            Thread.sleep(1000);
//            threadPoolExecutor.shutdown();
        System.gc();
    }
}
```
发生oom异常，jvm内存空间不足以创建更多的线程，没用调用ThreadPoolExecutor#finalize方法，控制台也确实没有看到finalize方法的相关输出。部分控制台输出如下:
```text
180
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 12083K->96K(38400K)] 25732K->23984K(125952K), 0.0093283 secs] [Times: user=0.02 sys=0.01, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 96K->0K(38400K)] [ParOldGen: 23888K->13660K(87552K)] 23984K->13660K(125952K), [Metaspace: 4962K->4962K(1056768K)], 0.0349870 secs] [Times: user=0.04 sys=0.00, real=0.04 secs] 
181
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 12032K->96K(38400K)] 25692K->23996K(125952K), 0.0055466 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 96K->0K(38400K)] [ParOldGen: 23900K->13671K(87552K)] 23996K->13671K(125952K), [Metaspace: 4962K->4962K(1056768K)], 0.0364074 secs] [Times: user=0.04 sys=0.00, real=0.03 secs] 
182
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 12135K->96K(38400K)] 25806K->24007K(125952K), 0.0137471 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 96K->0K(38400K)] [ParOldGen: 23911K->13683K(87552K)] 24007K->13683K(125952K), [Metaspace: 4962K->4962K(1056768K)], 0.0661352 secs] [Times: user=0.08 sys=0.00, real=0.07 secs] 
183
java.util.concurrent.atomic.AtomicBoolean#finalize
[GC (System.gc()) [PSYoungGen: 12032K->64K(38400K)] 25715K->23987K(125952K), 0.0094652 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 64K->0K(38400K)] [ParOldGen: 23923K->13694K(87552K)] 23987K->13694K(125952K), [Metaspace: 4962K->4962K(1056768K)], 0.0361095 secs] [Times: user=0.04 sys=0.00, real=0.04 secs] 
184
java.util.concurrent.atomic.AtomicBoolean#finalize
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
	at java.lang.Thread.start0(Native Method)
	at java.lang.Thread.start(Thread.java:717)
	at java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:957)
	at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1367)
	at java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:112)
	at com.xzy.test.LinkedHashSetDemo.main(LinkedHashSetDemo.java:38)
```

因为`java.util.concurrent.Executors#newSingleThreadExecutor()`返回的是一个装饰器，所以在没有引用的时候，`FinalizableDelegatedExecutorService`对象实例是可以被gc的，`FinalizableDelegatedExecutorService`的`finalize`方法调用了被装饰的`ExecutorService#shutdown()`方法，所以导致了线程池可以被gc。而`ThreadPoolExecutor`因为有存活的线程的原因，无法被gc，当然也无法被调用finalize方法。

[stackoverflow上关于finalize方法的解释](https://stackoverflow.com/questions/7728102/why-doesnt-this-thread-pool-get-garbage-collected)
```text
This doesn't really have anything to do with GC being non-deterministic, although it doesn't help! (That is one cause in your example, but even if we 'fixed' it to eat up memory and force a collection, it still wouldn't finalize)

The Worker threads that the executor creates are inner classes that have a reference back to the executor itself. (They need it to be able to see the queue, runstate, etc!) Running threads are not garbage collected, so with each Thread in the pool having that reference, they will keep the executor alive until all threads are dead. If you don't manually do something to stop the threads, they will keep running forever and your JVM will never shut down.
```
总结:
+ 不要依靠对象的finalize方法，因为gc和finalize方法的调用是不可靠的。
+ 线程池在退出之前，一定要手动显示的调用的shutdown()方法。