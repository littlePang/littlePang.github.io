---
layout: post
title: guavaCache学习
category: 技术
tags: guava
keywords:
description:
---


# guava cache

## 个人yy

guavaCache 如何实现的缓存? 内部持有一个Map?

如何实现 定时刷新 ? 有定时任务定时清理缓存中的对象?



segment的作用

weight 的作用
如果设置了,权重,那么它会影响初始化的大小,如果有个权重为5的对象,那它就相当于有5个权重为1的对象

    int initialCapacity = Math.min(builder.getInitialCapacity(), MAXIMUM_CAPACITY);
    if (evictsBySize() && !customWeigher()) {
      initialCapacity = Math.min(initialCapacity, (int) maxWeight);
    }

concurrencyLevel 影响分段个数

    int segmentShift = 0;
    int segmentCount = 1;
    while (segmentCount < concurrencyLevel
           && (!evictsBySize() || segmentCount * 20 <= maxWeight)) {
      ++segmentShift;
      segmentCount <<= 1;
    }
    this.segmentShift = 32 - segmentShift;
    segmentMask = segmentCount - 1;

    this.segments = newSegmentArray(segmentCount);

    int segmentCapacity = initialCapacity / segmentCount;
    if (segmentCapacity * segmentCount < initialCapacity) {
      ++segmentCapacity;
    }

    int segmentSize = 1;
    while (segmentSize < segmentCapacity) {
      segmentSize <<= 1;
    }


tricker : 计时器

## 网上资料

网上关于guavaCache的使用的文章蛮多,但是讲如何实现的并不多,关于guava cache如何做的,
找到这篇好文 [Java Cache系列之Guava Cache实现详解](http://www.blogjava.net/DLevin/archive/2013/10/20/404847.html)
,这篇文章介绍了,guava cache 的数据结构,接口设计,以及代码实现逻辑,讲的挺好的(此处应竖起大拇指)

## 源码学习

看完了上面那篇文章后,对Guava Cache有了一定的了解,下面就到了自己看源码,解答自己的疑问,梳理实现逻辑的时候了.

### 初次见面

既然Guava Cache 是由 CacheBuilder来配置以及实例化的,那自然要先看CacheBuilder咯.

进入 CacheBuilder的源码,首先进入视野的就是CacheBuilder类上的一大片注释说明:

    /**
    * <p>A builder of {@link LoadingCache} and {@link Cache} instances having any combination of the
    * following features:
    *
    * <ul>
    * <li>automatic loading of entries into the cache
    * <li>least-recently-used eviction when a maximum size is exceeded
    * <li>time-based expiration of entries, measured since last access or last write
    * <li>keys automatically wrapped in {@linkplain WeakReference weak} references
    * <li>values automatically wrapped in {@linkplain WeakReference weak} or
    *     {@linkplain SoftReference soft} references
    * <li>notification of evicted (or otherwise removed) entries
    * <li>accumulation of cache access statistics
    * </ul>
    *
    ...
把说明都看了一遍, 主要是介绍了这个Builder构建的Cache含有的
特性,简单的使用例子,以及特性使用说明.

### 初步认识

看完注释,往下面走,可以看到CacheBuilder里面
定义的一些属性以及它的默认值(初始化大小,并发等级,计数器,对象移除监听器等)

    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    private static final int DEFAULT_CONCURRENCY_LEVEL = 4;
    private static final int DEFAULT_EXPIRATION_NANOS = 0;
    private static final int DEFAULT_REFRESH_NANOS = 0;
    ...
    int initialCapacity = UNSET_INT;
    int concurrencyLevel = UNSET_INT;
    long maximumSize = UNSET_INT;
    long maximumWeight = UNSET_INT;
    Weigher<? super K, ? super V> weigher;
    Strength keyStrength;
    Strength valueStrength;
    long expireAfterWriteNanos = UNSET_INT;
    long expireAfterAccessNanos = UNSET_INT;
    long refreshNanos = UNSET_INT;
    Equivalence<Object> keyEquivalence;
    Equivalence<Object> valueEquivalence;
    ...



再往下走,可以看到是一些设置Cache特性的一些方法(设置大小,value对象比较器,key比较器,并发等级,权重设置器,
引用类型(软引用,弱引用,强引用), 刷新时间, 过期时间, 对象移除监听器等), 这里需要注意一点的是,所有的这些
属性设置方法中,都有对属性是否被设置过的判断,如果以及被设置过了,再次设置的时候,会抛出异常,也就是说,
这些属性是不能被重复设置的.

    public CacheBuilder<K, V> initialCapacity(int initialCapacity)
    public CacheBuilder<K, V> concurrencyLevel(int concurrencyLevel)
    public CacheBuilder<K, V> weakKeys()
    public CacheBuilder<K, V> expireAfterWrite(long duration, TimeUnit unit)
    public CacheBuilder<K, V> refreshAfterWrite(long duration, TimeUnit unit)
    ...

drainReferenceQueues() 中为何最多排除 16个entry
