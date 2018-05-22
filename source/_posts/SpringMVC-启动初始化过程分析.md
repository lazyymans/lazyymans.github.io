---
title: SpringMVC 启动初始化过程分析
date: 2018-05-14 17:42:56
tags: SpringMVC
categories: 源码分析
---

## SpringMVC 启动过程分析

之前我们写过`Servlet`的生命周期分析，在这个基础上，我们来看看SpringMVC启动过程中是如何出事化的，它都做了哪些事情。

先来看看SpringMVC核心Servlet`DispatcherServlet`。看下他的继承关系：

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc1.png?raw=true)

我们可以看到他的顶级接口是Servlet，这是必然的。所以我们从Servlet 的`init` 开始一步一步往下分析：

--HttpServletBean.init()

​	--FrameworkServlet.initServletBean()

​					   .initWebApplicationContext()

​		--DispatcherServlet.onRefresh()

​						  .initStrategies()





```java
RequestMappingHandlerMapping.class

@Override
protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}

AbstractHandlerMethodMapping.class

protected void registerHandlerMethod(Object handler, Method method, T mapping) {
    this.mappingRegistry.register(mapping, handler, method);
}
```

91