---
layout: post
title: spring boot 使用简介
category: 技术
tags: spring
keywords:
description:
---

# 介绍

Spring Boot 用来轻松的创建一个基于Spring的独立并且生产级别的应用。使得配置最小化。

## 特性
* 创建一个独立的spring应用
* 内嵌tomcat，Jetty 或者 Undertow directly
* 最简化maven配置
* 尽可能的自动化配置
* 支持生产级别的支持，例如 helth check 和 扩展配置
* 绝对没有代码生成，也不需要xml配置

# 入门使用
新建一个maven工程，在pom.xml文件中加入如下内容

          <parent>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-parent</artifactId>
              <version>1.5.3.RELEASE</version>
          </parent>

          <dependencies>
              <dependency>
                  <groupId>org.springframework.boot</groupId>
                  <artifactId>spring-boot-starter-web</artifactId>
              </dependency>
          </dependencies>

其中parent配置是spring推荐的是spring-boot的方式，如果项目中已经依赖了其他的parent配置，则可使用 [这里](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#using-boot-maven-without-a-parent) 的配置来使用spring-boot。

然后再新写一个如下Controller：


        @Controller
        @EnableAutoConfiguration
        public class SampleController {
            @RequestMapping("/")
            @ResponseBody
            String home() {
                return "Hello World!";
            }

            public static void main(String[] args) throws Exception {
                SpringApplication.run(SampleController.class, args);
            }
        }

运行main方法，即可启动一个tomcat实例，访问 127.0.0.1:8080 即可输出 Hello World! 文案。

从这里可以发现使用spring-boot，可以非常大程度的减少我们的配置。


# 使用详情

这里在看的是[1.5.3RELEASE版本的使用说明文档](http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/reference/htmlsingle/)。

在上面例子中的`@EnableAutoConfiguration`注解，是告诉Spring-Boot基于项目中jar包的依赖去猜测如何配置Spring，例子中的`spring-boot-starter-web`加入了tomcat和spring mvc依赖，所以这个自动配置会假设你正在开发一个web应用。

## 创建一个可执行的jar包

由于java不支持任何标准的方式去加载嵌套的jar文件，所以想发布一个独立的应用程序是，可能会有问题，大多数开发者可能会用“uber” （即将所有jar包的classes文件打包到一个jar文件中） jar 来解决这个问题，但是“uber”jar 有个问题是，你无法知道你使用的哪些jar包，而且可能还会存在相同文件名的问题。

为了生成一个可执行的jar包，需要加入 spring-boot-maven-plugin 插件

          <build>
              <plugins>
                  <plugin>
                      <groupId>org.springframework.boot</groupId>
                      <artifactId>spring-boot-maven-plugin</artifactId>
                  </plugin>
              </plugins>
          </build>

然后执行命令：`mvn package`，然后它就会在target目录下生成一个jar包。

最后使用命令 `java -jar` ，即可将jar包运行起来。

spring-boot使用[另一种方式](http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/reference/htmlsingle/#executable-jar)，来允许直接嵌套jar包。

### starts 和 自动配置
自动配置会根据 starters 进行处理，但是这两个不是直接绑定的。所以你可以手动选择jar包依赖，而不使用 spring boot 的starts，spring boot 也可以自动配置你的应用。

安装spring-boot命令行工具 ：[传送门](http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/reference/htmlsingle/#getting-started-installing-the-cli)

spring-boot-maven-plugin 插件的使用；[http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/maven-plugin/usage.html](http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/maven-plugin/usage.html)

spring-boot的starts列表[http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/reference/htmlsingle/#using-boot-starter](http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/reference/htmlsingle/#using-boot-starter)

spring-boot常见问题：[传送门](https://stackoverflow.com/tags/spring-boot)

spring-boot使用例子：[创送门](https://github.com/spring-projects/spring-boot/tree/v1.5.3.RELEASE/spring-boot-samples)

## spring-boot的使用

推荐使用maven或者 gradle 依赖管理工具进行依赖管理，其他的依赖管理（例如ant），支持不是特别好。

spring boot的release版本都会伴随着一个依赖支持的列表。在实践中，你基本不用设置任何一个依赖的版本，全部交由spring boot来进行管理，当你更新spring boot时，相关的依赖也一并更新了。

### Maven
在maven中，使用继承spring-boot-starter-parent父pom的方式的默认特性：
* 默认 Java 1.6 的编译等级
* 源码使用 UTF-8 的编码
* 依赖可忽略`<version>`标签的填写
* 只能资源过滤 [resource filter](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)
* 智能插件配置 , [git commit id](https://github.com/ktoso/maven-git-commit-id-plugin),


# Starters
Starters是一个非常方便的依赖描述集合，可一站式获取所需的相关技术的包。例如如果想使用SPring和JPA（java persistent api）来做数据访问，则只需引入 spring-boot-starter-data-jpa 依赖即可。

util: http://docs.spring.io/spring-boot/docs/1.5.3.RELEASE/reference/htmlsingle/#using-boot-starter
