---
layout: post
title: web安全
category: 技术
tags: web
keywords:
description:
---

##  Cross Site Request Forgery (CSRF)
跨站请求伪造.

### 攻击例子

考虑以下请求, 登陆后通过下面的请求进行转账操作.

        POST /transfer HTTP/1.1
        Host: bank.example.com
        Cookie: JSESSIONID=randomid; Domain=bank.example.com; Secure; HttpOnly
        Content-Type: application/x-www-form-urlencoded

        amount=100.00&routingNumber=1234&account=9876

如果在没有退出登陆的情况下,登陆了另外一个网站,在网站中有如下一个按钮:

        <form action="https://bank.example.com/transfer" method="post">
        <input type="hidden"
        	name="amount"
        	value="100.00"/>
        <input type="hidden"
        	name="routingNumber"
        	value="evilsRoutingNumber"/>
        <input type="hidden"
        	name="account"
        	value="evilsAccountNumber"/>
        <input type="submit"
        	value="Win Money!"/>
        </form>

点击按钮后,则你的账户会少100块钱 :(

虽然在另外一个站点中,它无法获取 bank.example.com 的cookie信息,但是cookie信息会随着这次请求被一起带过去,多么痛的领悟.

更坏的情况,是这个点击按钮的事儿,完全可以有js代码自动执行.也就是一旦进入上面的站点,钱就少了.

这就是 CSRF 攻击.

### 原因
造成CSRF攻击的原因,主要是,由其他站点过来的请求 和 bank.example.com 过来的请求完全一致, 也就是说服务端没有方法可以判断是否应该拒绝这个请求.所以解决方法就是,在请求中放置一些从其他站点过来的请求无法提供的信息.

### 解决方案
令牌模式(Synchronizer Token Pattern), 这个方案的核心就是除了cookie信息之外,为每一个请求随机生成一个令牌,作为请求参数.当请求提交到服务端后,需将请求参数中的token和服务端真实的token进行比较,如果不匹配,则请求失败.

增加token后的请求参数,类似如下:

        POST /transfer HTTP/1.1
        Host: bank.example.com
        Cookie: JSESSIONID=randomid; Domain=bank.example.com; Secure; HttpOnly
        Content-Type: application/x-www-form-urlencoded

        amount=100.00&routingNumber=1234&account=9876&_csrf=<secure-random>

### 使用的例子
[例子传送门](https://github.com/spring-projects/spring-mvc-showcase/commit/361adc124c05a8187b84f25e8a57550bb7d9f8e4)

更多使用:[传送门](http://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#csrf-include-csrf-token-ajax)
