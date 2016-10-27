  ---
layout: post
title: AbstraceQueuedSynchronizer源码阅读
category: 学习
tags: java concurrent
keywords:
description:
---

AQS使用的等待队列是一个"CLH"(Craig, Landin, and Hagersten)锁队列的变形. CLH通常在自旋锁中使用,但在AQS中,在阻塞同步器时也使用了相同的基本策略来持有线程在它所在节点的前一个节点中的控制信息.在每个节点中使用"status"域来检测一个线程是否应该被阻塞.节点释放锁时会唤醒它的后一个节点.队列中的每一个节点都会被指定唤醒."status"域不控制线程是否获取锁,

CLH队列中要求插入时在tail(链表尾节点)上是一个原子操作.
