---
layout: post
title: spring中@Cacheable注解使用
category: 技术
tags: Spring
keywords:
description:
---

## @Cachebale
  看名字还蛮好猜这个注解是用来做什么的,这个注解是用来做缓存的,下面是Cacheable注解的注释,
大概意思就是说,这个注解用来标识一个方法或者一个类的的所有方法的返回值都是可缓存的,
在调用被这个注解标识的方法时,会先判断,在对应的缓存中是否存在,若存在这直接返回,否则就执行
方法,并将结果放入缓存.

    /**
     * Annotation indicating that the result of invoking a method (or all methods
     * in a class) can be cached.
     *
     * <p>Each time an advised method is invoked, caching behavior will be applied,
     * checking whether the method has been already invoked for the given arguments. A
     * sensible default simply uses the method parameters to compute the key, but a SpEL
     * expression can be provided via the {@link #key} attribute, or a custom
     * {@link org.springframework.cache.interceptor.KeyGenerator KeyGenerator} implementation
     * can replace the default one (see {@link #keyGenerator}).
     *
     * <p>If no value is found in the cache for the computed key, the method is invoked
     * and the returned value is used as the cache value.
     */


下面是注解中的属性配置:

    String[] value() default {}; // 缓存存放的地方(实现了org.springframework.cache.Cache接口的bean名称)
    String[] cacheNames() default {}; // 同value
    String key() default ""; // 缓存的key值,默认是使用所有方法参数作为key
    String keyGenerator() default ""; // key生成器(实现了org.springframework.cache.interceptor.KeyGenerator接口的bean名称)
    String cacheManager() default ""; // 缓存管理器(实现了org.springframework.cache.CacheManager接口的bean名称)
    String cacheResolver() default ""; // 换成转换(实现了org.springframework.cache.interceptor.CacheResolver接口的bean名称)
    String condition() default ""; // 缓存条件,满足条件则缓存,支持Spring Expression Language (SpEL)
    String unless() default ""; // 缓存条件,满足条件则不缓存,支持Spring Expression Language (SpEL)

## 注解使用

### 配置
在spring配置文件中加入如下内容,其中SimpleCacheManager是spring提供的一个缓存管理的简单实现, simpleCacheImpl 是我自己使用hashMap自己写的一个org.springframework.cache.Cache的实现(这个缓存可以替换成redis,mencache等).

    <!-- 启动缓存注解功能 -->
    <cache:annotation-driven/>
    <bean name="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
        <property name="caches">
            <set>
                <ref bean="simpleCacheImpl" />
            </set>
        </property>
    </bean>


### 在方法上使用
##### 最简单的使用,只需配置所使用的缓存容器即可.
    @Cacheable(cacheNames = "simpleCacheImpl")
    public String load(String key) {
        logger.info("load for key {}", key);
        return String.format("value for key [%s]", key);
    }

##### 测试
    @Test
    public void loadTest() {
        String value = springCacheAnnotation.load("little");
        logger.info("little value is {}", value);

        String cacheValue = springCacheAnnotation.load("little");
        logger.info("little value is {}", cacheValue);
    }

##### 结果
可以看到只有第一次调用的时候才真正执行了方法,第二次调用时,直接返回了缓存中的值.

    21:13:20.605 [main] INFO  c.l.p.b.c.s.SpringCacheAnnotation - load for key little
    21:13:20.609 [main] INFO  serivce.SpringCacheAnnotationTest - little value is value for key [little]
    21:13:20.610 [main] INFO  serivce.SpringCacheAnnotationTest - little value is value for key [little]

##### 注意
测试发现spring中是使用cache中的`public ValueWrapper get(Object o)`这个方法来判断是否需要执行实际的方法调用的,如果这个方法返回null,则目标方法会被执行一次.

### 完整的测试小项目在 [boring_code_world仓库中](https://github.com/littlePang/boring_code_world/tree/cacheable_annotation_test)
