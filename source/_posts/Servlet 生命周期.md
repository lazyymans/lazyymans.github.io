---
title: Servlet 生命周期
date: 2018-05-03 14:00:33
tags:
---

一个Servlrt 的生命周期可以被定义为从创建到销毁的整个过程。以下是Servlrt 所遵循的路径，我们先看下Servlet

有哪些方法，中点关注那些方法（init()、service()、destroy()）

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/tupian1.png?raw=true)

Servlrt 的生命周期分为三个阶段：

#### 	1、初始化阶段

```java
类加载器加载Servlet --> 容器创建Servlet --> 容器调用Servlet.init()
Servlet.init()只会被调用一次，它只在Servlet 创建的时候调用，之后的任何用户请求都不会在被调用。
这里在Servlet 创建并初始化的时候，tomcat 会把它放到容器的缓存中来进行管理，在下一次访问的时候看缓存中
是否有这个Servlet，如果有则直接拿出来调用。

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
我们看打印的日志
DefaultServlet construct
DefaultServlet init
DefaultServlet doGet
即Servlet 在我们访问的时候被创建，并且初始化，之后再次调用不再调用Servlet.init()方法。第一次调用
Servlet.doGet()的方法时，tomcat 会判断容器中是否有这个Servlet 的实例，如果有则直接调用
Servlet.doGet()，如果没有则创建Servlet 并调用 init 方法。之后再次访问就不再调用Servlet.init()
日志如下图：
```

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/rizhi1.png?raw=true)

```
通常我们会在 web.xml Servlet 配置里面加上 <load-on-startup>1</load-on-startup> 这样一个配置，这个
配置的作用就是告诉web容器(tomcat) 在容器启动时就加载这个Servlet，现在我们把这个配置加上，然后再启动web容器，我们发现tomcat 在启动时就帮我们创建了Servlet 并且初始化
  <servlet>
    <!-- servlet的内部名称，自定义。尽量有意义 -->
    <servlet-name>defaultServlet</servlet-name>
    <!-- servlet的类全名： 包名+简单类名 -->
    <servlet-class>com.blog.demo.servlet.DefaultServlet</servlet-class>
    <!-- 默认容器启动时创建Servlet -->
    <load-on-startup>1</load-on-startup>
  </servlet>
  <!-- servlet的映射配置 -->
  <servlet-mapping>
    <!-- servlet的内部名称，一定要和上面的内部名称保持一致！！ -->
    <servlet-name>defaultServlet</servlet-name>
    <!-- servlet的映射路径（访问servlet的名称） -->
    <url-pattern>/servlet</url-pattern>
  </servlet-mapping>
```

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/rizhi2.png?raw=true)

####	2、相应客户端请求阶段

```java
Servlet 在创建好之后，就处于响应就绪状态，只要有请求过来就会执行Servlet.service()。为什么我们在上面直接
说请求Servlet.doGet()方法呢？
以下是HttpServlet 对service 的处理
public abstract class HttpServlet extends GenericServlet implements Serializable {
	public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
        HttpServletRequest request;
        HttpServletResponse response;
        try {
            request = (HttpServletRequest)req;
            response = (HttpServletResponse)res;
        } catch (ClassCastException var6) {
            throw new ServletException("non-HTTP request or response");
        }

        this.service(request, response);
    }
    
	protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod();
        long lastModified;
        if (method.equals("GET")) {
            lastModified = this.getLastModified(req);
            if (lastModified == -1L) {
                this.doGet(req, resp);
            } else {
                long ifModifiedSince = req.getDateHeader("If-Modified-Since");
                if (ifModifiedSince < lastModified / 1000L * 1000L) {
                    this.maybeSetLastModified(resp, lastModified);
                    this.doGet(req, resp);
                } else {
                    resp.setStatus(304);
                }
            }
        } else if (method.equals("HEAD")) {
            lastModified = this.getLastModified(req);
            this.maybeSetLastModified(resp, lastModified);
            this.doHead(req, resp);
        } else if (method.equals("POST")) {
            this.doPost(req, resp);
        } else if (method.equals("PUT")) {
            this.doPut(req, resp);
        } else if (method.equals("DELETE")) {
            this.doDelete(req, resp);
        } else if (method.equals("OPTIONS")) {
            this.doOptions(req, resp);
        } else if (method.equals("TRACE")) {
            this.doTrace(req, resp);
        } else {
            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[]{method};
            errMsg = MessageFormat.format(errMsg, errArgs);
            resp.sendError(501, errMsg);
        }

    }
}
通过HttpServlet 的源码，我们知道在调用service 方法的时候，它会根据请求类型，调用自身的对应的 doGet，
doPost，doPut，doDelete ...等方法
所以我们在自定义的时候只要Override 这些方法即可。
当请求过来的时候，web容器会创建一个新的线程来处理这个请求，web容器来调用对应Servlet.service()，方法
Servlet.service()根据请求调用doGet、doPost等方法并完成请求响应，之后线程结束，请求结束。
```

####	3、销毁阶段

```
web容器销毁阶段，会调用应用中所有的Servlet.destroy()。该方法在整个生命周期也是只被调用一次，在Servlet 
被销毁的时候，此时我们可以释放掉Servlet 所占用的资源。例如关闭与数据库的连接。
下图是容器销毁时调用
```

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/xiaohui1.png?raw=true)