---
layout: post
title: SpingMvc的@Responsebody返回值中文乱码问题
category: 技术
tags: Spring
keywords:
description:
---

#SpringMvc 返回值中文乱码问题

## 在springMvc的使用中,想要使用以下代码,返回一个包含,中文的字符串,然而返回结果里面的中文全都变成了乱码.

        @RequestMapping("/test")
        @ResponseBody
        public Object tgqServiceStandard(String domain, String orderNo) {
            return "这是一个测试Controller";
        }

## 解决方法
在RequestMapping注解中加入如下的参数即可:

        @RequestMapping(produces="text/plain;charset=UTF-8",value="/test")
        @ResponseBody
        public Object tgqServiceStandard(String domain, String orderNo) {
            return "这是一个测试Controller";
        }

这样,返回值中的中文即可正常显示了.

## 中文乱码产生原因

在springMvc所使用的返回值装换器 org.springframework.http.converter.StringHttpMessageConverter 中.
所使用的字符编码默认是 ISO-8859-1 , 所以字符串就乱码了

    public static final Charset DEFAULT_CHARSET = Charset.forName("ISO-8859-1");
