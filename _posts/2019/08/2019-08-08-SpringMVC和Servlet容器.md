---
layout: blog
title: "SpringMVC和Servlet"
catalog: true
tag: [Java,Spring,Web,2019]
---
# SpringMVC和Servlet
HTTP服务器将请求发到Servlet容器，Servlet容器会将请求转发到具体的Servlet，如果此Servlet尚未创建，则加载并实例化这个Servlet(init方法)，然后调用这个Servelt的`service`方法，真正处理请求。
![servlet容器](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/servlet-container.jpg)
## Servlet接口
```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```
上面是`Servlet`接口定义的五个方法。
+ `init`和`destory`方法描述了这个`Servlet`创建和销毁时应该做的一些动作。`Servlet`容器在加载`Servlet`容器的时候，会调用`init`方法；在卸载的时候，会调用`destroy`方法。
+ `getServletInfo`方法，主要用于返回此Servlet的一些信息,包括author、version、copyright等等。
+ `getServletConfig`方法，返回`ServletConfig`对象，`ServletConfig`对象包含了`Servlet`的初始化参数。
+ `service`方法，由`Servlet`容器调用，处理请求。方法只可能在servlet成功`init`后才可以被调用。

`Servlet`是网络处理，请求抽象的封装，针对Http协议的实现，对应的`Servlet`实现类就是`javax.servlet.http.HttpServlet`。 `HttpServlet`只处理参数格式是`HttpServletRequest`和`HttpServletResponse`的请求。`HttpServletRequest`和`HttpServletResponse`表示HTTP协议中的请求和响应。`Service`方法，会根据HTTP请求的请求方法，调用`doGet`，`doPost`等方法处理。

每个web应用都会创建一个全局唯一的`ServletContext`对象，你可以通过`Servletontext`来共享`Servlet`数据，`ServletContext`是线程不安全的，Tomcat中的实现是`org.apache.catalina.core.ApplicationContext`。在分布式环境中，由于每个VM中都会有一个`ServletContext`,无法做到全局共享一个context,建议使用db代替。

## filter过滤器 和 listner监听器
filter过滤器，在Web应用部署完成后，Servlet容器会实例化filter，并把filter连接成一个`FilterChain`(链表)。当请求进来时，获取第一个filter，并调用`doFilter`方法
> Filters use the FilterChain to invoke the next filter in the chain, 
> or if the calling filter is the last filter in the chain, 
> to invoke the resource at the end of the chain.

Listener监听器，主要用于监听某些特定的事件，当特定事件发生时，Servlet容器回负责调用监听器的方法。
Servlet的事件监听接口有8个:
+ `ServletContextListener`：用于监听 ServletContext 的启动和销毁。
+ `ServletContextAttributeListener`：用于监听 ServletContext 属性的变化(add/remove/replace)。
+ `HttpSessionListener`：用于监听 session 的创建和销毁。
+ `HttpSessionIdListener`：用于监听 session 的 id 是否被更改。(使用的很少,不清楚什么样的属于)
+ `HttpSessionAttributeListener`：用于监听 session 范围的属性变化(add/remove/replace)。
+ `HttpSessionActivationListener`：用于监听绑定在 HttpSession 对象中的 JavaBean 状态。
+ `HttpSessionBindingListener`：用于监听对象与 session 的绑定和解绑。
+ `ServletRequestListener`：用于监听 ServletRequest 对象的初始化和销毁。
+ `ServletRequestAttributeListener`：用于监听 ServletRequest 对象的属性变化。
![HTTP请求处理流程](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/servlet-filtetr.gif)

## SprinMVC
```xml
<!-- 配置启动参数->spring xml入口，加载spring容器-->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath*:applicationContext.xml</param-value>
</context-param>

<!-- Servlet容器启动和销毁时触发的时间，加载spring容器的入口 -->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<!-- DispatcherServlet:SpringMVC的核心，负责加载SpringMVC容器，请求的处理转发 -->
<servlet>
    <servlet-name>SpringMVC</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<!-- 交给SpringMVC这个Servlet处理的路由请求 -->
<servlet-mapping>
    <servlet-name>SpringMVC</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```
### SpringMVC的启动过程
我们在配置SpringMVC的时候，会配置一个Listener:`org.springframework.web.context.ContextLoaderListener`,这个`ContextLoaderListener`实现了`ServletContextListener`接口，所以当Servlet容器启动的时候，会触发事件，然后调用`org.springframework.web.context.ContextLoaderListener#contextInitialized`方法。

`ContextLoaderListener`类的部分源码:
```java
/**
  * Initialize the root web application context.
  */
@Override
public void contextInitialized(ServletContextEvent event) {
    //创建spring web容器(这是spring ROOT容器)
    initWebApplicationContext(event.getServletContext());
}
```
在创建完成Spring的 web application context 后，会将其保存到`ServletContext`中，方便获取。

Servlet一般是会延迟加载，但是我们配置了`load-on-startup`为1，当`load-on-startup`的value>=0时，容器必须在部署后加载这个Servlet。Tomcat/Jetty发现DispatcherServlet还没有被实例化，就会调用`DispatcherServlet`的init方法。

`DispatcherServlet`会初始化自己的IoC上下文(Spring MVC容器)，并将<B>Spring的`application context`作为自己的父级context</B>，`DispatcherServlet`初始化的，主要是我们本例中定义在`springmvc.xml`中配置的bean(一般都是扫描controller这类bean，各类解析器bean等)以及`org.springframework.web.servlet.DispatcherServlet#initStrategies`初始化的一些策略。如视图解析器，请求和处理的映射(mapping)，异常处理器等等。

## 关于ServletContext,Spring容器,SpringMVC容器的总结
Tomcat在启动时会为Web应用创建一个全局的上下文环境，这个就是ServletContext。启动web应用时，`ContextLoaderListener`会读取配置的xml文件，初始化Spring容器，并将其保存到ServletContext中。然后在初始化`DispatcherServlet`时，会初始化SpringMVC容器，并从ServletContext容器中获取Spring容器，并将Spring容器作为自己的`parent`容器，最后`DispatcherServlet`会将初始化好的SpringMVC容器也保存到ServletContext中。

<B>因为Spring容器和SpringMVC容器是两个容器，Spring容器是SpringMVC容器的parent，所以在SpringMVC容器中，可以获取到Spring容器中的Bean。而在Spring容器中不能获取到SpringMVC中的bean。</B>