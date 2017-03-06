---
layout: post
title: Spring MessageSource 使用
category: 技术
tags: spring
keywords:
description:
---

前面 [springIoc初始化过程](/2017-02-17-springIoc初始化过程)中 看到有一个叫`MessageSource`的东东,接下来,看看这个接口的使用场景吧.

> /**
> * Strategy interface for resolving messages, with support for the parameterization
> * and internationalization of such messages.
> *
> * <p>Spring provides two out-of-the-box implementations for production:
> * <ul>
> * <li>{@link org.springframework.context.support.ResourceBundleMessageSource},
> * built on top of the standard {@link java.util.ResourceBundle}
> * <li>{@link org.springframework.context.support.ReloadableResourceBundleMessageSource},
> * being able to reload message definitions without restarting the VM
> * </ul>
> *
> * @author Rod Johnson
> * @see org.springframework.context.support.ResourceBundleMessageSource
> * @see org.springframework.context.support.ReloadableResourceBundleMessageSource
> */

从这个接口的注释可以看出,这个接口是用来解决消息国际化的问题的,并且提供了`ResourceBundleMessageSource`和`ReloadableResourceBundleMessageSource`两个开箱即用的实现.


## 示例

下面以一个简单的例子,来说明它是如何使用的

第一步:

在类路径下加入 message-source_zh.properties 配置文件,内容如下:

> message.key=hello, {0}, welcome home


第二步:

在spring的bean配置文件中加入`ResourceBundleMessageSource`的bean配置,注意这里的basename不能加'.properties'后缀.(这里使用了 p 标签, 需在beans的命名空间中加入`xmlns:p="http://www.springframework.org/schema/p"`)

          <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource"
                    p:basename="message-source"
              />


第三步:

在逻辑中使用 messageSource

            @Resource
            private MessageSource messageSource;

            public String doProcessMessage() {
                String msg = messageSource.getMessage("message.key", new Object[]{"jaky.wang"}, Locale.CHINESE);
                System.out.println(msg);
                return msg;
            }

执行结果输出:

> hello, jaky.wang, welcome home

上面是一个简单的使用示例, 这个示例中我指定了参数以及对应的地区.(在实际开发中,地区多半是通过配置来获取的)

## 其他

`MessageSource`接口的三个方法,大同小异,通过注释即可明白其使用方法, 在注释中,你还可以发现参数的其他用法,比如指定参数的类型 `{0,date}`,即这个参数的类型是Date, 例如将 message-source_zh.properties 文件内容修改为 :

        message.key=hello, {0,date}, welcome home

然后执行

        messageSource.getMessage("message.key", new Object[]{new Date()}, Locale.CHINESE);

即可输出结果:

        hello, 2017-3-6, welcome home

通过调试阅读源码,你会发现 `ResourceBundleMessageSource`中获取配置是有优先顺序的,它会优先去获取以对应地区二字码(可在`java.util.Locale`中查看其他地区二字码)缩写结尾的文件,如果获取不到,它会获取无二字码缩写的文件(在上面的例子中,就是 message-source.properties) 中的配置
