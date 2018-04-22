---
layout: post
title: jdk8的ConcurrentHashMap疑惑总结
category: 技术
tags: java
keywords:
description:
---

# 前言

JDK8之前的ConcurrentHashMap都是使用分段锁来提高并发处理能力的，而在JDK8中ConcurrentHashMap做了比较大的变化，已经不再使用分段锁来处理了，
在JDK8中，ConcurrentHashMap 和 HashMap 的数据结构是一样的（数组+链表+红黑树）

在扩容时，每个链表(或者树)都会变成，最终的结果只有两种情况，要么继续在 第 i 个位置，要么在 i+n 的位置，因为数组的长度永远的2的n次方,在初始化时就已经处理了
