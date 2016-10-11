---
layout: post
title: Collection.sort实现
category: 学习
tags: JDK
keywords:
description:
---

首先说下,我看的是版本是JDK 1.7.0_71的实现,因为各个版本的实现可能会不一样,所以先声明下版本.

Collection.sort()的代码如下:

        public static <T extends Comparable<? super T>> void sort(List<T> list) {
                Object[] a = list.toArray();
                Arrays.sort(a);
                ListIterator<T> i = list.listIterator();
                for (int j=0; j<a.length; j++) {
                    i.next();
                    i.set((T)a[j]);
                }
            }

底层实际的排序是交由Array.sort()来做的.

        public static void sort(Object[] a) {
                if (LegacyMergeSort.userRequested)
                    legacyMergeSort(a);
                else
                    ComparableTimSort.sort(a);
            }

`LegacyMergeSort.userRequested`这个的意义是说,是否使用之前版本所使用简单归并排序对数组进行
排序,可通过在启动参数中使用`java.util.Arrays.useLegacyMergeSort`来指定.默认是使用新的ComparableTimSort.sort()来进行排序的.

接下来就来看下 ComparableTimSort.sort() 的实现

看了代码后发现, ComparableTimSort.sort() 的实现后,发现它不是简单的只使用了某一种排序算法,而是混合使用了.

在数组元素个数不超过 32 个的时候,是使用 折半插入排序的.

超过32个元素后,则使用 TimSort排序算法,进行排序, TimSort是一个对归并排序做了大量优化的版本.
TimSort是为了减少对升序部分的回溯和对降序部分的性能倒退, 按照降序或者升序对待排序数组做了分区,
将输入当中已经有序的段,作为一个分区.那么就节省了在这个分区进行回溯的时间了.比如[1,2,3,0]分区后就变成了[[1,2,3],[0]]两个分区了.而对于降序部分,则将其直接反转作为一个分区,例如[2,1,0,1],分区后变为[[0,1,2],[1]]两个分区.最后只需对所有的分区进行归并排序,则可得到最终排序完成的序列.

## 两个疑问:
TimSort排序分区的minRunLength,最小分区单元的大小为什么要那样设置?
分区的长度保证,下一个分区是上一个分区的2倍,不然就merge两个分区.
