---
layout: blog
title: "Java中的invokedynamic和lambda表达式"
catalog: true
tag: [Java,JVM,2019]
---
# Java中的invokedynamic和lambda表达式

## 关于invokedynamic和其他四种字节码方法调用指令
在Java7中，JVM新增了invokedynamic指令。在此之前，JVM就已经提供了四种不同的字节码方法调用指令:

+ invokevirtual——对实例方法的标准分派
+ invokestatic——用于分派静态方法
+ invokeinterface——用于通过接口进行方法调用的分派
+ invokespecial——当需要进行非虚（也就是“精确”）分派时会用到，调用实例构造方法（方法），私有方法，父类继承方法。

下面是关于这四种方法分派方式的demo。
```java
public class ByteCodeInstructionDemo implements Comparable {

    public static void main(String[] args) {
        ByteCodeInstructionDemo bs = new ByteCodeInstructionDemo();
        System.out.println(bs.sayHello());
        sayHi();
        Comparable comparable = bs;
        System.out.println(comparable.compareTo(null));
    }

    public String sayHello() {
        return "say Hello";
    }

    public static void sayHi() {
        System.out.println("say Hi");
    }

    @Override
    public int compareTo(Object o) {
        return -1;
    }
}
```

当前的JDK版本:
```shell
java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)
```

生成的部分Java字节码如下:
```shell
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=3, args_size=1
         0: new           #2                  // class com/xzy/demo/ByteCodeInstructionDemo
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        11: aload_1
        12: invokevirtual #5                  // Method sayHello:()Ljava/lang/String;
        15: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        18: invokestatic  #7                  // Method sayHi:()V
        21: aload_1
        22: astore_2
        23: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        26: aload_2
        27: aconst_null
        28: invokeinterface #8,  2            // InterfaceMethod java/lang/Comparable.compareTo:(Ljava/lang/Object;)I
        33: invokevirtual #9                  // Method java/io/PrintStream.println:(I)V
        36: return
```
加入invokedynamic指令的目的是为了在Java平台支持`动态语言类型`([`JSR-292`](https://jcp.org/en/jsr/detail?id=292))。

## lambda和invokedynamic
lambda最后实际的运行其实还是`invokevirtual`或者`invokeinterface`，`invokedynamic`只是推迟到到了运行时。运行时动态生成实现类(由调用`java.lang.invoke.LambdaMetafactory.metafactory`实现)。invokedynamic指令调用`metafactory`方法，会返回一个`CallSite`，此`CallSite`返回目标类型的一个匿名实现类， 此类关联编译时产生的方法。

## 关于类加载顺序
[https://segmentfault.com/n/1330000019799723?from=timeline&isappinstalled=0](https://segmentfault.com/n/1330000019799723?from=timeline&isappinstalled=0)

[https://www.bilibili.com/video/av59801808](https://www.bilibili.com/video/av59801808)


## 参考资料
+ [https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokedynamic](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokedynamic)
+ [https://www.zhihu.com/question/39462935](https://www.zhihu.com/question/39462935)
+ [http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)
+ [https://stackoverflow.com/questions/30002380/why-are-java-8-lambdas-invoked-using-invokedynamic](https://stackoverflow.com/questions/30002380/why-are-java-8-lambdas-invoked-using-invokedynamic)
