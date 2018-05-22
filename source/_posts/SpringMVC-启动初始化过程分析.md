---
title: SpringMVC 启动初始化过程分析
date: 2018-05-14 17:42:56
tags: SpringMVC
categories: 源码分析
---

## SpringMVC 启动过程分析

之前我们写过`Servlet`的生命周期分析，在这个基础上，我们来看看SpringMVC启动过程中是如何初始化的，它都做了哪些事情。

先来看我们的`web.xml`中的部分配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         id="WebApp_ID" version="3.0">
    
    <!-- Spring 启动监听 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    
    <!-- Spring 配置文件所在目录 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:config/spring/spring.xml</param-value>
    </context-param>

    <!-- Spring mvc 配置 -->
    <servlet>
        <servlet-name>spring</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:config/spring/spring-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
        <async-supported>true</async-supported>
    </servlet>

    <servlet-mapping>
        <servlet-name>spring</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    
</web-app>
```

我们先得知道`web.xml`中组件的加载顺序：**context-param -> listener -> filter -> servlet** ，看到这个顺序后。根据这个顺序，我们来开始分析。

从这里开始初始化SpringIOC容器（Spring父容器）`ContextLoaderListener.contextInitialized`

```java
ContextLoaderListener
@Override
public void contextInitialized(ServletContextEvent event) {
    initWebApplicationContext(event.getServletContext());
}

ContextLoader
//调用ContextLoader.initWebApplicationContext方法
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	//...部分代码省略
	if (this.context == null) {
		//创建Spring Context上下文
		this.context = createWebApplicationContext(servletContext);
	}
	if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
		if (!cwac.isActive()) {
			if (cwac.getParent() == null) {
				ApplicationContext parent = loadParentContext(servletContext);
				cwac.setParent(parent);
			}
			//配置刷新容器
			//重要部分
			configureAndRefreshWebApplicationContext(cwac, servletContext);
		}
	}
	//...部分代码省略
}

ContextLoader
//调用ContextLoader.configureAndRefreshWebApplicationContext方法
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	//...部分代码省略
	wac.refresh();
}

AbstractApplicationContext
//调用AbstractApplicationContext.refresh方法
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		// 调用容器准备刷新的方法， 获取容器的当时时间， 同时给容器设置同步标识
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		// 在子类中启动refreshBeanFactory()的地方，也是IOC初始化过程，Bean 定位、载入、注册
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
		
		// Prepare the bean factory for use in this context.
		// 为 BeanFactory 配置容器特性， 例如类加载器、 事件处理器等
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			// 设置BeanFactory的后置处理
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			// 调用所有注册的 BeanFactoryPostProcessor 的 Bean
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			// 为 BeanFactory 注册 BeanPost 事件处理器.
			// BeanPostProcessor 是 Bean 后置处理器， 用于监听容器触发的事件
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			// 对上下文中的消息源进行初始化，初始化信息源， 和国际化相关.
			initMessageSource();

			// Initialize event multicaster for this context.
			// 初始化上下文中的事件机制
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			// 初始化其他 的特殊Bean
			onRefresh();

			// Check for listener beans and register them.
			// 为事件传播器注册事件监听器 
			registerListeners();
			
			// Instantiate all remaining (non-lazy-init) singletons.
			// 初始化所有剩余的单例 Bean
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			// 初始化容器的生命周期事件处理器， 并发布容器的生命周期事件
			finishRefresh();
		}
		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
			}

			destroyBeans();

			cancelRefresh(ex);

			throw ex;
		}
		finally {
			resetCommonCaches();
		}
	}
}
```

这里也就结束了SpringIOC 的全部流程。

调用图如下：

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc1.png?raw=true]()

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc2.png?raw=true]()

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc3.png?raw=true]()

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc4.png?raw=true]()

接下来，开始初始化SpringMVC容器（SpringMVC子容器）`DispatcherServlet`初始化，我们先来看一张`DispatcherServlet`继承图

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc5.png?raw=true]()

之前我们分析了`Servlet`的生命周期，不难得出分析`DispatcherServlet`的初始化，也就是从`Servlet.init`开始

```java
HttpServletBean
//调用HttpServletBean.init方法
@Override
public final void init() throws ServletException {
	//...部分代码省略
	// Let subclasses do whatever initialization they like.
	initServletBean();
	//...部分代码省略
}

