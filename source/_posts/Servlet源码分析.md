---
title: Servlet 生命周期
date: 2018-05-03 14:00:33
tags:
---

一个Servlrt 的生命周期可以被定义为从创建到销毁的整个过程。以下是Servlrt 所遵循的路径

Servlrt 的生命周期分为三个阶段：

#### 	1、初始化阶段

```java
类加载器加载Servlet --> 容器创建Servlet --> 容器调用Servlet.init()
Servlet.init()只会被调用一次，它只在Servlet 创建的时候调用，之后的任何用户请求都不会在被调用。

默认情况下Servlet 是什么时候被创建的呢?
  我们在web.xml 里面配置我们的Servlet 
  
  代码如下
  <servlet>
    <!-- servlet的内部名称，自定义。尽量有意义 -->
    <servlet-name>defaultServlet</servlet-name>
    <!-- servlet的类全名： 包名+简单类名 -->
    <servlet-class>com.blog.demo.servlet.DefaultServlet</servlet-class>
  </servlet>
  <!-- servlet的映射配置 -->
  <servlet-mapping>
    <!-- servlet的内部名称，一定要和上面的内部名称保持一致！！ -->
    <servlet-name>defaultServlet</servlet-name>
    <!-- servlet的映射路径（访问servlet的名称） -->
    <url-pattern>/servlet</url-pattern>
  </servlet-mapping>
  
  Servlet
  public class DefaultServlet extends HttpServlet {

    @Override
    public void init() throws ServletException {
        System.out.println("DefaultServlet init");
        super.init();
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("DefaultServlet doGet");
        super.doGet(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("DefaultServlet doPost");
        super.doPost(req, resp);
    }
}

此时我们启动我们的容器，这里以tomcat容器为例，启动我们的tomcat 
查看启动日志，我们并没有发现控制台有打印 DefaultServlet init，这是我们访问一下我们的Servlet doGet方法
我们发现打印了两行日志
DefaultServlet init
DefaultServlet doGet
即Servlet 在我们访问的时候被创建，并且初始化，之后再次调用不再调用Servlet.init()方法。第一次调用Servlet.doGet()的方法时，tomcat 会判断容器中是否有这个Servlet 的实例，如果有则直接调用Servlet.doGet()，如果没有则创建Servlet 并调用 init 方法。
日志如下图：
```

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/rizhi1.png?raw=true)



####	2、相应客户端请求阶段

####	3、销毁阶段

