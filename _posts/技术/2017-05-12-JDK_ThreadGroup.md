---
layout: post
title: JDK ThreadGroup简述
category: 技术
tags: jdk
keywords:
description:
---

## 线程组

线程组表示的是一组线程的集合。此外，一个线程组也可以包含其他的线程组，所有的线程组组成一棵树型结构，除了最初的那个系统线程组，其他线程组都有一个父线程组。

一个线程允许去访问它所在的线程组的相关信息，但是不允许访问它所在的线程组的父线程组或者其他线程组的信息。这里所指的无法获取其他线程组的信息，应该是指 在当前线程组中没有其他线程组的相关信息，而最开始理解的是，在一个线程中无法强行拿到另外一个线程组的信息，会抛出异常。但其实下面的代码是可以正常执行的。

                  final ThreadGroup tg1 = new ThreadGroup("thread_group_one");
                  final ThreadGroup tg2 = new ThreadGroup("thread_group_two");

                  final Thread t1 = new Thread(tg1, new Runnable() {
                      public void run() {
                          try {
                              logger.info("线程一开始执行");
                              TimeUnit.SECONDS.sleep(20);
                              logger.info("线程一执行完毕");
                          } catch (InterruptedException e) {
                              logger.error("线程 one 被中断");
                          }
                      }
                  });


                  Thread t2 = new Thread(tg2, new Runnable() {
                      public void run() {
                          try {
                              TimeUnit.SECONDS.sleep(5);
                              logger.info("线程二开始执行");
                              ThreadGroup t1tg = t1.getThreadGroup();
                              logger.info("t1gq的相关信息 {}", t1tg.activeGroupCount());
                              logger.info("t1gq的相关信息 {}", t1tg.getName());
                              logger.info("t1gq的相关信息 {}", t1tg.getParent());
                              t1tg.interrupt();


                              logger.info("线程二执行中断线程一结束");
                          } catch (Exception e) {
                              e.printStackTrace();
                          }
                      }
                  });

                  t1.start();
                  t2.start();

                  TimeUnit.SECONDS.sleep(30);

## 线程组和线程池

两者的目的不太一样，线程组是为了更好的管理线程。而线程池是为了复用线程，减少线程创建和销毁的开销。


## 线程组的使用

以下简单列了下使用方式，具体其他方法的使用，可查阅文档或源码进行查看

        // 中断线程组中的所有线程
        threadGroup.interrupt();

        // 获取线程组以及其所有子线程组的活动线程数的预估值
        threadGroup.activeCount();

        // 设置为守护线程
        threadGroup.setDaemon(true);

        // 获取线程组中的所有线程（也可同时获取所有子线程组的所有线程）
        threadGroup.enumerate();
