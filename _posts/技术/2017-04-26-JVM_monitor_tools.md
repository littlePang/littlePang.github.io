---
layout: post
title: JVM 性能监控与故障处理工具
category: 技术
tags: jvm
keywords:
description:
---

## jps（JVM Process Status Tool）
显示指定系统内所有的HotSpot虚拟机进程。_注意只能查看当前用户有访问权限的进程_ , 由于目前线上机都是在tomcat这个用户下运行的，所以使用root或者自己的账户登陆时，无法查看，so，这个工具暂时没有用。

## jstat（JVM Statistics Monitoring Tool）
用于收集HotSpot虚拟机各方面的运行数据。和上面一个命令一样它也只能访问当前用户下的JVM进程。

## jinfo (Configuration Info for Java)
显示虚拟机的配置信息,这个命令总算能用了，也是不容易，可通过root权限查看其他账户下的JVM进程的配置信息。（首先要先拿到想查看的JVM进程的PID），不过下面的 -flag[+|-] 或者 -flag name=value 在线上环境是不行的（权限问题）。

            不带参数（eg：`sudo jinfo 23590`）
             打印系统属性和命令设置的属性。

          -flag name
             打印出命令行设置的属性中name的属性值

          -flag [+|-]name
             使一个命令行标记有效或无效化。

          -flag name=value
             设置一个命令行标志

          -flags
             打印出所有的命令行标志

          -sysprops
             打印系统属性

## jmap （Memory Map for Java）
生成虚拟机的内存转储快照（heapdump文件）


## jhat （JVM Heap Dump Browser）
用于分析headdump文件，它会建立一个http／html服务器，让用户可以在浏览器上查看分析结果。

## jstack （Stack Thace for Java）
显示虚拟机的线程快照

## jconsole Java监控与管理控制台

## VisualVM 多合一故障处理工具
