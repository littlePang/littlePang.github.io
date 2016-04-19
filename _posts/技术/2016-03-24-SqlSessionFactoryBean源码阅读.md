---
layout: post
title: SqlSessionFactoryBean阅读
category: 技术
tags: mybatis
keywords:
description:
---

## sqlSessionFactoryBean配置的参数

      <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
              <property name="dataSource" ref="dataSource"/>
              <property name="configLocation" value="mybatis.xml" />
              <property name="mapperLocations" value="classpath*:mapper/*"/>
      </bean>

然而在源码中只看到这样一个set方法:

    /**
     * Set the location of the MyBatis {@code SqlSessionFactory} config file. A typical value is
     * "WEB-INF/mybatis-configuration.xml".
     */
    public void setConfigLocation(Resource configLocation) {
      this.configLocation = configLocation;
    }

__所以spring在调用setter方法的时候,会判断参数类型,并且自动创建对应类型的对象进行赋值?__
