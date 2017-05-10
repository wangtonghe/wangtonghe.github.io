---
layout: post
title:  "Servlet工作机制解析"
date:   2017-04-30 08:12:00 +0800
categories: javaweb servlet
header-img: img/posts/servlet/servlet-cover.jpg
tags:
 - javaweb
 - servlet
 - java
---

# Servlet工作机制解析

Servlet是java Web技术的基础，也是学习Web 框架原理绕不过去的部分。本章我们来学习学习。

## 1. Servlet和Servlet容器

什么时servlet?从概念上来说是这样的：

> Servlet是用java编写，遵守java servlet API的一些类，原则上这些类可以响应任何类型的请求，我们一般用它来响应web方面的请求。

说的通俗一点就是，**servlet就是专门处理请求的一些类，主要作用就是接受请求，处理后返回结果。**

而Servlet容器又是什么呢？各个请求总要找到对应的servlet吧？每个请求总有一些状态需要管理吧？好了，servlet容器就是干这个的，它负责初始化并创建这些servlet，将请求解析成具体的类等一系列操作。比如我们常见的tomcat就是使用范围最广的servlet容器，其他的还有Jetty、jboss等。

### 一个例子

我们先看一段简单的servlet请求响应的代码片段

```java
//HelloServlet类
public class HelloServlet extends HttpServlet{

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String name  = req.getParameter("name");
        resp.setContentType("text/html;charset=UTF-8");
        PrintWriter printWriter = resp.getWriter();
        printWriter.write("<html><head><head><body><h1>");
        printWriter.write("Hello "+name);
        printWriter.write("</h1></body></html>");
        printWriter.close();
    }
}
```
然后在web.xml注册此servlet
```xml
<?xml version="1.0" encoding="utf-8"?>
<web-app version="3.0"
        xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
        http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">

    <display-name>wthfeng 的mvc练习项目</display-name>
    <servlet>
        <servlet-name>hello</servlet-name>
        <servlet-class>com.wthfeng.mymvc.servlet.HelloServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
  <servlet-mapping>
      <servlet-name>hello</servlet-name>
      <url-pattern>/hello</url-pattern>
  </servlet-mapping>
</web-app>
```

启动项目，效果如下图所示

![这里写图片描述](http://img.blog.csdn.net/20170429162918593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3RoZmVuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这样我们就写完了一个简单的servlet，有关这个例子，我们需要知道：

1. 编写Servlet必须继承`HttpServlet`或`GenericServlet`或直接实现`Servlet`接口。`HttpServlet`已经为我们封装了有关http请求的相关参数，一般情况下我们直接继承此类即可。
2. 此servlet重写了`doGet`方法，也就是它可以响应`/myprojectName/hello?name=xxx` 的请求，并在页面打印`xxx`的内容。


### Servlet的创建和初始化

好了，这些逻辑符合上面所说的，浏览器发出请求`/hello`，一个名为`hello`的 servlet被命中，接受请求处理后返回。这里我们再深入一些，为什么要把servlet的定义在web.xml中？web.xml又和servlet容器有什么关系？

**`web.xml`是web项目的入口文件**。在项目启动过程中，servlet容器（如tomcat）会读取`web.xml`并进行相应配置以初始化该项目。

具体容器启动流程如下（以tomcat为例）：

 1. 先解析tomcat路径下的conf/web.xml等web.xml文件,它们是全局web配置文件，做一些基本配置工作。如注册了default、jsp等servlet。

 2. 解析web.xml，将其各个配置项（包括servlet、filter、listener)经处理包装后设在Tomcat的Context容器中，一个 Web 应用对应一个 Context 容器。

 3. 创建servlet并初始化。`load-on-startup`大于1的servlet会在此时初始化。初始化servlet就是调用servlet的`init`方法。**注意：若不设置 此 值，init() 方法只在第一次 HTTP 请求命中时才被调用。**此时servlet容器就算启动了。

**简而言之就是，`web.xml`是项目和servlet容器（服务器）关联的桥梁。通过在`web.xml`注册servlet、filter、listener等，使得项目具有处理特定请求的功能。**

## 2. filter和listener

在有关web servlet的配置中，常能在`web.xml`看到filter、listener的配置。实际上，filter是过滤器，listener是监听器。这两项配置都是servlet中的重要部分。我们一一来看。

### fliter(过滤器)

filter可用于拦截请求，在请求处理前进行一些预处理工作，一般用于处理编码问题，记录日志等。Filter类需实现`javax.servlet.Filter`接口。其中有3个方法`init()、、dofilter()、destroy()`，分别用于过滤器的初始化、过滤处理、销毁。

