---
layout: blog
title: "Spring容器中的init和destroy方法"
catalog: true
tag: [Java,Spring,2019]
---
# Spring容器中的init和destroy方法
Spring Bean的生命周期大致如下:
![Spring Bean-生命周期](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/spring-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.jpg)
Spring提供了定义好的Spring Bean生命周期内的`init`和`destroy`方法，实现`InitializingBean`和`DisposableBean`方法即可。同时，Spring也支持自定义的初始化和销毁方法。

基于xml配置的方式，在配置bean的时候，我们可以指定`init-method`和`destroy-method`方法。在完成`org.springframework.beans.factory.InitializingBean#afterPropertiesSet`方法调用后，会调用配置的`init-method`方法。同理,在容器关闭的时候，会调用配置的`destroy-method`方法。

基于注解的方法，Spring支持JSR-250规范(为Java SE和Java EE平台中的通用语义概念开发注释,核心还是out-of-the-box开箱即用,主要是指`javax.annotation`包下的几个注解)。其中包括`javax.annotation.PostConstruct`、`javax.annotation.PreDestroy`和`javax.annotation.Resource`。在Spring生命周期中，`PostConstruct`和`PreDestroy`可以标记Spring bean的初始化/销毁动作。其实现类是`org.springframework.context.annotation.CommonAnnotationBeanPostProcessor`。

三种不同的指定定初始化/销毁动作的方式：
+ 实现`InitializingBean`/`DisposableBean`接口，在`afterPropertiesSet`/`destroy`方法中定义初始化/销毁动作。
+ 通过xml中`<bean />`定义的`init-method`和`destroy-method`方法，指定初始化/销毁动作。
+ 通过`@PostConstruct`/`@PreDestroy`注解指定初始化/销毁动作。

根据Spring Bean生命周期的示意图我们可以知道，如果同时配置了三种方式，那么:
>`@PostConstruct`>`InitializingBean#afterPropertiesSet`>`init-method`>`@PreDestroy`>`DisposableBean#destroy`>`destroy-method`

## `InitializingBean`/`DisposableBean`接口 和 `init-method`/`destroy-method`方法
基于接口形式的定制的初始化/销毁动作。由`BeanFactory`调用，可以参考官方源码上的javadoc。
```java
public interface InitializingBean {

	/**
	 * Invoked by a BeanFactory after it has set all bean properties supplied
	 * (and satisfied BeanFactoryAware and ApplicationContextAware).
	 * <p>This method allows the bean instance to perform initialization only
	 * possible when all bean properties have been set and to throw an
	 * exception in the event of misconfiguration.
	 * @throws Exception in the event of misconfiguration (such
	 * as failure to set an essential property) or if initialization fails.
	 */
	void afterPropertiesSet() throws Exception;
}
```
Spring默认使用的BeanFactory`DefaultListableBeanFactory`,其继承自`AbstractAutowireCapableBeanFactory`，`AbstractAutowireCapableBeanFactory`主要负责Bean的实例化,属性注入，调用初始化方法(包括`BeanNameAware`/`BeanClassLoaderAware`/`BeanFactoryAware`/`BeanPostProcessor`/`InitializingBean`等方法)等。

`AbstractAutowireCapableBeanFactory#invokeInitMethods`
```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
        throws Throwable {
    //判断是否实现了 InitializingBean 接口
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isDebugEnabled()) {
            logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {  //如果启用了启用Java安全管理器
            try {
                AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                    @Override
                    public Object run() throws Exception {
                        //调用afterPropertiesSet方法
                        ((InitializingBean) bean).afterPropertiesSet();
                        return null;
                    }
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            //调用afterPropertiesSet方法
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null) {
        String initMethodName = mbd.getInitMethodName();
        //是否指定自定义的`init-method`方法
        if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            //如果指定了`init-method`方法，反射调用指定的`init-method`方法
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

## 使用支持`@PostConstruct`和`@PreDestroy`注解的BeanPostProcessor
基于xml配置的话，如果配置了`<context:annotation-config />`，会隐式地向Spring容器注册`CommonAnnotationBeanPostProcessor`,`AutowiredAnnotationBeanPostProcessor`,`RequiredAnnotationBeanPostProcessor`等注解。下面是官方xml的schema描述:
```xsd
<xsd:element name="annotation-config">
    <xsd:annotation>
        <xsd:documentation><![CDATA[
Activates various annotations to be detected in bean classes: Spring's @Required and
@Autowired, as well as JSR 250's @PostConstruct, @PreDestroy and @Resource (if available),
JAX-WS's @WebServiceRef (if available), EJB3's @EJB (if available), and JPA's
@PersistenceContext and @PersistenceUnit (if available). Alternatively, you may
choose to activate the individual BeanPostProcessors for those annotations.

Note: This tag does not activate processing of Spring's @Transactional or EJB3's
@TransactionAttribute annotation. Consider the use of the <tx:annotation-driven>
tag for that purpose.
        ]]></xsd:documentation>
    </xsd:annotation>
</xsd:element>
```
值得一提的是，如果配置了`<context:component-scan />`的话，就可以不用配置`<context:annotation-config />`了，因为`component-scan`内部包含了`annotation-config`的功能。
```xsd
<xsd:element name="component-scan">
    <xsd:annotation>
        <xsd:documentation><![CDATA[
Scans the classpath for annotated components that will be auto-registered as 
Spring beans. By default, the Spring-provided @Component, @Repository, 
@Service, and @Controller stereotypes will be detected.

Note: This tag implies the effects of the 'annotation-config' tag, activating @Required,
@Autowired, @PostConstruct, @PreDestroy, @Resource, @PersistenceContext and @PersistenceUnit
annotations in the component classes, which is usually desired for autodetected components
(without external configuration). Turn off the 'annotation-config' attribute to deactivate
this default behavior, for example in order to use custom BeanPostProcessor definitions
for handling those annotations.

Note: You may use placeholders in package paths, but only resolved against system
properties (analogous to resource paths). A component scan results in new bean definition
being registered; Spring's PropertyPlaceholderConfigurer will apply to those bean
definitions just like to regular bean definitions, but it won't apply to the component
scan settings themselves.
        ]]></xsd:documentation>
```

基于注解的形式，可以参考`AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry, Object)`方法，`CommonAnnotationBeanPostProcessor`通过这个此方法被注册到spring容器中


## 容易踩坑的地方
`@PostConstruct`和`@PreDestroy`注解属于Java EE，而从JDK 9开始，Java EE被`@Deprecated`，并在JDK 11中被正式移除。使用这些注解，必须另外进行引入相关依赖。
```xml
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
```