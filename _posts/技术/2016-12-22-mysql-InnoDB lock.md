---
layout: post
title: mysql InnoDB lock 类型
category: 技术
tags: mysql
keywords:
description:
---

# 前言

近期在查一些线上问题的时候,发现了一些数据库死锁的case, 所以想填充下我脑子里关于这块的知识. 由于mysql里最常用的还是InnoDB引擎,所以先来了解下InnoDB里面锁的类型吧.

# 锁的分类

参考: http://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html

## Shared and Exclusive Locks

InnoDB中 Shared Lock 和 Exclusive Lock 都是 行级 锁.

 * Shared Lock (共享锁) 即 S 锁, 获取一行数据的 S 锁后,允许在行上进行读操作.

 * Exclusive Lock (排他锁) 即 X 锁, 获取一行数据的 X 锁后,允许在行上进行 更新或删除 操作.

如果 事务t1 在行r上持有了 S 锁, 则其他事务在 行r 上获取锁时,有如下规则:

 * 如果事务t2 在r行上要求一个S锁,则可立即获得,并且记录 事务t1 和 事务t2 在行r上持有了 S锁.

 * 如果事务t2 在r行上要求一个X锁, 则不可立即获得, 需等待行r上的S锁释放之后,才可获得.

如果 事务t1 在r行上持有了一个X锁, 则其他事务想获取任何类型的锁,都不能立即获得,需要等待事务t1释放了X锁后,其他事务才可获取锁.

 ## Intention Lock
意图锁,表明事务可能会在之后获取S锁或者X锁,这是一种多粒度锁,即允许同时锁住多行(如果锁住所有行,也就相当于表锁了),它表明需要S锁或者X锁的事务稍后会在表上执行.这种锁在InnoDB中有两种类型:

* Intention shared(IS): 事务T想要在表的几行上获取S锁
* Intention exclusive(IX): 事务T想要在这些行上获取X锁

例如: `select ... lock in share mode` 会设置一个IS锁, `select ... for update`会设置一个IX锁.

intention lock遵循如下协议:

* 一个事务在表上获取一行的S锁之前,首先必须获取IS锁或者其他更强的锁(例如X锁)
* 一个事务在获取X锁之前,首先必须获取表上的IX锁.

下表可以方便的总结出上面这几个类型的锁之前的兼容关系

----
| | X锁 | IX锁 | S锁 | IS锁|
| ---| ---| ---| ---|
| X锁 | 冲突 |  冲突 | 冲突 | 冲突|
|IX锁|冲突|兼容|冲突|兼容|
|S锁|冲突|冲突|兼容|兼容|
|IS锁|冲突|兼容|兼容|兼容|
----
一个事务在需要获取的锁,如果与已经存在的锁是兼容的,则可以立即获得锁.如果和已经存在的锁是冲突的,事务必须等待存在的冲突锁全部释放之后,才能获取锁. 还有一种情况是, 如果是由于发生了死锁导致的无法获取锁,则会报错.

因此, intention lock 除了全表的请求(例如`lock table ... write`)外,不会阻塞任何事儿. IX锁和IS锁最主要的目的是表明某个事务正在锁住一行,或者想要锁住一行.

## Record Locks
record lock 是索引记录上的锁. 例如`select c1 from t where c1=10 for update`, 这个锁的目的是防止其他事务在 t.c1的值等于10的行上,进行插入,更新,或者删除操作.

record lock只锁索引记录,即使表没有定义索引, 例如: InnoDB 会创建一个 隐藏的聚集索引(clustered index),并在这个索引上使用record lock. (关于聚集索引的说明请看这里: [传送门](/2016/12/23/mysql-Clustered-Index.html))
