---
layout: blog
title: "Spring IOC 循环依赖"
tag: [Java,Spring]
---
# Spring IOC 循环依赖

## 1.什么是循环引用？

![.循环引用](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/spring-circular-reference1.jpg)
所谓循环依赖，就是在注入Bean到Spring容器的时候，存在两个Bean，它们通过构造函数注入，且a依赖于b，b依赖于a。

对于这种通过构造函数注入的循环依赖，Spring是无法进行实例化的，在实例化的工程中会抛出`org.springframework.beans.factory.BeanCurrentlyInCreationException`,对应给出的异常信息:`Requested bean a is currently in creation: Is there an unresolvable circular reference?`

## 2.Spring是如何检测循环引用的

Spring在创建a这个bean的元素报错，因为调用构造函数的时候无法设置b这个bean。(这个和xml中配置的bean顺序有关)，而创建b这个bean的时候检测出a这个bean可能存在循环依赖。

下面是节选的一些堆栈上的异常信息:

```text
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'a' defined in class path resource [applicationContext.xml]: Cannot resolve reference to bean 'b' while setting constructor argument; nested exception is org.springframework.beans.factory.
BeanCreationException: Error creating bean with name 'b' defined in class path resource [applicationContext.xml]: Cannot resolve reference to bean 'a' while setting constructor argument; 
nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

下面是Spring检测循环引用的相关代码:

```java
/**
 * 解析引用部分
 */
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
   try {
      String refName = ref.getBeanName();
      refName = String.valueOf(doEvaluate(refName));
      //判断该引用是否是位于父一级的BeanFactory中
      if (ref.isToParent()) {
         if (this.beanFactory.getParentBeanFactory() == null) {
            throw new BeanCreationException(
                  this.beanDefinition.getResourceDescription(), this.beanName,
                  "Can't resolve reference to bean '" + refName +
                  "' in parent factory: no parent factory available");
         }
         //如果存在父一级BeanFactory，且不为空，从中获取bean
         return this.beanFactory.getParentBeanFactory().getBean(refName);
      }
      else {
         //不存在父一级BeanFactory的，直接从当前的BeanFactory中获取bean元素
         Object bean = this.beanFactory.getBean(refName);
         this.beanFactory.registerDependentBean(refName, this.beanName);
         return bean;
      }
   }
   catch (BeansException ex) {
      throw new BeanCreationException(
            this.beanDefinition.getResourceDescription(), this.beanName,
            "Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
   }
}
```

从上面的代码以及加载Spring容器时的堆栈错误信息，我们可以笃定是在调用getBean(String refName) 方法时发生的异常。接下来，让我们看看在调用getBean(String refName) 方法时，Spring是怎样检测出错误的。

```java
protected <T> T doGetBean(
      final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
      throws BeansException {

   final String beanName = transformedBeanName(name);
   Object bean;

   // 先从已经加载的单例bean中获取(可能不存在)
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isDebugEnabled()) {
          //如果当前实例在
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
      }

      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      try {
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               registerDependentBean(dep, beanName);
               getBean(dep);
            }
         }

         // Create bean instance.创建bean实例,此处省略
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }
   // 检查创建的bean实例与所要求的是否匹配，此处省略
   return (T) bean;
}
```