FrameworkServlet
//调用FrameworkServlet.initServletBean方法
@Override
protected final void initServletBean() throws ServletException {
	//...部分代码省略
	try {
		//初始化SpringMCV上下文
		this.webApplicationContext = initWebApplicationContext();
		initFrameworkServlet();
	}
	catch (ServletException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}
	catch (RuntimeException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}
	//...部分代码省略
}

FrameworkServlet
//调用FrameworkServlet.initWebApplicationContext方法
protected WebApplicationContext initWebApplicationContext() {
	// 获取Spring父容器（SpringIOC）
	WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

	if (this.webApplicationContext != null) {
		// A context instance was injected at construction time -> use it
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				if (cwac.getParent() == null) {
					//设置父容器
					cwac.setParent(rootContext);
				}
				//配置刷新容器
				//重要部分
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		// No context instance is defined for this servlet -> create a local one
		wac = createWebApplicationContext(rootContext);
	}
	//...部分代码省略

	return wac;
}

FrameworkServlet
//调用FrameworkServlet.configureAndRefreshWebApplicationContext方法
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
	//...部分代码省略
	wac.refresh();
}

AbstractApplicationContext
//调用AbstractApplicationContext.refresh方法
@Override
public void refresh() throws BeansException, IllegalStateException {
	//...部分代码省略
	synchronized (this.startupShutdownMonitor) {
		try {
			//...部分代码省略
			// Instantiate all remaining (non-lazy-init) singletons.
			// 初始化所有剩余的单例 Bean，
			// 在初始化RequestMappingHandlerMapping的时候
			// 会设置所有的 映射关系，initHandlerMethods();
			// 拿到所有子容器中的 Bean，判断是否有Controller 和 RequestMapping注解，建立映射关系
			finishBeanFactoryInitialization(beanFactory);
			
			// Last step: publish corresponding event.
			// 初始化容器的生命周期事件处理器， 并发布容器的生命周期事件
			finishRefresh();
		}
		//...部分代码省略
	}
}

AbstractApplicationContext
//调用AbstractApplicationContext.finishRefresh方法
protected void finishRefresh() {
	//...部分代码省略
	// Publish the final event.
	// 发布事件
	// 最终调用到 FrameworkServlet.onApplicationEvent 然后调用 DispatcherServlet.onRefresh
	publishEvent(new ContextRefreshedEvent(this));
	
	//...部分代码省略

}

FrameworkServlet
//调用FrameworkServlet.onApplicationEvent方法
public void onApplicationEvent(ContextRefreshedEvent event) {
	this.refreshEventReceived = true;
	onRefresh(event.getApplicationContext());
}

DispatcherServlet
//调用DispatcherServlet.onRefresh方法
@Override
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}

DispatcherServlet
//调用DispatcherServlet.initStrategies方法
@Override
protected void initStrategies(ApplicationContext context) {
	// 初始化多文件上传解析器
    initMultipartResolver(context);
    // 初始化区域解析器
    initLocaleResolver(context);
    // 初始化主题解析器
    initThemeResolver(context);
    // 初始化处理器映射集
    initHandlerMappings(context);
    // 初始化处理器适配器
    initHandlerAdapters(context);
    // 初始化处理器异常解析器
    initHandlerExceptionResolvers(context);
    // 初始化请求到视图名转换器
    initRequestToViewNameTranslator(context);
    // 初始化视图解析器
    initViewResolvers(context);
    // 初始化 flash 映射管理器
    initFlashMapManager(context);
}
```

`DispatcherServlet` 初始化的工作就是负责初始化各种不同的组件。它好比给 SpringMVC 添加功能模块，一旦功能模块添加完毕，SpringMVC 就可以正常工作。因此 SpringMVC 的初始化工作也到此结束 。