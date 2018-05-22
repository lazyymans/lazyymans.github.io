---
title: SpringMVC 请求过程
date: 2018-05-23 03:14:25
tags: SpringMVC
categories: SpringMVC
---

## SpringMVC 请求过程

之前，我们写过`Servlet的生命周期`，现在我们来看看SpringMVC `DispatcherServlet`中是如何处理请求的。

分析，`DispatcherServlet`集成顶端接口`Servlet`，那么真个请求，也就是从`Servlet.service`开始。下面我们开始分析。

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc5.png?raw=true]()

```java
HttpServlet
//调用 HttpServlet.service
public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
	HttpServletRequest request;
	HttpServletResponse response;
	try {
		request = (HttpServletRequest)req;
		response = (HttpServletResponse)res;
	} catch (ClassCastException var6) {
		throw new ServletException("non-HTTP request or response");
	}
	
	//调用this.service，这里也就是调用DispatcherServlet.service，如果没有，就找父类的service方法
	this.service(request, response);
}

FrameworkServlet
//调用 FrameworkServlet.service
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {

	HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
	// in order to intercept PATCH requests，异常执行
	if (HttpMethod.PATCH == httpMethod || httpMethod == null) {
		processRequest(request, response);
	}
	else {
		//正常执行
		super.service(request, response);
	}
}

HttpServlet
//调用 HttpServlet.service，这里可以看到，全部执行this.doGet、this.doPost ... 方法
//也就是调用DispatcherServlet.doxxx 方法
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

FrameworkServlet
//doxxx中调用processRequest
//调用 FrameworkServlet.processRequest
@Override
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	//...部分代码省略
	try {
		doService(request, response);
	}
	catch (ServletException ex) {
		failureCause = ex;
		throw ex;
	}
    //...部分代码省略
}

DispatcherServlet
//调用 DispatcherServlet.doService
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	//...部分代码省略
	try {
		doDispatch(request, response);
	}
	finally {
		if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}
}

DispatcherServlet
//调用 DispatcherServlet.doDispatch
//最终调到了这里，我们进行详细分析
@Override
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			// 1、文件上传解析，如果请求类型是multipart将通过MultipartResolver进行文件上传解析；
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// Determine handler for the current request.
			// 2.通过HandlerMapping，获取到本次请求的执行器链（返回一个HandlerExecutionChain，它包括一个处理器、多个HandlerInterceptor拦截器）；
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// Determine handler adapter for the current request.
			// 3.从HandlerExecutionChain 对象 获取到 HandlerAdapter（处理器适配器）
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (logger.isDebugEnabled()) {
					logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
				}
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}
			
			// 4.执行HandlerExecutionChain 中拦截器preHandle
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			// 5.执行目标方法（Controller 中的实际映射方法），返回ModelAndView 对象
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(processedRequest, mv);
			// 6.执行HandlerExecutionChain 中拦截器postHandle
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
		
		// 7.处理视图并渲染视图
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

SpringMVC核心架构图

![https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/springmvc6.png?raw=true]()

```
1. 用户向服务器发送请求，请求被Spring 前端控制Servelt DispatcherServlet捕获；

2. DispatcherServlet对请求URL进行解析，得到请求资源标识符（URI）。然后根据该URI，调用HandlerMapping获得该Handler配置的所有相关的对象（包括Handler对象以及Handler对象对应的拦截器），最后以HandlerExecutionChain对象的形式返回；

3. DispatcherServlet 根据获得的Handler，选择一个合适的HandlerAdapter。（附注：如果成功获得HandlerAdapter后，此时将开始执行拦截器的preHandler(...)方法）

4. Handler执行完成后，向DispatcherServlet 返回一个ModelAndView对象；

5. 根据返回的ModelAndView，选择一个适合的ViewResolver（必须是已经注册到Spring容器中的ViewResolver)返回给DispatcherServlet ；

6. DispatcherServlet 执行拦截器的postHandle(...)方法

7. ViewResolver 结合Model和View，来渲染视图

8. 将渲染结果返回给客户端。
```