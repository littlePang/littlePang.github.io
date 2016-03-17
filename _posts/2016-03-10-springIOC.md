---
layout: post
title: SpringIoc学习
category: 技术
tags: Spring
keywords:
description:
---

# spring IoC


以下内容都是阅读 Spring技术内幕：深入解析Spring架构与设计原理（第2版）所写的

## IoC - Inversion of Control(控制反转)

### 什么是IoC?

控制反转,也就是依赖对象的获取被反转了,将对象所依赖的对象的获取交由外部框架进行管理.

### 注入方式

* 接口注入
* setter注入
* 构造器注入

## IoC的 BeanFactory和ApplicationContext

springIoc主要接口设计
![](/assets/picture/springIocIntergface.png)

使用IoC容器在获取对象时,可以使用转义符"&"来获取一个bean的FactoryBean本身.
FactoryBean是某个Bean的创建工厂,只能创建这种bean的工厂,类似于工厂模式.

## DefaultListableBeanFactory
