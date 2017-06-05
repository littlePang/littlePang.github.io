---
layout: post
title: 理解G1垃圾收集器的GC日志
category: 技术
tags: JVM
keywords:
description:
---

这篇文章的目的是解释以下 G1 的 GC 日志，用于在做跟踪诊断的时候使用，我们会看下在 开启 -XX:+PrintGCDetails 选项时的GC详细信息。以及开启 -XX:UnlockDiagnosticVMOptions 和 G1PrintRegionLivenessInfo 选项，这俩选项在每次标记循环借宿时会打印出使用率和每个region区域存活对象使用的空间大小，G1PrintHeapRegions 支持输出堆中所有region区域的内存分配和回收的详细信息。

# -XX:+PrintGCDetails 选项输出信息说明

以下GC日志是在 JDK 1.7.0_04 下使用 -XX:+PrintGCDetails 选项输出，下面是一个日子样例：

        0.522: [GC pause (young), 0.15877971 secs]
           [Parallel Time: 157.1 ms]
              [GC Worker Start (ms):  522.1  522.2  522.2  522.2
               Avg: 522.2, Min: 522.1, Max: 522.2, Diff:   0.1]
              [Ext Root Scanning (ms):  1.6  1.5  1.6  1.9
               Avg:   1.7, Min:   1.5, Max:   1.9, Diff:   0.4]
              [Update RS (ms):  38.7  38.8  50.6  37.3
               Avg:  41.3, Min:  37.3, Max:  50.6, Diff:  13.3]
                 [Processed Buffers : 2 2 3 2
                  Sum: 9, Avg: 2, Min: 2, Max: 3, Diff: 1]
              [Scan RS (ms):  9.9  9.7  0.0  9.7
               Avg:   7.3, Min:   0.0, Max:   9.9, Diff:   9.9]
              [Object Copy (ms):  106.7  106.8  104.6  107.9
               Avg: 106.5, Min: 104.6, Max: 107.9, Diff:   3.3]
              [Termination (ms):  0.0  0.0  0.0  0.0
               Avg:   0.0, Min:   0.0, Max:   0.0, Diff:   0.0]
                 [Termination Attempts : 1 4 4 6
                  Sum: 15, Avg: 3, Min: 1, Max: 6, Diff: 5]
              [GC Worker End (ms):  679.1  679.1  679.1  679.1
               Avg: 679.1, Min: 679.1, Max: 679.1, Diff:   0.1]
              [GC Worker (ms):  156.9  157.0  156.9  156.9
               Avg: 156.9, Min: 156.9, Max: 157.0, Diff:   0.1]
              [GC Worker Other (ms):  0.3  0.3  0.3  0.3
               Avg:   0.3, Min:   0.3, Max:   0.3, Diff:   0.0]
           [Clear CT:   0.1 ms]
           [Other:   1.5 ms]
              [Choose CSet:   0.0 ms]
              [Ref Proc:   0.3 ms]
              [Ref Enq:   0.0 ms]
              [Free CSet:   0.3 ms]
           [Eden: 12M(12M)->0B(10M) Survivors: 0B->2048K Heap: 13M(64M)->9739K(64M)]
         [Times: user=0.59 sys=0.02, real=0.16 secs]

这是一个 Evacuation Pause （这里理解为搬空停顿？G1收集器GC时的region区域间的复制动作？）收集的典型日志，它表示存活对象从一个regions集合(young或者 young+old)被复制到另一个集合。它是一个 stop-the-world 事件而且所有的应用线程在这段时间都停在安全点上。

这一次停顿有几个子任务组成，在日志中由缩进来表示。这次撤空停顿(Evacuation Pause)最上面一行为

`0.522: [GC pause (young), 0.15877971 secs]`

这是最高等级的信息，告诉我们这次搬空停顿是在这次处理开始后的0.522秒开始的。并且所有被搬空的region区域都是年轻代，即eden区域和survivor区域。完成这一次收集花费了 0.15877971 秒。

如果执行时选取的region区域包含了所有的年轻代region区域以及一些老年代的region区域，则搬空停顿是混合的。输出结果形式如下：

`1.730: [GC pause (mixed), 0.32714353 secs`

接下来让我们看看这次停顿中所有子任务的执行过程。

`[Parallel Time: 157.1 ms]`

并行时间是完成所有并行GC工作线程所花费的时间。这一行下面对应的行信息是这些工作线程在这段并行时间内的执行信息，这个case中花费了 151.1ms。

`[GC Worker Start (ms):  522.1  522.2  522.2  522.2
  Avg: 522.2, Min: 522.1, Max: 522.2, Diff:   0.1]`

