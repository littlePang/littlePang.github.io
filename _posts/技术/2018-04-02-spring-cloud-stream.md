---
layout: post
title: spring-cloud-stream使用
category: 技术
tags: spring-cloud
keywords:
description:
---

# 简介

官方文档：https://docs.spring.io/spring-cloud-stream/docs/Ditmars.SR3/reference/htmlsingle/

spring cloud stream 是一个用来构建消息驱动微服务的框架。

# 主要概念
* spring cloud stream 应用模型
* 绑定器抽象（The Binder abstraction）
* 发布-订阅 持久化支持
* 消费组支持
* 插件式绑定API

## Application Model 应用模型

spring cloud stream 应用由一系列中间件组成，和外部世界的通讯，又spring cloud stream 注入的 input 和 output 管道（channels）来进行。

![](/assets/picture/2018-04-02_1.png)

## 绑定器抽象

spring cloud stream 提供 kafka 和 rabbit mq的绑定器实现。同时也提供 `TestSupportBinder`来进行测试。绑定器抽象使得应用和中间件的连接更加灵活。例如可以在运行时动态选择目标管道（kafka主题或者rabbitmq的exchanges）进行连接

## 发布订阅持久化支持

![](/assets/picture/2018-04-02_2.png)

数据通过http发布到 raw-sensor-data，两个不同的应用 Averages 和 Ingest HDFS 都可以从 raw-sensor-data 订阅到数据，Averages用来做一些计算操作，Ingest HDFS 将数据持久化到 HDFS 上。

## 消费组

![](/assets/picture/2018-04-02_3.png)

可以通过应用分组，将多个应用

spring.cloud.stream.rabbit 使用的地方 RabbitExtendedBindingProperties

# rabbitmqctl 命令详细

https://www.rabbitmq.com/rabbitmqctl.8.html

bindings中的 destination 对应的group 如果不设置则会将其声明为匿名队列，再重启是会删除绑定关系，导致数据丢失

详情：RabbitExchangeQueueProvisioner.provisionConsumerDestination

尝试rabbitmq使用：http://tryrabbitmq.com/
