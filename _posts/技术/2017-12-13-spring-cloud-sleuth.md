---
layout: post
title: spring-cloud-sleuth使用
category: 技术
tags: spring-cloud
keywords:
description:
---

# 说明
此文档基于 spring-cloud-sleuth 的 1.2.5-RELEASE 版本，官方文档：[传送门](http://cloud.spring.io/spring-cloud-static/spring-cloud-sleuth/1.2.5.RELEASE/single/spring-cloud-sleuth.html)

# 名词解释

* Span: 工作的基本单位，对它的基本定义是一个 64字节的唯一ID，它还包含一些例如 描述，时间戳事件，键值对注释等信息。
* Trace: 一个由Span集合组成的树形结构
* Annotation：用来记录时间戳事件，一些用来定义启动和结束的核心注释：
  * cs（Client Send）：客户端发送请求，描述一个Span的启动
  * sr（Server Reveived）：服务端接收到一个请求，并开始处理它（用它减去cs的时间戳就是网络延迟时间）
  * ss（Server Send）：服务端完成请求处理，并发送请求返回结果。（用它减去 sr 即服务端处理请求所花费的时间）
  * cr（Client Received）：客户端接收到请求结果，表示 span的结束。客户端成功的接收到了服务端的处理结果。（用它减去 cs 的时间戳即为 整个从客户端到服务端处理完成并返回结果所花费所有时间）

一个整体的请求，从请求发起到请求返回，过程中 TraceId 和 SpanId的变化，可见下图，整个请求过程中 TraceId 不变，每一个内部请求都对应一个新的 SpanId.（图来自spring-cloud-sleuth官方文档：[地址](http://cloud.spring.io/spring-cloud-static/spring-cloud-sleuth/1.2.5.RELEASE/single/spring-cloud-sleuth.html)）

![](/assets/picture/2018-01-29_1.png)

# 使用

需要将 traceId 和 spanId 将入 slf4j 的 MDC 当中，用于日子收集器提取日志，例如：

>>
2016-02-02 15:30:57.902  INFO [bar,6bfd228dc00d216b,6bfd228dc00d216b,false] 23030 --- [nio-8081-exec-3] ...
>>
2016-02-02 15:30:58.372 ERROR [bar,6bfd228dc00d216b,6bfd228dc00d216b,false] 23030 --- [nio-8081-exec-3] ...

上面INFO后面中括号中信息依次为：[appname,traceId,spanId,exportable]，appname为应用名称，exporttable为是否需要被zipkin提取

# Span 生命周期

可通过 `org.springframework.cloud.sleuth.Tracer` 接口操作span的生命周期

* start: 记录一个Span被命名的开始
* close: span 的完成，如果 exportable 为true，则会被 zipkin收集，并且这个span将会从当前线程中移除。
* continue: 创建一个span的应用，但是它是一个继续的副本
* detach: span不是停止也不是关闭，只是从当前线程中移除
* create with explicit parent：创建一个新的span，并明确指定一个parent span

*永远记得在创建span后，将其进行关闭。
命中不可超过 50个字符，超过后会被截取。
*


# @NewSpan注解处理
SleuthAdvisorConfig

在类上注解只是pointcut，但是并不会真正增加一个span