先来个例子

```java
public class MyFilter implements Filter {

    private String param; 
     
    //初始化方法，在容器启动时调用
    public void init(FilterConfig filterConfig) throws ServletException {
        //做一些初始化操作
        param = filterConfig.getInitParameter("myParam");
        System.out.println("filter:"+param);
    }
 
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        //处理请求
        chain.doFilter(request,response); //调用下一个过滤器
        //处理响应
    }

    @Override
    public void destroy() {
      //在servlet销毁后销毁
      //做一些销毁后的善后工作
    }
}
```

在 `web.xml`中添加
```xml
     <filter>
        <filter-name>myFilter</filter-name>
        <filter-class>com.wthfeng.mymvc.filter.MyFilter</filter-class>
        <init-param>
            <param-name>myParam</param-name>
            <param-value>myValue</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>myFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

```

配置后filter就会拦截所有servlet请求，需注意：

1. 所有的filter在容器启动时即初始化。
2. filter的调用顺序为在web.xml中的定义顺序。若多余一个会形成过滤链依次处理。


### Listener(监听器)

servlet  监听器用于监听servlet容器中事件变化，当指定事件变化时，会触发注册该事件的监听器。监听器基于观察者模式。

Servlet监听器分为3类，分别用于监听`ServletContent`（Servlet上下文）、`HttpSession`（Session）,`HttpRequest`(Request)。

主要有以下几个类：

```java
ServletContextListener //监听Servlet容器创建销毁
ServletContextAttributeListener  //监听Servlet容器级别属性的添加及删除

HttpSessionListener  //监听Session创建销毁
HttpSessionAttributeListener //监听Session属性创建删除

ServletRequestListener  //监听请求创建及销毁
ServletRequestAttributeListener  //监听请求属性变化

```

如我们常见的Spring项目中的如下片段，就是Spring的监听器。其实现了`ServletContextListener`，用于监听Servlet上下文，以便在项目初始化时加载Spring的必要配置。

```xml
  <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
 </listener>
```

### Servlet 、Filter、Listerer 执行顺序

分别了解了servlet、filter、listener后，来确定一下他们之间的执行顺序。我们已经知道，filter在servlet之前，那listener呢？

应该想到，监听器负责监听事件变化，应该有最先执行的权限，测试一下，我把上述三种都加了日志，打印结果如下：

```java
//开启服务器并请求
listener 容器初始化开始  //servletContent 监听器 contextInitialized()
filter初始化        //filter init()
初始化servlet       //servlet init()
listener request初始化  // request 监听器requestInitialized()
开始filter            //fiter doFilter() chain.doFilter前
执行servlet        //servlet service()
结束filter          //fiter doFilter() chain.doFilter后
listener request销毁  //request 监听器 requestInitialized()

//关闭服务器
servlet销毁
filter销毁
listener 容器销毁
```
从此我们可以得出结论：

1.  对于涉及3者的部分，顺序为 listener - filter - servlet
2. filter和servlet的初始化部分，先filter后servlet
3. 销毁或结束顺序为加载顺序的反序

##3. 有关Servlet的注解

Servlet3后，添加了若干关于`servlet`,`filter`,`listener`的注解支持。分别为`@WebServlet`,`@WebFilter`,`@Webistener`。也就是说，可以不用`web.xml` 配置文件了。注册相关类型可直接在类上添加相应注解。

如，添加一个servlet上下文的监听器。可使用如下方式。

```java
@WebListener
public class ContentListener implements ServletContextListener {

  
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("listener 容器初始化开始");
    }
    
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("listener 容器销毁");
    }
}
```
相当方便有木有。。。


## 4. 总结

总结来说，狭义上servlet是指`javax.servlet` 这个servlet API类库。各大服务器厂商分别实现了这个标准并推出了诸如`Tomcat`、`jetty`等众多服务器。这些服务器已经帮我们实现了处理连接、协议，并根据servlet 标准帮我们封装好了请求及响应。总的这一套，称之为servlet的工作机制。


有了这些基础，我们就可以根据写自己的web框架了，下次见。







### 参考文章

1. [Servlet 工作原理解析](https://www.ibm.com/developerworks/cn/java/j-lo-servlet/)
2. [Java Servlet工作原理问答](http://www.importnew.com/17025.html)
3. [Java Servlet完全教程](http://www.importnew.com/14621.html)
4. [Filter与Servlet的区别和联系](http://blog.csdn.net/zs234/article/details/8832343)


















 






