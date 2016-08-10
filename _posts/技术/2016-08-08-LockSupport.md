---
layout: post
title: Java LockSupport类
category: 学习
tags: study
keywords:
description:
---

为创建锁和其他同步类提供线程同步原语.

调用`park()`时如果许可可用,则直接返回,并消费掉这个许可证,其他情况可能会阻塞.

调用`unpark()`时如果许可证不可用,则可以使许可证可用(这个许可证和信号量不一样,是不可累加的,最多只有一个许可是可用的.)

`park()`和`unpark()`比起已经过时的方法`Thread.suspend()`和`Thead.resume`的好处是,不用考虑阻塞任务线程和唤醒任务线程的执行顺序,即:无论`park()`和`unpark()`的执行顺序如何,都能保证阻塞任务线程被正常执行.

下面三种情况下`park()`方法会返回.
* 其他线程对当前线程执行了`unpark()`
* 其他线程中断了当前线程.
* 没有任何原因的情况下,在任何时间返回.

上面的第三种情况在[博客](http://blog.csdn.net/hengyunabc/article/details/28126139)中有说,是因为HotSpot中,对等待条件并没有使用循环判断,所以导致`park()`有可能在任意时间返回.基于这种情况,所以建议在调用`park()`外层,使用循环重复检查条件是否满足.类似于:

        while (condition) {
                 LockSupport.park(this);
                 if (Thread.interrupted()) {
                   // do something
                   return;
                  }
        }
        // do something

### 源码查看

类的开头就是下面这段代码, 类加载时,先调用了`unsafe.objectFieldOffset()`获取`Thread`类中`parkBlocker`属性的内存偏移量.
然后在设置`Thread`实例的`parkBlocker`属性是使用的是`unsafe.putObject()`.

`parkBlocker`是用来表示当前等待获取许可证的线程信息的.可以在调用`park()`时,同时设置它的值,然后在
`unpark()`线程中调用`LockSupport.getBlocker()`来获取等待线程信息.

看到这里我就想吐嘈,为啥不直接给`Thread`的`parkBlocker`属性添加getter和setter来获取和设置对应的值,
而是要使用Unsafe来做这个事情, 然后就自己YY了下,应该是对于`Thread`的使用方来说,`parkBlocker`并不应该
被直接通过`Thread`获取, `parkBlocker`就只是在`LockSupport`中被设值,所以就应该只能通过`LockSupport`来
进行获取(当然你强行要用反射来拿,我也木有办法).

      // Hotspot implementation via intrinsics API
          private static final Unsafe unsafe = Unsafe.getUnsafe();
          private static final long parkBlockerOffset;

          static {
              try {
                  parkBlockerOffset = unsafe.objectFieldOffset
                      (java.lang.Thread.class.getDeclaredField("parkBlocker"));
              } catch (Exception ex) { throw new Error(ex); }
          }

          private static void setBlocker(Thread t, Object arg) {
              // Even though volatile, hotspot doesn't need a write barrier here.
              unsafe.putObject(t, parkBlockerOffset, arg);
          }



代码的其他部分就比较简单了,就是直接调用了`Unsafe.park()`和`Unsafe.unpark()`来处理的.

上面提到的那篇[博客](http://blog.csdn.net/hengyunabc/article/details/28126139)中,有对HotSpot源码的实现分析,
然而当我看到那些C++代码时,我的表情是这样的:

![](/assets/picture/2016-08-10_yiLianMenBi.png)
