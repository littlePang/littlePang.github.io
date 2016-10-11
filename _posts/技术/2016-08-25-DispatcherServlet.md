  ---
layout: post
title: DispatcherServlet学习
category: 学习
tags: study
keywords:
description:
---



 http请求的分发中心,将请求分发给对应的请求处理器,支持方便的请求映射和异常处理.

 基于javaBeans的灵活配置:
 * 使用`HandlerMapping`处理请求映射(默认是`BeanNameUrlHandlerMapping` 和 `DefaultAnnotationHandlerMapping`)
* 使用`HandlerAdapter`(`HttpRequestHandler`默认适配器是`HttpRequestHandlerAdapter`,`Controller`默认适配器是`SimpleControllerHandlerAdapter`,还有针对注解的适配器`AnnotationMethodHandlerAdapter`)
* 使用 `HandlerExceptionResolver`处理异常.(默认有`AnnotationMethodHandlerExceptionResolver`,`ResponseStatusExceptionResolver`,`DefaultHandlerExceptionResolver`)
* 使用`ViewResolver`处理视图方案策略.解决逻辑视图到实际视图对象的映射.(默认是`InternalResourceViewResolver`)
* 使用`RequestToViewNameTranslator`处理用户提供的`View`或者试图名称.(默认是`DefaultRequestToViewNameTranslator`, 覆盖配置时beanName为 _viewNameTranslator_)
* 使用`MultipartResolver`解决分块请求(例如文件上传),可用`CommonsMultipartResolver`定制处理器,(没有默认处理器,配置时beanName为 _multipartResolver_)
* 使用`LocaleResolver`解决本地化解决策略.(默认是`AcceptHeaderLocaleResolver`, 配置时beanName为 _localeResolver_)
* 使用`ThemeResolver`决定主题解决方案.(默认是`FixedThemeResolver`, 配置时beanName为 _themeResolver_)

注意:
只有在相关的`HandlerMapping`(Type级别的注解处理)和`HandlerAdapter`(method级别的注解处理)在dispatcher是存在的,才会处理`@RequestMapping`注解(默认就是存在的).如果你定制自己的`HandlerMappings`或者`HandlerAdapters`,则需要保证配置了相应的`DefaultAnnotationHandlerMapping`和`AnnotationMethodHandlerAdapter`,这样才能正常使用`@RequestMapping`

一个web应用可以定义多个`DispacherServlet`,他们之间的配置(mappings, handlers等)都是在各自的命名空间下独享的.但是由`org.springframework.web.context.ContextLoaderListener`加载的 根应用上下文 在他们中间是共享的.

spring3.1 支持以注入的方式创建`DispatcherServlet`,而不是在内部自己创建,在Servlet3.0的环境中是比较有用的(可通过代码的方式注册servlet实例), 具体细节看`DispatcherServlet(WebApplicationContext)`的文档


DispacherServlet的整个类继承关系如下.
![](/assets/picture/2016-08-29_dispatcherServlet_inherit.png)
DispacherServlet继承自FrameworkServlet,在FrameworkServlet中将所有的http请求方法(get,post,delete,put等)都交由方法 doService() 处理, 所以DispatcherServlet处理请求的所有逻辑,都在这个doService()方法中.


        protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {

        		// 保存请求中的所有属性,在最后DipatcherServlet处理完后,将请求中的属性全部还原回去
        		Map<String, Object> attributesSnapshot = null;
        		if (WebUtils.isIncludeRequest(request)) {
        			attributesSnapshot = new HashMap<String, Object>();
        			Enumeration<?> attrNames = request.getAttributeNames();
        			while (attrNames.hasMoreElements()) {
        				String attrName = (String) attrNames.nextElement();
        				if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
        					attributesSnapshot.put(attrName, request.getAttribute(attrName));
        				}
        			}
        		}

        		// 在请求中放入, 上下文, 本地化,主题,等处理器
        		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
        		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
        		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
        		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
        		FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
        		if (inputFlashMap != null) {
        			request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
        		}
        		request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        		request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

            // 请求派发,将请求交由对应的控制器处理(包含拦截器调用)
        		try {
        			doDispatch(request, response);
        		}
        		finally {
        			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        				// 将最开始保存的请求属性全部还原回去
        				if (attributesSnapshot != null) {
        					restoreAttributesAfterInclude(request, attributesSnapshot);
        				}
        			}
        		}
        	}

真正派发请求到对应的控制器是在 doDispatch() 中:

        try {
        				processedRequest = checkMultipart(request);
        				multipartRequestParsed = (processedRequest != request);

        				// 获取当前的请求对应的处理器(包括拦截器和真实处理请求的控制器)
        				mappedHandler = getHandler(processedRequest);
        				if (mappedHandler == null || mappedHandler.getHandler() == null) {
        					noHandlerFound(processedRequest, response);
        					return;
        				}

        				// 获取当前请求对于的处理器适配器
        				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        				// 这里是在处理支持 last-modified 请求头的情况
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

                // 执行 所有拦截器的preHandle()方法,若有拦截器的preHandle未通过,则会执行所有已经执行过的拦截器的 afterCompletion() 方法
                // 拦截器未通过,则直接结束请求.
        				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        					return;
        				}

        				// 真正的控制器处理请求
        				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                // 这里是处理servlet3.0支持的异步处理的请求
        				if (asyncManager.isConcurrentHandlingStarted()) {
        					return;
        				}

                // 设置默认视图名称
        				applyDefaultViewName(processedRequest, mv);

                // 调用所有拦截器的 postHandle() 方法(与preHandle()执行顺序的相反)
        				mappedHandler.applyPostHandle(processedRequest, response, mv);
        			}
        			catch (Exception ex) {
        				dispatchException = ex;
        			}

              // 处理派发请求的结果,做视图渲染,异常处理,并且会调用所有拦截器的 afterCompletion() 方法
        			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);


# `RequestMappingHandlerMapping` 判断一个类是否是 控制器

那按照如下的方式判断,则是不是控制器不用使用`@Controller`声明也可以,只需要`@RequstMapping`即可?

      @Override
      	protected boolean isHandler(Class<?> beanType) {
      		return ((AnnotationUtils.findAnnotation(beanType, Controller.class) != null) ||
      				(AnnotationUtils.findAnnotation(beanType, RequestMapping.class) != null));
      	}

# preflight request 预飞请求

浏览器为了安全起见，会先发送一个 options 请求，确保请求发送是安全的，一般 POST DELETE PUT 等请求都会修改服务器资源，所以浏览器会先发一个请求，问问服务器是否会正确（允许）请求。

出现 OPTIONS 的情况一般为(跨域请求也会)：

非 GET | POST 请求
POST 请求的 content-type 不是常规的那三个
POST 请求的 payload 为 text/xml
