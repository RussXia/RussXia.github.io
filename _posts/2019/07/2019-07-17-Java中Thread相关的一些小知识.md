---
layout: blog
title: "Java中Thread相关的一些小知识"
catalog: true
tag: [Java,Java基础,2019]
---

## Java使用进程执行shell命令
```
public static void main(String[] args) throws IOException, InterruptedException {
    String[] cmdline = {"sh", "-c", "echo $MYSQL_HOME"};
    Runtime runtime = Runtime.getRuntime();
    Process process = runtime.exec(cmdline);
    BufferedReader read = new BufferedReader(new InputStreamReader(
            process.getInputStream()));
    try {
        process.waitFor();
    } catch (InterruptedException e) {
        System.out.println(e.getMessage());
    }
    while (read.ready()) {
        System.out.println(read.readLine());
    }
}
```

## Runnable的花式写法
```java
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(ThreadCreationDemo::action, "t1");
    t1.start();
    t1.join();
}

private static void action() {
    System.out.println(Thread.currentThread().getName() + " is doing action!");
}
```

## 关于Thread的join方法
下面是Thread.join()方法的源码，不难看出主要思路就是自旋(?)+wait。当前调用了thread.join()方法的线程(本地demo一般就是main线程)会进行wait操作，等待notify。
```java
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```
那么，是谁，在什么时候notify了正在wait中的主线程呢？
当前线程在退出之前，会调用`Thread.exit()`方法,去做一些clean up。
```java
/**
    * This method is called by the system to give a Thread
    * a chance to clean up before it actually exits.
    */
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```
他会通知当前线程所在的线程组，清除该线程。线程组清除线程的方式，将该线程从线程组中移除(`remove(t)`)，并且当前线程组没有活跃线程的话，唤醒其他等待的线程。
```java
/**
    * Notifies the group that the thread {@code t} has terminated.
    *
    * <p> Destroy the group if all of the following conditions are
    * true: this is a daemon thread group; there are no more alive
    * or unstarted threads in the group; there are no subgroups in
    * this thread group.
    *
    * @param  t
    *         the Thread that has terminated
    */
void threadTerminated(Thread t) {
    synchronized (this) {
        remove(t);

        if (nthreads == 0) {
            notifyAll();
        }
        if (daemon && (nthreads == 0) &&
            (nUnstartedThreads == 0) && (ngroups == 0))
        {
            destroy();
        }
    }
}
```
所以，`Thread.join()`方法，会使主线程(本例中)调用wait方法，并在子线程exit后，通过notify()方法唤醒主线程。