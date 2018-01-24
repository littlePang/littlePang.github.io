---
layout: post
title: spring-cloud-confg使用
category: 技术
tags: spring-cloud
keywords:
description:
---


# spring-cloud-config client端配置

            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-config</artifactId>
                <version>1.3.2.RELEASE</version>
            </dependency>


# 在服务器无法更新仓库

在服务器仓库下，多了一堆  ‘._spring-root.xml6255164676680237796.tmp’ 这种未 被git跟踪的文件，导致程序无法更新参考，暂未弄清出现此种文件的原因。

可将 force-pull 置为 true


在 1.1.0 版本中将 encrypt 配置加在application.yaml中可以生效，
但是在后续版本 例如 1.3.3 中 却不生效（效果是 获取到的配置只是 少了{cipher}，配置串还是加密的），必须加在 bootstrap.yaml 文件中才行，真是蛋疼

ReousrceController 用于获取某个配置文件中的指定的某个配置的值

EnvironmentController 用于获取整个配置文件的配置。

判断git仓库是否需要 pull org.springframework.cloud.config.server.environment.JGitEnvironmentRepository#shouldPull
如果仓库不是干净的(有修改，新增，未跟踪文件等),则不会pull

如果本地仓库fetch，并merge远端分支后，仓库并不是clean的（有冲突）,则会将仓库 reset -hard

git仓库更新，文件查询 都是 synchronized的。  

repos 的 pattern 使用 application/profile 进行匹配，其中一个找到之后，就不在继续往下找
，org.springframework.cloud.config.server.environment.MultipleJGitEnvironmentRepository.PatternMatchingJGitEnvironmentRepository#matches

repos 中的属性覆盖设置 forcePull 不会使用上面默认git的配置 org.springframework.cloud.config.server.environment.MultipleJGitEnvironmentRepository#afterPropertiesSet

git仓库配置都在 org.springframework.cloud.config.server.environment.MultipleJGitEnvironmentRepository
