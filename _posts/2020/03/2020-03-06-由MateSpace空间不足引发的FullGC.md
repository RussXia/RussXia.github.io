---
layout: blog
title: "由MateSpace空间不足引发的FullGC"
catalog: true
tag: [JVM,2020,踩坑,Java]
---

# 问题背景
> 测试环境和预发环境：JDK版本均为1.8，使用的GC算法均为CMS，但是具体的JVM参数有所差异，预发的JVM配置更小(环境问题搞死人呀，还是容器大法好)。

开发一个的一个功能，普通的业务功能，只是多了文件流的处理，但是量级也不算太大，在测试环境没有任何异常。

部署到预发环境，开始比较正常，但是系统出现504，业务日志出现MQ心跳检查超时，以及MateSpace OOM的异常。检查gc日志，发现频繁发生FullGC。基本可以断定，MQ心跳检查超时、接口504等异常，是由于FullGC的原因造成的。

![业务日志异常信息](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/mq-health-check-timeout.png)

![FullGC日志](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/gc-log-fullgc.png)

![jstat信息](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/jstat-info.png)


# 问题排查

首先，公司的预发环境，并不是互相隔离的，而是一台机器上部署了很多，单个应用配置的JVM最大可用内存是1G，而这期需要发布的项目，是一个巨无霸项目，杂糅了很多业务。稍微top看了下cpu的使用情况，发现并没有什么异常(忘记截图了)，基本可以断定，并不是受cpu资源等因素影响，而是单个JVM出现了问题。

分析gc log，发现FullGC前后，老年代其实并没有达到触发GC的阈值，而造成FullGC的GC Cause是:Metadata GC Threshold。查看应用启动的JVM参数，设置的MateSpace最大大小为128M。GC前后MateSpace空间大小也是从 "124340K->124340K"，124340/1024≈121.42578125mb。

至此基本可以确定，matespace空间不足，达到了设置的上限，因此触发了频繁的FullGC。


# 问题解决

确定了原因是由于Matespace的空间不足，解决的方法也很简单，调整Matespace空间的上限即可。

" -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m"，通过JVM参数调整，设置Metaspace的最大上限为256m。

改变参数后，持续观察，发现已经GC正常。

# 原因思考

在解决问题的过程中，查询了一些资料，并由这个问题，联想到了一些其他问题，特别在此记录一下/

1. 什么是Matespace，它使用的是什么内存，Metadata GC Threshold的含义是什么
2. Matespace和Perm的区别是什么
3. 堆外内存的GC

#### 什么是Matespace，它使用的是什么内存，Metadata GC Threshold的含义是什么

Mateapce：元空间，是JDK1.8中用来代替Perm的。它使用的是本地堆内存(native heap)，所以Matespace并不受JVM可使用内存大小限制。可以使用" **-XX:MaxMetaspaceSize** "参数指定Matespace最大可使用的空间，**-XX:MaxMetaspaceSize**默认是没有限制的。

Matespace主要由Class Metaspace和Non-Class MetaSpace组成。

+ Class Matespace:主要包括byte code、class等信息。如果开启了压缩指针 `-XX:+UseCompressedClassPointers` (默认开启)，这一块的数据会被放到"**Compressed Class Space**" 中，可以通过 `-XX:CompressedClassSpaceSize` 参数来控制大小(默认1G，最大3G)。
+ Non-Class Metaspace：专门来存klass相关的其他的内容，比如method，constantPool等

***

参考资料: 

+ [StackOverFlow:Is CompressedClassSpaceSize area contains MaxMetaspaceSize area?](https://stackoverflow.com/questions/54250638/is-compressedclassspacesize-area-contains-maxmetaspacesize-area)

+ [blog:JVM源码分析之Metaspace解密](http://lovestblog.cn/blog/2016/10/29/metaspace/)
+ [blog:深入理解堆外内存 Metaspace](https://www.javadoop.com/post/metaspace)

`Metadata GC Threshold` 顾名思义，就是 Metaspace 的空间大小超过了这个阈值，尝试FullGC收集可以卸载的类加载器来复用空间，如果空间仍然不足，则尝试对Metaspace进行扩容。

如果频繁发生原因是 `Metadata GC Threshold` 的FullGC ，那么需要做如下排查：

1. MaxMetaspaceSize 设定的是否过小
2. 使用的类加载，是否存在内存泄漏的情况

类加载器负责分配Metaspace的空间，当一个类加载器被卸载后，且发生GC时，这个类加载器加载的类所占用的Metaspace空间，将会被释放。释放的Metaspace空间并不会归还给系统内存，而是会被 JVM 保留下来。

#### Matespace和Perm的区别是什么

Perm主要存放元数据信息(metadata)和常量池，Hotspot的Perm对应的就是JVM规范中的方法区(Method Area)的具体实现(方法区是规范，而永久代是Hotspot针对这个规范的具体实现)。

永久代和堆内存(Heap)是物理内存上连续的，同时，永久代和堆是相互隔离的(永久代的空间属于非堆内存(Non-Heap memory)。为了垃圾回收方便，HotSpot 在永久代上一直是使用老年代的垃圾回收算法。

![image-20200309151232068](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/method-area-heap.png)

HotSpot在JDK1.7中，符号引用(Symbols) ==> Non-Heap Memory；字符串常量池(interned strings) ==> Heap Memory；类的静态变量(class statics) ==> Heap Memory。

JDK1.8中，HotSpot彻底取消了Perm，取而代之的就是Matespace。PermGen中，永久代的大小受JVM最大内存限制(`-Xmx`)，MetaSpace直接使用Non-Heap内存，只受 `-XX:MaxMetaspaceSize` 参数限制。

![image-20200309150738179](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/perm-method-area.png)

#### 堆外内存的GC

堆外内存是指除了`-Xmx` 设置的java堆外的，Java进程所使用的其他内存。主要包括:DirectByteBuffer分配的native memory；线程栈分配的系统内存；java 8里还包括metaspace元数据空间等等。

![644464997-5d809dd65306b_articlex](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/jvm-heam-non-heap.png)

DirectByteBuffer是通过Unsafe.allocateMemory(long size)来分配空间，通过Cleaner来实现堆外内存的释放。如果启动时`-DisableExplicitGC`禁止了System.gc()，可能会出现OOM的风险。

Cleaner : `public class Cleaner extends PhantomReference<Object>`，当GC检查到`Cleaner`的引用变成虚引用可达时，reference-handler线程会调用Cleaner的clean方法回收内存。但是当DirectByteBuffer对象进入老年代后失效了，由于DirectByteBuffer可能很小，所以可能一直没有触发FullGC，因而空间一直没有释放。所以建议调用Cleaner.clean手动释放内存(其内部会调用Unsafe.freeMemory(long size))。

而Unsafe是完全自己手动管理的，如:Unsafe.allocateMemory(long size),Unsafe.freeMemory(long size)方法来手动实现管理内存。

**堆外内存的优缺点**：

+ 堆外内存能减少IO时的内存复制，不需要堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。
+ 堆外内存难以控制，如果内存泄漏，比较难排查 (可以使用NMT排查JVM原生内存使用:Native Memory Tracking)

***

参考资料:

+ [blog：Java堆外内存增长问题排查Case](https://coldwalker.com/2018/08//troubleshooter_native_memory_increase/)
+ [blog：为什么不推荐使用-XX:+DisableExplicitGC](https://ezlippi.com/blog/2017/10/why-not-expliclitgc.html)
+ [blog：Netty之Java堆外内存扫盲贴](http://calvin1978.blogcn.com/articles/directbytebuffer.html)