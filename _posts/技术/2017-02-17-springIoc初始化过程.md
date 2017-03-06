---
layout: post
title: Spring IoC Container启动过程
category: 技术
tags: spring
keywords:
description:
---

前面讲了下 [springIOC容器的使用](/2016-12-23-spring-ioc-container),接下来,就可以扯一扯它整个的加载过程了.

以ClassPathXmlApplicationContext的初始化为例,它的整个载入过程,都是由与个刷新函数 `refresh()` 来触发的.

          public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
          			throws BeansException {

          		super(parent);
          		setConfigLocations(configLocations);
          		if (refresh) {
          			refresh();
          		}
          	}


在`refresh()`中,可以看到整个上下文的初始化流程一共是12步:

          synchronized (this.startupShutdownMonitor) {
          			// 1.刷新的前置准备
          			prepareRefresh();

          			// 2.获取一个新的beanFactory
          			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

          			// 3.将新的beanFactory在使用前做一些前置准备
          			prepareBeanFactory(beanFactory);

          			try {
          				// 4.执行beanFactory刚创建好后的处理钩子
          				postProcessBeanFactory(beanFactory);

          				// 5.执行在上下文中注册的 BeanFactoryPostProcessor 的实现
          				invokeBeanFactoryPostProcessors(beanFactory);

          				// 6.执行山下文中注册的 BeanPostProcessor 的实现
          				registerBeanPostProcessors(beanFactory);

          				// 7.为上下文初始化消息源
          				initMessageSource();

          				// 8.初始化事件广播处理类
          				initApplicationEventMulticaster();

          				// 9.子类初始化的回调钩子
          				onRefresh();

          				// 10.向事件广播处理类注册事件监听者
          				registerListeners();

          				// 11.初始化剩下的非懒加载的单例bean
          				finishBeanFactoryInitialization(beanFactory);

          				// 12.最后发布相关的容器事件
          				finishRefresh();
          			}

接下来详细看初始化流程的每一步所做的事情

###  prepareRefresh() 主要代码

              // 设置上下文的状态
              this.startupDate = System.currentTimeMillis();
          		this.closed.set(false);
          		this.active.set(true);

          		// 由子类来实现的初始化占位符的方法
          		initPropertySources();

          		// 校验所有初始化必须要的属性是否存在(通过 ConfigurablePropertyResolver#setRequiredProperties 设置)
              // 看实现可以发现默认情况下什么都不会校验, 子类可以通过覆盖 getEnvironment() 方法来校验指定的属性
          		getEnvironment().validateRequiredProperties();

          		// 用来保存在上下文初始化完成前的事件,待事件广播者可用时,就会发布这里的事件.
          		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();


### obtainFreshBeanFactory()

        protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {

            // 这个会创建一个新的BeanFactory,如果之前已经有一个BeanFactory,则会先将其销毁掉.
        		refreshBeanFactory();
        		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        		if (logger.isDebugEnabled()) {
        			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
        		}
        		return beanFactory;
        	}

refreshBeanFactory()主要代码如下:

          try {
                // 创建一个新的待使用的BeanFactory
                DefaultListableBeanFactory beanFactory = createBeanFactory();
          			beanFactory.setSerializationId(getId());

                // 设置一些可定制的属性(是否允许bean定义覆盖,循环依赖等)
          			customizeBeanFactory(beanFactory);

                // 这里就是整个BeanFactory的加载过程了, 包括配置文件解析,Bean定义构建,以及BeanDefinition的注册.
          			loadBeanDefinitions(beanFactory);
          			synchronized (this.beanFactoryMonitor) {
          				this.beanFactory = beanFactory;
          			}
          		}

loadBeanDefinitions() 的主要代码:

              // 为BeanFactory新建一个文件读取器
          		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

              // 配置bean定义的读取器,设置环境变量,设置资源加载器,设置键值对处理器
          		beanDefinitionReader.setEnvironment(this.getEnvironment());
          		beanDefinitionReader.setResourceLoader(this); // 当前的ApplicationContext也是一个ResourceLoader
          		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

              // 在真正加载BeanDefinition之前,可由之类进行定制处理
          		initBeanDefinitionReader(beanDefinitionReader);

              // 加载BeanDefinition
          		loadBeanDefinitions(beanDefinitionReader);

然后再往下看,可以发现 loadBeanDefinitions() 的加载过程实际是交由BeanDefinitionReader来做的,所以BeanDefinition的加载过程可在`XmlBeanDefinitionReader#loadBeanDefinitions()`中查看

`XmlBeanDefinitionReader#loadBeanDefinitions()`主要代码

        // 这一块在做是否有循环引用文件的情况(例如 文件A引用了文件B, 文件B又引用了文件A, 这种时候会抛出循环加载异常)
            Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        		if (currentResources == null) {
        			currentResources = new HashSet<EncodedResource>(4);
        			this.resourcesCurrentlyBeingLoaded.set(currentResources);
        		}
        		if (!currentResources.add(encodedResource)) {
        			throw new BeanDefinitionStoreException(
        					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        		}


            // 加载当前文件中的所有Bean定义
        		try {
        			InputStream inputStream = encodedResource.getResource().getInputStream();
        			try {
        				InputSource inputSource = new InputSource(inputStream);
        				if (encodedResource.getEncoding() != null) {
        					inputSource.setEncoding(encodedResource.getEncoding());
        				}

                // 这里 加载指定数据流中的所有Bean定义
        				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        			}
        			finally {
        				inputStream.close();
        			}
        		}
        		catch (IOException ex) {
        			throw new BeanDefinitionStoreException(
        					"IOException parsing XML document from " + encodedResource.getResource(), ex);
        		}

doLoadBeanDefinitions() 主要代码:

            try {
                // 从输入流中载入一个文档(这里使用了JDK中提供的关于xml文件解析组件)
          			Document doc = doLoadDocument(inputSource, resource);

                // 然后将其解析配置信息并注册到
          			return registerBeanDefinitions(doc, resource);
          		}

在registerBeanDefinitions中是将整个注册过程委派给`BeanDefinitionDocumentReader`执行的.

`DefaultBeanDefinitionDocumentReader#registerBeanDefinitions`中由是调用`DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions`进行解析注册的:

                  //任何在<beans>中的元素都会在这个方法中造成递归调用.
                  // 为了正确传播上一个<beans>的默认属性
                  // 需要保存当前的上级元素, 然后创建新的委派对象,最后再将 this.delegate 重新设置为 最初的引用
                  // 这个操作模仿的是一个栈的行为
                  BeanDefinitionParserDelegate parent = this.delegate;
              		this.delegate = createDelegate(getReaderContext(), root, parent);

              		if (this.delegate.isDefaultNamespace(root)) {
              			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
              			if (StringUtils.hasText(profileSpec)) {
              				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
              						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
              				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
              					if (logger.isInfoEnabled()) {
              						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
              								"] not matching: " + getReaderContext().getResource());
              					}
              					return;
              				}
              			}
              		}

              		preProcessXml(root);
              		parseBeanDefinitions(root, this.delegate);
              		postProcessXml(root);

              		this.delegate = parent;