上面的第一行告诉我们每一个工作线程启动的毫秒时间点。启动时间根据线程的id进行排序，在这次处理中，线程0在522.1毫秒启动，线程1在522.2毫秒启动，
第二行告诉我们平均时间，做小时间，最大时间，以及所有工作线程的启动时间差。

`[Ext Root Scanning (ms):  1.6  1.5  1.6  1.9
 Avg:   1.7, Min:   1.5, Max:   1.9, Diff:   0.4]`

这里给出了每个工作线程扫描根节点(gloables 全局变量，registers 程序计数器？，thread stacks线程栈 和 VM data structure虚拟机数据)所花费的时间。这里线程0花费了1.6ms，线程1花费了1.5ms 执行根节点的扫描任务，第二行清晰的展示出平均时间，最小时间，最大时间，以及所有工作线程花费时间的最大差值。

`[Update RS (ms): 38.7 38.8 50.6 37.3
  Avg: 41.3, Min: 37.3, Max: 50.6, Diff: 13.3]`

这里给出的是每个线程 更新 RS(Remembered Sets) 所花费的时间。RS 是 记录对象引用追踪堆内存空间区域的数据结构。突变线程（Mutator threads 不知该如何理解这个线程？google翻译为 突变线程，然而并不能帮我理解）会持续改变对象关系图让引用指向一个常规的region区域。我们在缓存中更新改变这些跟踪叫做更新缓存(Update Buffers)。更新 RS 的自任务处理 更新缓存 时不能同时处理和更新所有regions区域的相关记录集合。

`[Processed Buffers : 2 2 3 2
Sum: 9, Avg: 2, Min: 2, Max: 3, Diff: 1]`

这里告诉了我们每个工作线程处理的 更新缓存 （上面提到的 Update Buffers）的数量

`[Scan RS (ms): 9.9 9.7 0.0 9.7
Avg: 7.3, Min: 0.0, Max: 9.9, Diff: 9.9]`

这里是每个工作线程扫描 记录集合 所花费的时间。每个region区域的记录集合包含一些卡片（cards）表示指向这个region区域的相关引用。这个阶段会扫描这些cards找出所有指向region区域的引用。

`[Object Copy (ms): 106.7 106.8 104.6 107.9
Avg: 106.5, Min: 104.6, Max: 107.9, Diff: 3.3]`

这里是每个工作线程从垃圾收集集合的region区域复制存活对象到另一个region区域所花费的时间。

`[Termination (ms): 0.0 0.0 0.0 0.0
Avg: 0.0, Min: 0.0, Max: 0.0, Diff: 0.0]`

终止时间是每个工作线程提供终止所花费的时间。但是在终止前，它会检查其他线程的工作队列，如果在其他线程的工作队列中仍然还有对象引用，它还会尝试偷取对象引用，如果偷取成功，它会讲引用处理掉，并再次尝试终止。（工作窃取算法）

`[Termination Attempts : 1 4 4 6
Sum: 15, Avg: 3, Min: 1, Max: 6, Diff: 5]`

这里给出了每个线程终止尝试的次数。

`[GC Worker End (ms): 679.1 679.1 679.1 679.1
Avg: 679.1, Min: 679.1, Max: 679.1, Diff: 0.1]`

这里是每个工作线程停止所花费的时间。

`[GC Worker (ms): 156.9 157.0 156.9 156.9
Avg: 156.9, Min: 156.9, Max: 157.0, Diff: 0.1]`

这里是每个工作线程整个生命周期所花费的时间。

`[GC Worker Other (ms): 0.3 0.3 0.3 0.3
Avg: 0.3, Min: 0.3, Max: 0.3, Diff: 0.0]`

这里是每个工作线程花费在整个并行处理时间中 以上没有说到的其他任务所花费的时间。

`[Clear CT: 0.1 ms]`

这个是清理卡片表(Card Table)所花费的时间，这个任务是以串行模式执行的。


`[Other: 1.5 ms]`

这个时间是这一行下面所列的任务执行的时间。这些子任务是以串行的方式执行的（个别的任务可能会是并行执行的）。

`[Choose CSet: 0.0 ms]`

选取垃圾收集集合所花费的时间。

`[Ref Proc: 0.3 ms]`

处理引用对象所花费的时间。

`[Ref Enq: 0.0 ms]`

将引用放入`ReferenceQueue`队列所花费的时间。

`[Free CSet: 0.3 ms]`

释放垃圾收集集合数据结构所花费的时间。

