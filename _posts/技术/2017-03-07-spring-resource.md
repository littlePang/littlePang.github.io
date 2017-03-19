---
layout: post
title: Spring resources 使用
category: 技术
tags: spring
keywords:
description:
---

# introduction

Java标准的`java.net.URL`和标准的处理类无法满足所有的底层资源访问,例如,没有一个标准的`URL`实现可以用来访问需要从 classpatch (或者 ServletContext) 下面获取的资源.虽然可以为一个指定的`URL`前缀指定一个新的处理类(类似对`http:`前缀的处理类),但是通常都比较复杂,并且`URL`接口还缺乏一些令人满意的方法.例如判断一个资源是否存在的方法.

# `Resource` 接口

spring的`Resource`接口是想定义一个对访问底层资源更加合适的接口

          public interface Resource extends InputStreamSource {

              boolean exists();

              boolean isOpen();

              URL getURL() throws IOException;

              File getFile() throws IOException;

              Resource createRelative(String relativePath) throws IOException;

              String getFilename();

              String getDescription();

          }


          public interface InputStreamSource {

              InputStream getInputStream() throws IOException;

          }

`Resource`接口的一些主要方法

* `getInputStream()` : 定位并打开资源,返回一个读取这个资源的输入流.每一次执行都返回一个新的输入流.由调用者来关闭这些流.
* `exists()` : 返回 boolean 表示资源是否真实存在.
* `isOpen()` : 返回这个资源是否有操作流被打开. true 表示输入流(InputStram)不能被多次读,并且需要读一次就关闭操作流,避免资源泄露. 其他所有的资源实现都应该返回false.
* `getDescription()` : 返回资源的描述.用于做资源处理时的错误输出.通常这个返回的是资源的全路径名或者实际的URL

还有其他的一些方法,可以用来获取一个代表这个资源的`URL`或者`File`对象.

`Resource`抽象在spring中广泛的使用.在大量方法需要资源时,在方法产生中都声明了`Resurce`参数类型.其他的一些方法,例如`ApplicationContext`的实现,通过接收一个`String`来创建一个`Resource`.

虽然`Resource`接口在spring中使用的非常多,但是即使你的代码不适用spring的其他部分,它也可以作为一个通用访问资源的工具类.虽然它会使代码和spring耦合,但是也只有工具类这一小部分,并且它提供了比`URL`更加优秀的服务.

`Resource`抽象不会替换功能,它会尽量将其包装起来,例如 `UrlRsource`用来包装`Url`为一个`Resource`

# 内建的`Resource`实现.

* `UrlResource`: `java.net.URL`的包装类(`http:`,`file:`,`ftp:`等前缀)．
* `ClassPathResource`: 这个类表示的资源应该是包含在classpath目录下的.通过当前线程的类加载器,或者指定的加载器,又会在指定的类的来加载资源
* `FileSystemResource`:这个实现用来处理`java.io.File`的处理.
* `ServletContextResource`: 处理`ServleteContext`资源.处理web应用根目录相关的资源
* `InputStreamResource` : 处理`InputStream`, 它的`isOpen()`返回true,所以需要保存资源描述符或者需要使用多个读取流的时候,不要使用此类.
* `ByteArrayResource` :处理`ByteArrayInputStream`.通常用来在一个任意给定的字节数组中获取文本内容.

# `ResourceLoader`
表明类可以返回一个`Resource`实例引用.

          public interface ResourceLoader {
              Resource getResource(String location);
          }

所有的`ApplicationContext`子类都实现了`ResourceLoader`接口.每一个对应的不同的`ApplicationContext`,都返回器所对应的`Resource`,例如 `ClassPathXmlApplicationContext`返回`ClassPathResource`,`FileSystemXmlApplicationContext`返回`FileSystemResource`,`WebApplicationContext`返回`ServletContextResource`

由于所有的`Resource`都支持`java.net.URL`的使用方式,所以可以像下面一样使用:

          Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
          Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");
          Resource template = ctx.getResource("http://myhost.com/resource/path/myTemplate.txt");


# `ResourceLoaderAware`接口
标记型接口,实现了此类的接口,可以获取到`ResourceLoad`, 又或者通过`@Autowired`等方式来获取.

# 资源依赖
在bean配置中可使用以下简单配置使用`Resource` :

        <bean id="myBean" class="...">
            <property name="template" value="some/resource/path/myTemplate.txt"/>
        </bean>

其中属性`template`的类型为`Resource`, value值没有指定前缀,则具体使用的`Resource`实现,根据当前上下文类型来决定使用哪个(`ClassPathResource`,`ServletContextResource`等), 也可通过下面这种前缀的方式指定`Resource`的实现.

          <property name="template" value="classpath:some/resource/path/myTemplate.txt">
          <property name="template" value="file:///some/resource/path/myTemplate.txt"/>

# 通配符

资源地址支持 ant-style 风格的模式

          /WEB-INF/*-context.xml
            com/mycompany/**/applicationContext.xml
            file:C:/some/path/*-context.xml
            classpath:com/mycompany/**/applicationContext.xml

# 参考
[http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#resources](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#resources)