在 `parseBeanDefinitions()` 中:

          protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
          		if (delegate.isDefaultNamespace(root)) {
          			NodeList nl = root.getChildNodes();
          			for (int i = 0; i < nl.getLength(); i++) {
          				Node node = nl.item(i);
          				if (node instanceof Element) {
          					Element ele = (Element) node;
          					if (delegate.isDefaultNamespace(ele)) {
                    // 处理默认命名空间 http://www.springframework.org/schema/beans 下的元素
                     // 例如 import, alias, profile等元素的解析
          						parseDefaultElement(ele, delegate);
          					}
          					else {
                     // 这里处理子定义命名空间的元素解析
          						delegate.parseCustomElement(ele);
          					}
          				}
          			}
          		}
          		else {
          			delegate.parseCustomElement(root);
          		}
          	}

在 `	delegate.parseCustomElement()` 中:

这里去拿当前元素的命名空间对应的NamespaceHandler, 元素的命名空间也就是在配置文件中`<beans/>元素中的xmlns属性值(例如:`xmlns:context="http://www.springframework.org/schema/context"`)
, 每一个命名空间对应的处理类默认配置在 `META-INF/spring.handlers`文件下, 在执行`this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);`这里
时,会自动去加载配置文件, 并获取配置中命名空间uri指定的处理类(例如在spring-context包的`META-INF/spring.handlers`,可见`http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler`, 即:
命名空间`http\://www.springframework.org/schema/context` 使用的元素处理类为 `org.springframework.context.config.ContextNamespaceHandler`), 具体怎么加载这个文件的代码在`DefaultNamespaceHandlerResolver#getHandlerMappings()`中可见.


          public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
          		String namespaceUri = getNamespaceURI(ele);
          		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
          		if (handler == null) {
          			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
          			return null;
          		}
          		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
          	}


这样一路跟下来,所有BeanDefinition的定义载入过程就已经完成了.

# prepareBeanFactory(beanFactory)

主要设置一些spring预定义的一些处理类,例如 `ApplicationContextAwareProcessor` `ApplicationEventPublisher`等.

# postProcessBeanFactory(beanFactory)

子类可覆盖重写的,用于在所有bean都未实例化时的一些处理.

# invokeBeanFactoryPostProcessors(beanFactory)

执行所有的 `BeanFactoryPostProcessor`, 先执行`BeanDefinitionRegistryPostProcessor`,然后再执行其他的`BeanFactoryPostProcessor`

# registerBeanPostProcessors(beanFactory)
注册所有的`BeanPostProcessor`,并按钮实现了`Ordered`的顺序排序, 以便在bean真正实例化的时候进行调用.

# initMessageSource()
实例化配置中的`MessageSource`实现. _*暂不知这个MessageSource是使用场景,挖坑待填*_

# initApplicationEventMulticaster()
初始化`ApplicationEventMulticaster`实现,为后续应用事件广播,准备

# onRefresh()
由子类来重写定制的,刷新操作

# registerListeners()

将 应用事件监听器注册到应用事件广播器中,并广播之前在`earlyApplicationEvents`中保存的应用事件.

# finishBeanFactoryInitialization(beanFactory)

初始化所有的单例Bean,以及非懒加载的Bean, 这里还会有一个冻结配置的操作( beanFactory.freezeConfiguration() ), 即bean加载后,不期望配置再有任何改动.

# finishRefresh()

          protected void finishRefresh() {
          		// Initialize lifecycle processor for this context.
          		initLifecycleProcessor();

          		// Propagate refresh to lifecycle processor first.
          		getLifecycleProcessor().onRefresh();

          		// Publish the final event.
          		publishEvent(new ContextRefreshedEvent(this));

          		// Participate in LiveBeansView MBean, if active.
          		LiveBeansView.registerApplicationContext(this);
          	}

其实spring容器的加载过程主要是三个步骤

第一步 配置文件的定位

第二步 配置载入

第三步 bean注册