`[Eden: 12M(12M)->0B(13M) Survivors: 0B->2048K Heap: 14M(64M)->9739K(64M)]`

这一行给出了在这次清理停顿(Evacuation Pause)中，堆大小改变的详细信息。上面的线上展示为：Eden 占用了12M 并且在垃圾收集前他的容量也是12M，在收集之后，所有东西都从Eden区域被清理或者提升之后，它的内存占用减少到了0，并且它的大小上升到了13M。新的Eden区域13M的容量在这个点是没有保留的，这个值就是Eden区域的目标大小。随着需求的增加，region区域被加入到Eden区域，当加入的region区域达到了目标大小，我们就会开始下一次的垃圾回收了。

相似的，Survivor区域在这次垃圾收集之后，内存占用从0字节上涨到了2048KB，在垃圾收集前，整个堆内存被占用了14M（堆内存的整体容量为64M），垃圾收集后，堆内存占用变为了9739K（堆内存整体容量为64M）

除了清理停顿(Evacuation Pause)之外，G1也会执行并发标记(concurrent-marking)来构建每个region区域的存活数据信息。

              1.416: [GC pause (young) (initial-mark), 0.62417980 secs]
              …....
              2.042: [GC concurrent-root-region-scan-start]
              2.067: [GC concurrent-root-region-scan-end, 0.0251507]
              2.068: [GC concurrent-mark-start]
              3.198: [GC concurrent-mark-reset-for-overflow]
              4.053: [GC concurrent-mark-end, 1.9849672 sec]
              4.055: [GC remark 4.055: [GC ref-proc, 0.0000254 secs], 0.0030184 secs]
               [Times: user=0.00 sys=0.00, real=0.00 secs]
              4.088: [GC cleanup 117M->106M(138M), 0.0015198 secs]
               [Times: user=0.00 sys=0.00, real=0.00 secs]
              4.090: [GC concurrent-cleanup-start]
              4.091: [GC concurrent-cleanup-end, 0.0002721]

标记循环的第一个阶段是初始化标记(Initial Marking),这个阶段会标记所有从根节点直接可达的对象，  这个阶段是整个年轻代清理暂停(young Evacuation Pause)的背后支持。            

`2.042: [GC concurrent-root-region-scan-start]`

初始化标记阶段，并发标记扫描region区域根节点集合直接可达的对象的开始时间

`2.067: [GC concurrent-root-region-scan-end, 0.0251507]`

并发标记根region区域扫描的结束时间点，花费的时间为 0.0251507 秒。

`2.068: [GC concurrent-mark-start]`

在 2.068 秒开始这次处理的并发标记阶段。

`3.198: [GC concurrent-mark-reset-for-overflow]`

这个是表明全局标记栈满了，出现了栈溢出，并发标记会检测这次溢出然后重新设置数据结构并再次开始标记

`4.053: [GC concurrent-mark-end, 1.9849672 sec]`

并发标记阶段的结束，持续了 1.9849672 秒。

`4.055: [GC remark 4.055: [GC ref-proc, 0.0000254 secs], 0.0030184 secs]`

这里表示的是重新标记的阶段，这是个 STW（stop-the-world）阶段。完成前一个阶段剩下的标记工作（SATB 缓存处理），在这个case中，这个阶段花费了 0.0030184 秒，其中 0.0000254 秒用于引用处理。

`4.088: [GC cleanup 117M->106M(138M), 0.0015198 secs]`

cleanup清除阶段也是一个 STW（stop-the-world）阶段。它会通过所有region区域的标记信息计算出每个region区域的存活对象信息，重新设置标记数据结构并根据region区域的GC功效(gc-efficiency)进行排序。在这个例子中，堆内存的大小一共为138M，在经过存活数据计算之后，总的存活数据大小从117M减少为106M。

`4.090: [GC concurrent-cleanup-start]`

并发清除阶段的开始时间，用来释放在上一个stop-the-world阶段找到的需要被清空的region区域（不包含任何存活对应）。

`4.091: [GC concurrent-cleanup-end, 0.0002721]`

并发清除阶段的结束时间，这个阶段花费了0.0002721秒来释放所有的空region区域。


# -XX:G1PrintRegionLivenessInfo 选项输出信息说明


## 翻译自：[https://blogs.oracle.com/poonam/understanding-g1-gc-logs](https://blogs.oracle.com/poonam/understanding-g1-gc-logs)


## 其他

G1垃圾收集器官方文档：http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

GCViewer：https://github.com/chewiebug/GCViewer

G1收集器说明：http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html
