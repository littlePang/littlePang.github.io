---
layout: post
title: effective java 读书笔记
category: 技术
tags: java
keywords:
description:
---

# 创建和销毁兑现

## 考虑用静态工厂方法代替构造器

使用静态工厂方法，而不是构造器的优点：
* 静态工厂方法有名称，可以更加容易的描述返回对象，使客户端代码更易于阅读
* 静态工厂方法不必每次都创建一个新对象，可以每次都返回预先构建好的实例（Integer.valueOf()默认在入参是 -128 到 127 之间的值，则会返回缓存对象，而不是每次都实例化一个新的Integer）
* 静态方法可以返回类型的任何子类的对象，使得返回对象的类具有更大的灵活性（java.util.Collections 就是各种集合类构建的静态类，使得客户端实例化某种集合类更加方便简单）
* 静态工厂方法使用在创建参数化类型（泛型）实例的时候，代码更加简洁（在JDK7之后，已经有了构造器的类型推导，比之前也会简洁一点了）

使用静态方法的缺点：
* 类如果不含有公有的或者受保护的构造器，就不能被子类化。（但其实也满足Java鼓励使用复合而不是继承的思想）
* 静态工厂方法与其他的静态方法实际上没有区别，不像构造器在new的时候可以看到有哪些构造器方法。所以想要查明如何实例化一个类，会稍微困难一点

## 遇到多个构造器参数时要考虑用构建器

例如如下这种方式。

1. 在属性很多的情况下，构造器的参数将变得非常多，降低代码的易读性。

2. 而如果使用默认无参构造器先实例化对象，然后再将属性逐个set的方式，则由于构造过程被分到了几个调用中，那么 JavaBean 在构造过程中，可能处于不一致的状态

        public class User {
          private String name;
          private String password;

          public User() {
          }

          public User(String name, String password) {
            this.name = name;
            this.password = password;
          }

          // getter and setter
        }

可将以上代码改造成如下的方式，使用Builder构建器，并且可以将User的属性设置为final，使其成为不可变类，从而避免构建过程中处于不一致状态。

        public class User {
          private String name;
          private String password;

          public User(String name, String password) {
            this.name = name;
            this.password = password;
          }

          // getter and setter

          public static class Builder {
              private String name;
              private String password;

              public Builder setName(String name) {
                this.name = name;
                return this;
              }

              public Builder setPassword(String password) {
                this.password = password;
                return this;
              }

              public User build() {
                return new User(name, password);
              }
          }
        }

## 用私有构造器或者枚举类型强化singleton属性

## 通过私有构造器强化不可实例化的能力

## 避免穿甲不必要的对象

## 清楚过期的对象应用

## 避免使用终结方法
