---
layout: post
title: spring framword 数据访问
category: 技术
tags: spring
keywords:
description:
---

# spring framwork 事务管理简介

使用spring的综合事务管理有许多令人信服的理由。Spring Framwork 提供了用来进行事务管理一致的抽象，并提供了下面一些特性：

* 访问不同事务（例如JTA（Java Transaction API）、JDBC、Hibernate、JPA（Java Persistent API）、JDO（Java Data Objects）），统一的编程模型。
* 支持[申明式事务管理](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-declarative)。
* 比 JTA 等复杂事务API更加简单的[编程式](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-programmatic)事务管理。
* 和spring的数据访问抽象杰出的整合。

下面的是Spring Framework 对事务的增值和其使用的技术（也包括最佳事件的讨论，应用服务的整合，和常见问题的解决方案）。

* [Spring事务支持模型的优势](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-motivation) 描述的是为什么你应该使用Spring的事务管理抽象 来代替 EJB 容器管理事务（CMT）或者 使用像Hibernate这种专属的事务管理 的原因。
* [理解Spring的事务管理抽象](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-strategies),是对核心类的概述，以及在多个数据源时如何配置 `DataSource`实例
* [使用事务进行数据同步](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#tx-resource-synchronization),描述应用程序代码如何正确的进行资源创建，复用，以及清理。
* [申明式事务管理](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-declarative)，描述对声明式事务管理的支持。
* [编程式事务管理](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-programmatic)，编程式（指显式代码编程）事务管理支持。
* [事务绑定事件](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-event)，描述如何在事务中使用应用事件。

# Spring事务支持模型的优势

巴拉巴拉，总结一句就是，Spring框架的事务管理很牛逼。

# 理解Spring的事务管理抽象
Spring事务抽象抽象的关键是一个叫 _事务策略(transaction strategy)_ 的抽象。事务策略的抽象由`org.springframework.transaction.PlatformTransactionManager`来进行定义。

        public interface PlatformTransactionManager {

            TransactionStatus getTransaction(
                    TransactionDefinition definition) throws TransactionException;

            void commit(TransactionStatus status) throws TransactionException;

            void rollback(TransactionStatus status) throws TransactionException;
        }


尽管它可以在应用程序中通过编程式的来使用，但它是一个主要的服务提供接口(SPI)。由于 `PlatformTransactionManager`是一个接口，所以它在必要时可以轻易的被mock或者stubbed（分段处理？）。它没有强行绑定查询策略（像JNDI一样）。`PlatformTransactionManager`的实现在spring容器中就和其他的对象(bean)定义类似。这个单独的好处使得Spring的事务管理即使在使用JTA的时候也会是一个有价值的抽象。事务代码比起直接使用JTA可以更加简单的进行测试。

这里再次保持了Spring的哲学思想。`TransactionException`可以由 `PlatformTransactionManager`接口的任何一个方法抛出（这个异常继承自 `java.lang.RuntimeException`），事务基础设施失败几乎是致命的。只有在罕见的一些案例中，应用代码可以在事务失败时进行真正的恢复处理，这时可以通过捕获`TransactionException`进行处理。需要说明的是，开发者并不是强制要求处理这个异常的。

`getTransaction(..)`方法依赖于`TransactionDefinition`参数返回一个`TransactionStatus`对象，这个返回的`TransactionStatus`可能代表的是一个新的事务，如果在当前调用栈中存在一个匹配的事务，则也可能是一个以及存在的事务。后一种情况的含义是说，在Java EE的事务上下文中，`TransactionStatus`和一个 线程的执行 相关联。

`TransactionDefinition`接口定义：

* Isolation(隔离级别)，当前事务和其他事务的隔离程度。例如，这个事务是否能看到其他事务未提交的写操作数据。
* Propagation(传播性)，基本上，在一个事务中的所有代码都在那个事务中执行。但是，在一个事务上下文已经存在的时候，你也可以指定某一个事务方法的处理行为。例如，代码在这个已经存在的事务中继续执行（大部分情况下），或者将已存在的事务睡眠并创建一个新的事务。spring提供了EJB CMT中所有事务传播相似的特性。可在[这里](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#tx-propagation)查看spring事务处理传播性的语义。
* Timeout(超时时间)，在一个基础事务下，一个事务可以用来在超时和自动回滚前执行事务的时长。
* Read-only status(只读状态)，当你的代码只读取数据，不修改数据时，可以将事务指定为只读事务。只读事务在一些case中（例如使用Hibernate时）是非常有用的一个优化。

这些设置反应的是一个标准事务的概念。如果必要，可以参考事务隔离级别和其他的一些事务核心概念的讨论。要使用Spring或者任何的事务管理解决方案，理解这些概念都是非常必要的。

`TransactionStatus`接口提供已一个简单的方式用来控制事务的执行和查询事务的状态。这些概念都是非常相似的，在所有的事务API中都是公共的一部分：

        public interface TransactionStatus extends SavepointManager {

            boolean isNewTransaction();

            boolean hasSavepoint();

            void setRollbackOnly();

            boolean isRollbackOnly();

            void flush();

            boolean isCompleted();

        }
