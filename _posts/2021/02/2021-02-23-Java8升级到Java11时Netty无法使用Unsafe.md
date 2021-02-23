---
layout: blog
title: "JDK从8升级时Netty报错"
catalog: true
tag: [Java,2021]
---

## 使用Netty时，JDK从8升级到11  时出现报错(debug时可见，不影响运行)

在某个JDK11项目中，debug运行时提示错误信息，错误堆栈信息如下。
增加 `--add-opens java.base/jdk.internal.misc=ALL-UNNAMED` 和 `-Dio.netty.tryReflectionSetAccessible=true` 参数即可。如果要屏蔽掉警告，可以再加上`--illegal-access=warn`参数。

```text
2021-02-23 15:45:48.948 [DEBUG] [io.netty.util.internal.PlatformDependent0.<clinit>] -----direct buffer constructor: unavailable
java.lang.UnsupportedOperationException: Reflective setAccessible(true) disabled
	at io.netty.util.internal.ReflectionUtil.trySetAccessible(ReflectionUtil.java:31) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.internal.PlatformDependent0$4.run(PlatformDependent0.java:238) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at java.base/java.security.AccessController.doPrivileged(Native Method) ~[na:na]
	at io.netty.util.internal.PlatformDependent0.<clinit>(PlatformDependent0.java:232) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.internal.PlatformDependent.isAndroid(PlatformDependent.java:289) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.internal.PlatformDependent.<clinit>(PlatformDependent.java:92) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.ConstantPool.<init>(ConstantPool.java:32) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.channel.ChannelOption$1.<init>(ChannelOption.java:36) ~[netty-transport-4.1.53.Final.jar:4.1.53.Final]
	at io.netty.channel.ChannelOption.<clinit>(ChannelOption.java:36) ~[netty-transport-4.1.53.Final.jar:4.1.53.Final]
	at reactor.netty.tcp.TcpServerBind.<init>(TcpServerBind.java:42) ~[reactor-netty-core-1.0.0.jar:1.0.0]
	at reactor.netty.tcp.TcpServerBind.<clinit>(TcpServerBind.java:36) ~[reactor-netty-core-1.0.0.jar:1.0.0]
	at reactor.netty.tcp.TcpServer.create(TcpServer.java:73) ~[reactor-netty-core-1.0.0.jar:1.0.0]
	at io.rsocket.transport.netty.server.TcpServerTransport.create(TcpServerTransport.java:64) ~[rsocket-transport-netty-1.1.0.jar:na]
	at com.xzy.demo.rsocket.tcp.RequestChannelExample.main(RequestChannelExample.java:41) ~[classes/:na]
2021-02-23 15:45:48.949 [DEBUG] [io.netty.util.internal.PlatformDependent0.<clinit>] -----java.nio.Bits.unaligned: available, true
2021-02-23 15:45:48.952 [DEBUG] [io.netty.util.internal.PlatformDependent0.<clinit>] -----jdk.internal.misc.Unsafe.allocateUninitializedArray(int): unavailable
java.lang.IllegalAccessException: class io.netty.util.internal.PlatformDependent0$6 cannot access class jdk.internal.misc.Unsafe (in module java.base) because module java.base does not export jdk.internal.misc to unnamed module @443118b0
	at java.base/jdk.internal.reflect.Reflection.newIllegalAccessException(Reflection.java:361) ~[na:na]
	at java.base/java.lang.reflect.AccessibleObject.checkAccess(AccessibleObject.java:591) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:558) ~[na:na]
	at io.netty.util.internal.PlatformDependent0$6.run(PlatformDependent0.java:352) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at java.base/java.security.AccessController.doPrivileged(Native Method) ~[na:na]
	at io.netty.util.internal.PlatformDependent0.<clinit>(PlatformDependent0.java:343) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.internal.PlatformDependent.isAndroid(PlatformDependent.java:289) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.internal.PlatformDependent.<clinit>(PlatformDependent.java:92) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.util.ConstantPool.<init>(ConstantPool.java:32) ~[netty-common-4.1.52.Final.jar:4.1.52.Final]
	at io.netty.channel.ChannelOption$1.<init>(ChannelOption.java:36) ~[netty-transport-4.1.53.Final.jar:4.1.53.Final]
	at io.netty.channel.ChannelOption.<clinit>(ChannelOption.java:36) ~[netty-transport-4.1.53.Final.jar:4.1.53.Final]
	at reactor.netty.tcp.TcpServerBind.<init>(TcpServerBind.java:42) ~[reactor-netty-core-1.0.0.jar:1.0.0]
	at reactor.netty.tcp.TcpServerBind.<clinit>(TcpServerBind.java:36) ~[reactor-netty-core-1.0.0.jar:1.0.0]
	at reactor.netty.tcp.TcpServer.create(TcpServer.java:73) ~[reactor-netty-core-1.0.0.jar:1.0.0]
	at io.rsocket.transport.netty.server.TcpServerTransport.create(TcpServerTransport.java:64) ~[rsocket-transport-netty-1.1.0.jar:na]
	at com.xzy.demo.rsocket.tcp.RequestChannelExample.main(RequestChannelExample.java:41) ~[classes/:na]
```

## 原因分析

#### 第一个报错 

```text
direct buffer constructor: unavailable
java.lang.UnsupportedOperationException: Reflective setAccessible(true) disabled
```
在JDK9中，引入了Jigsaw模块化的概念，出于对安全的考虑，对于反射访问进行了限制。只有在
反射操作的模块和制定的包对反射调用者模块Open时，才能调用setAccessible方法。

参考官方文档地址 : [java.lang.reflect.AccessibleObject#setAccessible(boolean)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/reflect/AccessibleObject.html#setAccessible(boolean))


#### 第二个报错

```text
java.lang.IllegalAccessException: class io.netty.util.internal.PlatformDependent0$6 cannot access class jdk.internal.misc.Unsafe (in module java.base) because module java.base does not export jdk.internal.misc to unnamed module @443118b0
```
这个报错原因和第一个报错原因类似，`io.netty.util.internal.PlatformDependent0$6`这个类引用了`jdk.internal.misc.Unsafe`类。

在jdk9中，`Unsafe`类被从`sun.misc.Unsafe`移动到`jdk.unsupported`中了，`jdk.unsupported`模块下的类表示在未来的Java版本中，该模块中的类将被其他 API 所替换。目前依然保留这些类，主要是为了向前兼容(比如旧版本的Netty、Akka等流行的类库就大量使用了Unsafe)。

JDK又推出了一个`jdk.internal.misc.Unsafe`类，但是这个`jdk.internal.misc.Unsafe`类是不开放给开发者的，从路径命名方面来看，`jdk.unsupported`模块下的`Unsafe`应该是为了向前兼容，而`jdk.internal.misc.Unsafe`才是以后jdk内部主要使用的工具类。


## 参考
+ [Netty- cannot access class jdk.internal.misc.Unsafe](https://stackoverflow.com/questions/57885828/netty-cannot-access-class-jdk-internal-misc-unsafe)
+ [浅析 Unsafe 的使用](https://segmentfault.com/a/1190000018037554)