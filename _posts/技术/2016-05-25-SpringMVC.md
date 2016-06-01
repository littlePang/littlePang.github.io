---
layout: post
title: SpringMVC学习
category: 技术
tags: Spring
keywords:
description:
---

官方文档: [传送门](http://docs.spring.io/spring/docs/4.1.9.RELEASE/spring-framework-reference/htmlsingle/#mvc)

pdf下载 : [传送门](http://docs.spring.io/spring/docs/4.1.9.RELEASE/spring-framework-reference/pdf/)

Spring Web MVC (Model-View-Controller) 框架是围绕着 DispatcherServlet 设计用于处理请求的框架(使用可配置的处理器映射,视图方案,区域,时区,主题方案,并且对文件上传有较好的支持).

在spring web mvc 中可以使用任何对象作为一个命令或者表单后盾对象,不需要实现框架指定的接口或者基类. spring的数据绑定高度灵活: 例如 处理类型匹配错误作为验证错误, 可以由应用来评估, 而不是作为系统错误,从而可以尽量简单使业务对象属性不再重复,没有类型的字符串在表单对象中可以简单的处理验证提交, 或者转换成字符串属性,代替的是,常常最好是直接绑定业务对象.


spring 的视图方案是非常灵活的.一个Controller可以用一个数据Map和一个视图名称来作为请求应答,也可是直接把数据写入输出流里面.视图名称解决方案是高可配置的(通过文件扩增名, Header中的Accept Content-type类型, Bean名称,属性文件,又或者是实现ViewResolver来指定),数据是个Map接口,允许完成视图技术的虚实现. 也可以直接与模板整合(JSP,Velocity,Freemarker等,或者直接生成XML,JSON,Atom,和其他类型的文本).数据模型Map会被简单的翻译成一个合适的格式,例如JSP中的request attributes, Velocity temllate model.


## spring web模块支持的特性
* 清晰的角色划分,每一种角色- controller, validator, command object, form object, model
object, DispatcherServlet, handler mapping, view resolver 等, 都可以被专门的对象所验证
* 强大且直接了当的配置(将框架可应用的类都作为JavaBean). 配置能力包括 通过上下文简单的引用,从Controller中应用业务对象和校验器
* 适应性,非入侵式,和灵活性.可定义任何一个所需要的Controller方法签名,在一些场景下可能需要用到一些参数注解(@RequestParam,@RequestHeader,@PathVariable等).
* 可复用的业务代码,无需重复.使用一个已经存在业务对象zowie一个命令或者表单对象 代替 为了扩展指定的框架基类而复制他们.
* 可定制的绑定与校验.将类型匹配失败作为应用级别的校验错误,并持有违规的值, 用来代替只用字符串接收表单对象并手工解析和转换为业务对象
* 可定制的处理映射和视图方案.
* 灵活的模型转换.模型转换使用简单的 name/value 的 map形式,从而可以简单的与任何视图技术整合.
* 可定制的区域,时区和主题方案.支持JSP, JSTL, Velocity
* 简单且强大的JSP标签库
* Bean 的生命周期的范围可以是 一次Http请求,或者是 Http Session 的范围.(WebApplicationContext中Bean的作用范围分为:Requst,session 和 global session scopes)

## The DispatcherServlet
spring web framework和其他MVC框架一样, 请求驱动,围绕着一个中心Servlet来派发请求到controller 并且提供其他的一些利于开发的功能.同时也整合了Spring IOC容器,所以也包含spring的特性.
请求工作流如下图所示:

![](/assets/picture/2016-05-30_1.png)

DispatcherServlet除了在web.xml中配置外,也可使用编程的方式使用

        public class MyWebApplicationInitializer implements WebApplicationInitializer {
          @Override
          public void onStartup(ServletContext container) {
              ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new
              DispatcherServlet());
              registration.setLoadOnStartup(1);
              registration.addMapping("/example/\*");
          }
        }



在springMVC中每个DispatcherServlet都有它自己的WebApplicationContext,这个上下文会继承所有已经定义在根上下文WebApplicationContext中的Bean,这些继承可以在servlet指定的作用域中覆盖,也可以自己给Servlet引用定义一个新的本地作用域.

springMVC默认寻找 /WEB-INFO下的 [servlet-name]-servlet.xml文件,作为DispacherServlet的配置文件

根上下文的在web.xml中的配置(默认去获取上下文参数中contextConfigLocation所对应的值,作为根上下文的配置文件路径)

      <web-app>
        <context-param>
          <param-name>contextConfigLocation</param-name>
          <param-value>/WEB-INF/root-context.xml</param-value>
        </context-param>
        <listener>
          <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
        </listener>
      </web-app>

可以使用 `RequestContextUtils`来获取 WebApplicationContext

### Special Bean Types In the WebApplicationContext
DispatcherServlet使用指定的beans来处理请求和渲染合适的视图(如果不做任何配置则SpringMVC会有默认的bean来处理)
* HanderMapping 处理到来的请求映射关系 (最受欢迎的处理器是基于注解的处理器)
* HanderAdapter 帮助DIspatcherServlet执行请求映射处理,忽略真正的处 的注解.理器.例如:执行一个基于注解的Controller要去处理一系列注解,HanderAdapter的主要目的就是向DispatcherServlet屏蔽处理细节.
* HandlerExceptionResolver 映射异常处理视图,也可以加入复杂异常的处理代码
* ViewResolver 解决基于字符串的逻辑视图名字 到一个 真实的视图类型
* LocaleResolver & LocaleContextResolver 解决用户区域和时区问题,提供一个国际化的视图
* ThemeResolver 解决web应用使用的主题问题,例如提供个性化的布局
* MultipartResovler 解析分块的请求,比如支持处理文件上传
* _FlashMapManager_ 存储和检索输入和输出,可以用来将一个参数从一个请求传递给另外一个.常用来处理重定向

### Default DispatcherServlet Configuration
上面提到的DispacherServlet对每一种指定的bean都有一个默认的使用,这个默认配置在 包 `org.springframework.web.servlet` 下的 DispacherServlet.properties文件中


### DispatcherServlet Processing Sequence
请求处理流程
* 搜索webApplicationContext并将其作为一个attribute放到Requst中供controller和其他元素使用.webApplicationContext默认绑定到Request中的key为 DispacherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE.
* 绑定一个Locale Resovler到Request中
* 绑定一个theme Resovler到Request中
* 如果指定了分块文件处理器,请求会被检查是否为分块请求.如果是分块请求,则请求会被包裹进 MultipartHttpServletRequest中,以方便未来由其他元素来处理.
* 寻找一个合适的处理器,执行处理器的关系链(preprocessors,postprocessors以及controller),准备数据模型或渲染
* 如果返回了一个Model,则渲染视图.如果没有返回一个Model(可能是由于前置处理器或者后置处理器拦截了请求,或许是由于安全的原因),则不渲染视图,因为这个请求以及完成了.

异常处理器声明在WebApplicationContext中,如果在请求处理期间发生异常,则可以用这些异常处理器自定义异常对应的行为

spring DispatcherServlet也支持返回 last-modification-date, 由Servlet API指定.DispacherServlet会搜索合适的请求映射并且测试它是否实现了 `LastModified` 接口.如果实现了该接口,则使用接口的方法获取对应的时间,返回给客户端(该值可用于客户端实现缓存)

通过配置初始化参数,可以定制个人的DispacherServlet,下面罗列的是所有DispacherServlet的初始化参数
* contextClass 指定一个WebApplicationContext的子类,Servlet实例化上下文所用的类,默认使用XmlWebApplicationContext
* contextConfigLocation  指定上下文配置文件路径(通过逗号分割,支持多个配置文件)
* namespace   WebApplicationContext的命名空间,默认是 [servlet-name]-servlet.


## Implementing Controller
controller提供通过服务接口来访问应用行为.controller可以理解为 转换用户输出数据变成一个Model,以方便视图展现.spring实现一个controller的方式非常抽象,所以可以创建各种各样的controller

controller的实现无需继承任何指定基类或者实现指定接口,不会直接依赖Servlet或者Portlet 的APIs,因此可以轻松的在Servlet或者Portlet设备上使用

controller简单使用:

        @Controller
        public class HelloWorldController {
            @RequestMapping("/helloWorld")
            public String helloWorld(Model model) {
                model.addAttribute("message", "Hello World!");
                return "helloWorld";
            }
        }


使用`@Controller`来定义一个控制器
注解扫描配置:

        <?xml version="1.0" encoding="UTF-8"?>
        <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:p="http://www.springframework.org/schema/p"
          xmlns:context="http://www.springframework.org/schema/context"
          xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context.xsd">

          <context:component-scan base-package="org.springframework.samples.petclinic.web"/>
          <!-- ... -->

        </beans>



使用`@RequestMapping`来映射请求
常规用法:


        @RequestMapping(value="/{day}", method = RequestMethod.GET)
        public Map<String, Appointment> getForDay(@PathVariable @DateTimeFormat(iso=ISO.DATE) Date day,
        Model model) {
            return appointmentBook.getAppointmentsForDay(day);
        }
        @RequestMapping(value="/new", method = RequestMethod.GET)
        public AppointmentForm getNewForm() {
            return new AppointmentForm();
        }
        @RequestMapping(method = RequestMethod.POST)
        public String add(@Valid AppointmentForm appointment, BindingResult result) {
            if (result.hasErrors()) {
                return "appointments/new";
            }
            appointmentBook.addAppointment(appointment);
            return "redirect:/appointments";
        }


###  @Controller's and AOP Proxying (_这个鬼,不太清楚具体的问题,后续再看_)
一些时候controller可能需要通过动态代理来修饰,如果使用了Spring Context非回调式的接口(InitializingBean,\*Aware等),则需要指定class-based代理

### New Support Classes for @RequestMapping methods in Spring MVC 3.1
spring 3.1 有一个新的支持 @RequestMapping 的类集合.分别是RequestMappingHandlerMapping 和 RequestMappingHandlerAdapter.它们被推荐在以后的版本使用.

在spring 3.1之前,类型和方法级别的请求映射是由两步来决定的,首先,由DefailtAnnotationHandlerMapping来选取一个controller,然后再由AnnotationMethodHandlerAdapter来选取真正执行的方法.

在spring 3.1的新支持中,`RequestMappingHanderMapping`会决定出请求应该被哪个方法执行．考虑把controller的方法作为一个唯一端点的集合,使用类型和方法级别的@RequstMapping信息映射每一个方法.

新的支持中,以下问题将_不再出现_(如果强行配置为老的类,则依然有以下问题):
* 先由SimpleUrlHandlerMapping 或者 BeanNameUrlHandlerMapping选择一个controller,然后再基于@RequstMapping选择方法,
* 依赖方法名称作为回调机制,导致两个@RequestMapping方法没有明确的路径映射,但是其他都匹配的情况下,产生歧义.新的支持类每个@RequstMapping方法都是唯一映射的
*  使用默认方法(没有一个明确的路径映射)用来处理其他controller没有匹配的情况,新的支持类,如果不匹配则返回404页面


## URI Template Patterns

使用:
1.在方法上
      @RequestMapping(value="/owners/{ownerId}", method=RequestMethod.GET)
      public String findOwner(@PathVariable String ownerId, Model model) {
          Owner owner = ownerService.findOwner(ownerId);
          model.addAttribute("owner", owner);
          return "displayOwner";
      }

2. 在Controller上

        @Controller
        @RequestMapping("/owners/{ownerId}")
        public class RelativePathUriTemplateController {
            @RequestMapping("/pets/{petId}")
            public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {
                // implementation omitted
            }
        }

3. 正则表达式的使用
以下方法匹配的URL为 /spring-web/spring-web-3.0.5.jar

        @RequestMapping("/spring-web/{symbolicName:[a-z-]}-{version:\\d\\.\\d\\.\\d}{extension:\\.[a-z]}")
        public void handle(@PathVariable String version, @PathVariable String extension) {
            // ...
        }

正则表达式匹配问题:
URL匹配多个正则时,则考虑使用更加具体的匹配
* 使用变量更少的匹配(/hotels/{hotel}/* 比 hotels/{hotel}/** 更加具体)
* 变量个数相等,使用匹配更长的url映射(foo/bar* 和 foo/\*,使用前者)
* 变量个数和url长度都想等时,通配符越少越匹配.(/hotels/{hotel} 和 /hotels/* 使用前者)
正则匹配的特殊规则:
* 默认匹配串/\*\*比任何其他的正则都更加不具体.例如/api/{a}/{b}/{c},则优先使用后者
* 类似于 /public/** 的正则匹配串 比 任何不包含双重通配符

都更加不具体,例如:/public/path3/{a}/{b}/{c},后者更加具体
匹配URL的所有详情在`AntPathMatcher`中的`AntPatternComparator`中(路径匹配是可自定义的).


### Path Patterns with Placeholders
路径正则匹配,支持 ${...}的占位符. 详细信息在`PropertyPlaceholderConfigurer`中

### Suffix Pattern Matching
后缀匹配. springMvc默认匹配 ".{asterisk}"后缀. Controller 中的URL `/person` 默认匹配 `/person.{asterisk}` 比如 /person.pdf 和 /person.xml


### Matrix Variables
[RFC 3986](http://tools.ietf.org/html/rfc3986#section-3.3) 中定义了在路径中包含 name-value 的键值对

使用:
springMVC文件配置

        <mvc:annotation-driven enable-matrix-variables="true"/>

1.

        // GET /pets/42;q=11;r=22
        @RequestMapping(value = "/pets/{petId}", method = RequestMethod.GET)
        public void findPet(@PathVariable String petId, @MatrixVariable int q) {
            // petId == 42
            // q == 11
        }

2.

        // GET /owners/42;q=11/pets/21;q=22
        @RequestMapping(value = "/owners/{ownerId}/pets/{petId}", method = RequestMethod.GET)
        public void findPet(
        @MatrixVariable(value="q", pathVar="ownerId") int q1,
        @MatrixVariable(value="q", pathVar="petId") int q2) {
            // q1 == 11
            // q2 == 22
        }


3.

        // GET /pets/42
        @RequestMapping(value = "/pets/{petId}", method = RequestMethod.GET)
        public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {
            //  q == 1
        }

4.

        // GET /owners/42;q=11;r=12/pets/21;q=22;s=23
        @RequestMapping(value = "/owners/{ownerId}/pets/{petId}", method = RequestMethod.GET)
        public void findPet(
        @MatrixVariable Map<String, String> matrixVars,
        @MatrixVariable(pathVar="petId"") Map<String, String> petMatrixVars) {
            // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
            // petMatrixVars: ["q" : 11, "s" : 23]
        }


### Consumable Media Types
可使用媒体类型来匹配请求,若请求媒体类型不在配置list中,则无法匹配请求

        @Controller
        @RequestMapping(value = "/pets", method = RequestMethod.POST, consumes="application/json")
        public void addPet(@RequestBody Pet pet, Model model) {
            // implementation omitted
        }

### Producible Media Types
使用 请求头中的 Accept 匹配可接受的类型 来匹配 url

        @Controller
        @RequestMapping(value = "/pets/{petId}", method = RequestMethod.GET, produces="application/json")
        @ResponseBody
        public Pet getPet(@PathVariable String petId, Model model) {
            // implementation omitted
        }


### Request Parameters and Header Values
使用请求的参数进行url匹配

        @Controller
        @RequestMapping("/owners/{ownerId}")
        public class RelativePathUriTemplateController {
            @RequestMapping(value = "/pets/{petId}", method = RequestMethod.GET, params="myParam=myValue")
            public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {
                // implementation omitted
            }
        }

使用请求头中信息匹配url

        @Controller
        @RequestMapping("/owners/{ownerId}")
        public class RelativePathUriTemplateController {
            @RequestMapping(value = "/pets", method = RequestMethod.GET, headers="myHeader=myValue")
            public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {
                // implementation omitted
            }
        }



## Defining @RequestMapping handler methods (p454)
定义一个 `@RequstMapping`的方法签名是非常灵活的,除了 `BindingResult` 参数,其他大部分参数都可以随意排序.

### `@ReqiestMapping`方法支持的参数
* Request 和 response 对象 (Servlet API), 可选择任意指定的 request 或 response 类型,如 ServletRequest 或 HttpServletRequest
* Session 对象 (Servlet API) , 类型是 HttpSession, Session对象的访问并不是线程安全的,多线程并发访问Session, 可以考虑将 `RequestMappingHandlerAdapter`的 `synchronizeOnSession`配置为 true
* org.springframework.web.context.request.WebRequest 和 org.springframework.web.context.request.NativeWebRequest 对象. 通用的访问请求参数(request/session). 而不用绑定 servlet/portlet API
* java.util.Locale [对象说明](http://blog.csdn.net/hooabc/article/details/5997497), 由可用的 区域处理器 决定. MVC中的 LocaleResolver/LocaleContextResolver配置
* java.util.TimeZone (Java 6+) / java.time.ZoneId (on Java 8) , 由 LocaleContextResolver 决定
* java.io.InputStream / java.io.Reader , 用来访问请求内容(request’s content), InputStream/Reader 值由Servlet API 来暴露
* java.io.OutputStream / java.io.Writer , 用来访问返回内容(response's context), OutputStream/Writer 由Servlet API来暴露
* org.springframework.http.HttpMethod , 请求方法枚举(GET, POST, HEAD, OPTIONS, PUT, PATCH, DELETE, TRACE)
* java.security.Principal 包含当前验证用户
* 相关注解 `@PathVariable`, `@MatrixVariable`, `@RequestParam`, `@RequestHeader`, `@RequestBody`, `@RequestPart`, `HttpEntity<?>`(使用HttpMessageConverter进行参数转换)
* java.util.Map/org.springframework.ui.Model/org.springframework.ui.ModelMap, 丰富暴露给页面展示的数据
* org.springframework.web.servlet.mvc.support.RedirectAttributes , 指定重定向的属性集合(重定向带过去的参数信息), 返回视图包含"redirect:"或者RedirectView时,可用.  参数集合同时也会被放到 flash attribute 中[flash attribute说明](http://www.open-open.com/lib/view/open1397266120028.html)
* 命令或者表单对象使用可定制的转换绑定bean属性, 依赖 `@InitBinder` 方法 和 HandlerAdapter配置 (请看`RequestMappingHandlerAdapter`中的`webBingdingInitializer`属性). `ModelAttribute`注解.可以用在方法参数上去定制属性名称. ([ModelAttribute注解的使用](http://blog.csdn.net/li_xiao_ming/article/details/8349115))
* org.springframework.validation.Errors/org.springframework.validation.BindingResult, 绑定属性的验证结果, 每个绑定属性的结果对象必须放在,model对象的下一个参数声明.`public String processSubmit(@ModelAttribute("pet") Pet pet, Model model, BindingResult result)` 与 `public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result, Model model)`. 则前面一种声明,不会生效
* org.springframework.web.bind.support.SessionStatus. 与`@SessionAttributes`注解相关.  (获取SessionStatus后,可调用setComplete方法,标记session处理已完成,此时会触发Session的清理操作,清理掉`@SessionAttributes`存放的属性)
* org.springframework.web.util.UriComponentsBuilder. 当前请求的host, port, scheme, context path等信息


### 支持的方法返回类型:
* ModelAndView, Model中会包含通过`@ModelAttribute`放入的属性
* Model, Model中会包含通过`@ModelAttribute`放入的属性, 视图名称通过`RequestToViewNameTranslator`获取
* Map, 通过一个map来暴露一个model,其中也会包含通过`@ModelAttribute`放入的属性
* View, 视图的Model中包含通过`@ModelAttribute`放入的属性, 也可通过声明一个Model方法参数来丰富Model
* String, 表示一个逻辑视图的名称. 视图的Model中包含通过`@ModelAttribute`放入的属性, 也可通过声明一个Model方法参数来丰富Model
* void,   则可通过声明ServletResponse/HttpServletResponse参数来自己写入返回信息.或者通过`RequestToViewTranslator`来执行视图名称(此种情况在处理方法的签名中不可什么response参数)
* 如果方法被`@ResponseBody`注解标注.则它的返回值会被写入 Http的返回体中.返回值转换由 `HttpMessageConverters`来中操作
* HttpEntity<?> 或者 ResonseEntity<?> (待查看如何使用?)
* Callable<?> 如果想要异步处理请求,则可返回Callable实现
* DefferredResult<?> 应用想要通过自己选择的线程获取返回值(什么鬼?)
* ListenableFuture<?>  应用想要通过自己选择的线程获取返回值(这又什么鬼?)
* 其他任何类型的返回值都被当作一个单独的Model中的属性暴露给视图.(可通过`@ModelAttribute`指定该属性的名称,或者使用返回类型的类名作为默认属性名称)

### 使用说明:
* 1.. `@RequestParam` 如果被用在 Map<String, String> 或者 MultiValueMap<String, String>参数上,则map中会包含所有的请求参数
* 2.. `@RequestBody` 会将http请求体的中数据全部放入参数中.`public void handle(@RequestBody String body, Writer writer)`, 也可使用 HttpMessaeConverter 转换成支持的类型
* 3.. 可通过类似如下的方式增加类型转换器:

          <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
              <property name="messageConverters">
                <util:list id="beanList">
                  <ref bean="stringHttpMessageConverter"/>
                  <ref bean="marshallingHttpMessageConverter"/>
                </util:list>
              </property
              </bean>
          <bean id="stringHttpMessageConverter" class="org.springframework.http.converter.StringHttpMessageConverter"/>
          <bean id="marshallingHttpMessageConverter"
              class="org.springframework.http.conerter.xml.MarshallingHttpMessageConverter">
                <property name="marshaller" ref="castorMarshaller" />
                <property name="unmarshaller" ref="castorMarshaller" />
          </bean>
          <bean id="castorMarshaller" class="org.springframework.oxm.castor.CastorMarshaller"/>

如果controller中的方法声明中没有声明类型转换错误的参数(Error)时,则会抛出MethodArgumentNotValidException异常,并向客户端返回400状态码

* 4.. 使用 `@RestController` 则相当于将contrller中的所有请求处理方法上加上`@ResponseBody`注解, 它是 `@ResponseBody` 和 `@Controller` 的一种组合形式, 可以`@ControllerAdvice`一起使用(这个注解干嘛的?)

* 5.. HttpEntity的使用, 与`@RequestBody` and `@ResponseBody`类似, 可获取请求头中的信息

          @RequestMapping("/something")
          public ResponseEntity<String> handle(HttpEntity<byte[]> requestEntity) throws
          UnsupportedEncodingException {
                String requestHeader = requestEntity.getHeaders().getFirst("MyRequestHeader"));
                byte[] requestBody = requestEntity.getBody();
                // do something with request header and body
                HttpHeaders responseHeaders = new HttpHeaders();
                responseHeaders.set("MyResponseHeader", "MyValue");
                return new ResponseEntity<String>("Hello World", responseHeaders, HttpStatus.CREATED);
          }

* 6.. `@ModelAttribute`像下面一样用在方法参数上,则表明, 参数会优先在 Model中查找,如果找到,则使用该值(也可能是使用`@SessionAttribute`添加的对象), 如果没有找到,则实例化该对象,并用请求中的参数填充对象.

        @RequestMapping(value="/owners/{ownerId}/pets/{petId}/edit", method = RequestMethod.POST)
        public String processSubmit(@ModelAttribute Pet pet) { }

   下面的情况.则需要注册一个`Converter<String,Account>`转换器.

        @RequestMapping(value="/accounts/{account}", method = RequestMethod.PUT)
        public String save(@ModelAttribute("account") Account account) {
        }

* 7.. 以下使用,表明 将 controler的Modle中的 Pet 属性放入 Session中

        @Controller
        @RequestMapping("/editPet.do")
        @SessionAttributes("pet")
        public class EditPetForm {
        // ...
        }

* 8.. 使用`@CookieValue`获取cookie中的信息

            @RequestMapping("/displayHeaderInfo.do")
            public void displayHeaderInfo(@CookieValue("JSESSIONID") String cookie) {
            //...
            }


* 9.. 使用 `@InitalBean`定制参数类型转换

        @Controller
        public class MyFormController {
              @InitBinder
              public void initBinder(WebDataBinder binder) {
                  SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
                  dateFormat.setLenient(false);
                  binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
              }
        // ...
        }


* 10.. 页面 最后修改时间 last-modification 的使用, 在第二步中,如果方法返回为true,则会将response的状态码设置为304

          @RequestMapping
          public String myHandleMethod(WebRequest webRequest, Model model) {
                long lastModified = // 1. application-specific calculation
                if (request.checkNotModified(lastModified)) {
                // 2. shortcut exit - no further processing necessary
                return null;
                }
                // 3. or otherwise further request processing, actually preparing content
                model.addAttribute(...);
                return "myViewName";
          }


* 11.. `@ControllerAdvice` 标注的类中,常常用来包含 @ExceptionHandler, @InitBinder, @ModelAttribute 等注解方法,在所有满足条件的`@RequestMapping`调用的时候使用.

          // Target all Controllers annotated with @RestController
          @ControllerAdvice(annotations = RestController.class)
          public class AnnotationAdvice {}

          // Target all Controllers within specific packages
          @ControllerAdvice("org.example.controllers")
          public class BasePackageAdvice {}

          // Target all Controllers assignable to specific classes
          @ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
          public class AssignableTypesAdvice {}

* 12.. Jackson序列化支持,使用Jackson序列化的方式: 1. `@ResponseBody` 2. 返回`ResponseEntity`, 3. `@JsonView`

`@JsonView`的简单使用

          @RestController
          public class UserController {
              @RequestMapping(value = "/user", method = RequestMethod.GET)
              @JsonView(User.WithoutPasswordView.class)
              public User getUser() {
                  return new User("eric", "7!jd#h23");
              }
          }

          public class User {
              public interface WithoutPasswordView {};
              public interface WithPasswordView extends WithoutPasswordView {};
              private String username;
              private String password;

              public User() {
              }

              public User(String username, String password) {
                this.username = username;
                this.password = password;
              }

              @JsonView(WithoutPasswordView.class)
              public String getUsername() {
                  return this.username;
              }

              @JsonView(WithPasswordView.class)
              public String getPassword() {
                  return this.password;
              }
          }


* 13.. SpringMVC对JSONP的支持, 在使用了 @ResponseBody 和 ResponseEntity 的方法时.可按如下方式实现JSONP

        @ControllerAdvice
        public class JsonpAdvice extends AbstractJsonpResponseBodyAdvice {
            public JsonpAdvice() {
                super("callback");
            }
        }

### Asynchronous Request Processing ( 异步请求处理 )
关于使用异步应答的动机,以及在什么时候,为什么要使用异步应答的解答:[传送门](https://spring.io/blog/2012/05/07/spring-mvc-3-2-preview-introducing-servlet-3-async-support),
异步应答的两种方式

* 1..  返回 Callable<?>

        @RequestMapping(method=RequestMethod.POST)
        public Callable<String> processUpload(final MultipartFile file) {
              return new Callable<String>() {
                  public String call() throws Exception {
                      // ...
                      return "someView";
                  }
              };
        }

* 2.. 返回 DeferredResult<?>

          @RequestMapping("/quotes")
          @ResponseBody
          public DeferredResult<String> quotes() {
              DeferredResult<String> deferredResult = new DeferredResult<String>();
              // Save the deferredResult in in-memory queue ...
              return deferredResult;
          }

          // In some other thread...
          deferredResult.setResult(data);

两种方式最主要的差别是,执行异步处理的线程是由谁控制, 第一种情况,由SpringMVC控制,第二种请求,有应用自己来控制

第一种返回Callable<?>的处理流程(2,3,4的执行顺序依赖各个线程的执行速度):
1. controller返回Callable
2. springMVC开启一个异步处理提交Callable到TaskExcutor中,在另外一个线程中处理
3. DispatcherServlet 和 所有的 filter 都退出这个请求处理的线程,但是保存输出流为打开状态
4. Callable处理完成,产生一个结果,springMVC分派这个请求回Servlet容器中.
5. DispacherServlet使用Callable产生的结果,再次执行和恢复处理

第二种返回 DeferredResult 的处理流程(2,3,4的执行顺序依赖各个线程的执行速度) :
1. controller返回DeferredResult, 并且将它放到某个内存队列中
2. springMVC开启异步处理
3. DispatcherServlet 和 所有的 filter 都退出这个请求处理的线程,但是保存输出流为打开状态
4. 应用线程,设置DeferredResult的结果,SpringMVC重新分派这个请求进Servlet容器
5. DispacherServlet使用DeferredResult中的结果,再次执行和恢复处理

### 异步应答的异常处理
异步应答的异常处理同普通的同步应答异常处理一样,由`@ExceptionHandler`标记的方法 或者 `HandlerExceptionResolver` 处理.
如果使用 DeferredResult的方式,则 可以自行调用`setErrorResult(Object)`设置错误结果,或者与前面的处理一样

### Intercepting Async Requests 异步请求的拦截器
可通过实现 `AsyncHandlerInterceptor` 接口来处理
或者实现 `CallableProcessingInterceptor` 或者 `DeferredResultProcessingInterceptor` 接口来实现拦截

## 使用异步请求处理时,web.xml的配置

          <web-app xmlns="http://java.sun.com/xml/ns/javaee"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          http://java.sun.com/xml/ns/javaee
          http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
          version="3.0">
                ...
          </web-app>

DispatcherServlet 和 所有的 filter 都需要配置:

        <async-supported>true</async-supported>

完整配置 例子:

        <web-app xmlns="http://java.sun.com/xml/ns/javaee"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="
              http://java.sun.com/xml/ns/javaee
              http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
              version="3.0">
              <filter>
                  <filter-name>Spring OpenEntityManagerInViewFilter</filter-name>
                  <filter-class>org.springframework.~.OpenEntityManagerInViewFilter</filter-class>
                  <async-supported>true</async-supported>
              </filter>

              <filter-mapping>
                  <filter-name>Spring OpenEntityManagerInViewFilter</filter-name>
                  <url-pattern>/*</url-pattern>
                  <dispatcher>REQUEST</dispatcher>
                  <dispatcher>ASYNC</dispatcher>
              </filter-mapping>
        </web-app>

## 关于springMVC的测试
[传送门](http://docs.spring.io/spring/docs/4.1.9.RELEASE/spring-framework-reference/htmlsingle/#spring-mvc-test-framework)
