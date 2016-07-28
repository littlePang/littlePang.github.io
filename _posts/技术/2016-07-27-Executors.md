---
layout: post
title: Java Executors类
category: 学习
tags: study
keywords:
description:
---

### 线程池方法:
* newCachedThreadPool() 最小空置线程数为0, 最大线程数为`Integer.MAX_VALUE`
* newFixedThreadPool()  最小空置线程数和最大线程数都为参数 `nThread`
* newSingleThreadExecutor() 最小空置线程数和最大线程数都为 1 的线程池
