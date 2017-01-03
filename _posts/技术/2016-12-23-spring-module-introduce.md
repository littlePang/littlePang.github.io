---
layout: post
title: Spring Framework各模块简介
category: 技术
tags: spring
keywords:
description:
---

# Spring Framework 是什么
Spring Framework框架是一个轻量级,一站式构建企业级应用程序的解决方案, 而且它细分为各个模块,允许在应用程序中只引用必需的部分.(例如你可以在任何web框架中,只使用它的IoC部分).
它被设计为非侵入式的,那意味着你的逻辑代码并不会依赖于框架本身(可能会在例如数据层的一些地方依赖了spring包的数据访问技术,但是应该很容易在代码中将这些依赖隔离出来).


# 模块概述
Spring Framwork框架包含20个模块,各个模块包含不同的特性.这些模块分为核心容器,数据访问/整合,Web,AOP,Instrumentation,消息和测试.具体如下图所示:

![](/assets/picture/2016-12-28_1.png)

# 模块分类说明

## Core Container

由 `spring-core`,`spring-beans`,`spring-context`,`spring-contest-support`,`spring-expression`(Spring Expression Language) 这些包组成.

`spring-core` 和 `spring-beans` 提供包括控制翻转,依赖注入等最基本的特性.`BeanFactory`是工厂模式的一个复杂实现.它无需单独编程,并且允许你通过配置使它从真正的逻辑代码的依赖中解耦出来.

`context`模块建立在 `spring-core`和`spring-beans`提供的件事基础上,拿意味着以框架的风格来访问对象就和JNDI([JNDI说明](http://blog.csdn.net/zhaosg198312/article/details/3979435))注册一样简单, context模块继承了 beans 模块的所有特性,并且增加了 国际化(例如资源捆绑),事件传播, 资源加载,一些语义的透明创建(例如Servlet容器)的特性支持. context模块还支持 Java EE 中例如 EJB,JMX,basic remoting 等特性. `ApplicationContext`接口是context模块的一个重点.

`spring-context-support` 模块提供了将常用第三方包接入spring应用的支持,例如 缓存(EhCache,Guava,JCache), 邮件(JavaMail),调度器(CommonJ,Quartz),以及模板引擎(FreeMarker, JasperReports, Velocity).

`spring-expression` 模块提供了一个 在运行时用于查询和操作对象图的 强大的表达式语言.它是JSP2.1中指定的统一扩展语言(unified EL)的扩展.这个语言支持设置和获取对象的属性值,属性赋值,方法执行,访问数组,集合,索引等的内容,逻辑和算术操作,变量命名,以及通过对象名称从springIOC容器中回收对象等操作.它还支持集合的投影和选择,普通的list聚合操作.

## AOP and Instrumentation
`spring-aop`模块提供了一个与AOP联盟兼容的 并且允许你自定义的 面向切面的编程实现.例如,方法拦截器和切点可以方便的解耦应该被分割功能实现代码.

`spring-aspects`模块提供和AspectJ的整合

`spring-instrument`提供 整个服务端应用程序的 类设备支持 和 类加载器实现.

`spring-instrument-tomcat`模块包含spring对tomcat的设备代理.

## Messaging
Spring Framework 4 的 `spring-messaging`模块包含和spring整合项目的关键抽象(例如`Message`,`MessageChannel`,`MessageHandler`),以及作为其他一些基于消息的应用的基础.这个模块还包括一个用来映射消息到方法的注解集合,和 Spring MVC 编程模型中的注解类似.


## Data Access/Integration

由 JDBC, ORM, JMS, 以及事务模块组成

`spring-jdbc`模块提供JDBC的抽象层.可用从你的代码中移除冗长的JDBC代码以及数据供应商特殊错误代码解析.

`spring-tx`模块提供 编程式和声明式 事务管理.

`spring-orm`模块 对常用的对象关系映射api提供整合层(包括JPO, JDO, 以及 Hibernate). 这个模块让你可以使用spring提供的所有这些对象映射框架混合的特性.例如前面说到的声明式事务管理特性.

`spring-oxm`模块提供 对象和xml的映射的支持(例如JAXB, Castor, XMLBeans, JiBX 和 XStream)的抽象层

`spring-jms`(Java Messaging Service) 模块包含 生产和消费消息的特性.从Spring Framework 4.1开始,提供和`spring-messaging`的整合.


## Web
web层包括 `spring-web`, `spring-webmvc`, `spring-websocket`, 以及 `spring-webmvc-portlet`模块

`spring-web`一些基本的面向web的特性,例如 文件上传, 使用Servlet Listeners初始化IoC容器, 以及面向web的应用上下文(Application Context), 还有HTTP客户端,Spring远程支持的面向web的部分.

`spring-webmvc`模块(也称为`Web-Servlet`模块) 包含web应用中 Spring的MVC(model-view-controller)和REST风格的web服务 的实现. springMVC框架提供一个在域模型和web表单直接的明确分割,并且和spring的其他模块的整合.

`spring-webmvc-portlet`模块(也称为`Web-Portlet`模块)提供Portlet环境的MVC实现,以及`spring-webmvc`模块的功能镜像.


## Test
`spring-test`模块支持使用`JUnit`或者`TestNG`对应Spring组件进行单元测试和集成测试,它提供一致的`ApplicationContext`加载 以及 缓存这些上下文.它还支持在测试代码中mock对象([mock 对象](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mock-objects)).


# 参考
[http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#spring-introduction](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#spring-introduction)
