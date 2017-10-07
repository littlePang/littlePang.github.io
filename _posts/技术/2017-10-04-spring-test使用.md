---
layout: post
title: spring-test使用
category: 技术
tags: spring
keywords:
description:
---

## spring test 介绍
在软件开发整个生命周期中测试都是一个重要的部分。

## 单测（Unit Testing）
相比于传统的Java EE开发中，依赖注入使得你的代码更少的依赖于容器。POJO使得你的应用程序应该是可以使用JUnit或者TestNG,通过new操作进行实例化，从而进行测试的。你可以使用mock对象在隔离的环境中测试你的代码。

## Mock Objects
在包 `org.springframework.mock.env` 中包含 `Environment` `PropertySource` 抽象的mock实现。
在包 `org.springframework.mock.jndi` 中包含JNDI相关的mock对象
在包 `org.springframework.mock.web` 包含Servlet API相关的mock对象。在测试web上下文，controller，filters的时候是非常有用的。这些mock对象的目标是spring web mvc框架，比使用 EasyMock或者MockObjects更加方便。spring 5.0 之后，`org.springframework.mock.web` 包下都是基于 Servilet 4.0 API 的。

# 单测支持类 （Unit Testing support Classes）

## 通用测试工具类（General testing utilities）
`org.springframework.test.util` 这个包下包含一系列在单测和整合测试中的通用工具。(从Json串中提取值 https://github.com/json-path/JsonPath )

`ReflectionTestUtils` 是一个基于反射的工具方法集合。开发者可以在测试场景时水用这些方来修改常量的值，设置一个非public的域，执行一个非public的setter方法，或者执行一个非public的配置或者生命周期方法回调方法，
在测试的应用代码涉及使用例如如下的情况时：

* ORM 框架（例如JPA和Hibernate）使用private或者protected修饰属性访问，并通过public的setter方法设置对象属性。
* Spring 的例如 @Autowired @Inject和@Resource，他们对private或者protected属性提供依赖注入。
* 使用例如 @PostConStruct 和 @PreDestroy 的生命周期回调方法。

`AopTestUtils` 是AOP相关的工具方法集合。这些方法可以获取一个隐藏在多个Spring代理对象的目标对象。Spring Aop相关的工具类为 `AopUtils` 和 `ApoProxyUtils`.

## Spring MVC

`org.springframework.test.web` 这个包下，包含 `ModelAndViewAssert`，你可以结合 Junit，TestNG或者其他测试框架在单测试处理 ModelAndView 对象。

### Spring MVC 控制器单测
使用 `ModelAndViewAssert` 混合 `MockHttpServletRequest`,`MockHttpSession` 和其它的 Servlet API mock对象（org.springframework.mock.web 包下）。如果要连接`WebApplicationContext`的配置，整合spring MVC 和 Rest Controller, 需要使用 Spring MVC Test Franmwork 框架。

# 集成测试

在不部署应用程序或者连接其他企业组件时进行一些集成测试是非常重要的，例如以下一些测试的东西：

* 正确的写入Spring IoC 容器上下文
* 使用JDBC获取ORM工具进行数据访问。包括例如SQL的正确性，Hibernate查询，JPA 实体映射等。

Spring框架提供spring-test模块对集成测试提供一流的支持，'org.springframework.test' 包下 包含使用Spring容器进行集成测试非常有用的类。这个测试不依赖应用服务器或者其他的部署环境。它的运行速度比单测要慢，但是比需要远端部署的测试要快。

在spring 2.5 及后续版本，单测和集成测试提供基于注解的测试（[Spring TestContext Framework](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-framework)）。TestContext测试框架对真正使用的测试框架是不知道的，所以可以使用 JUnit，TestNG等进行测试。

## 集成测试的目的

Spring的集成测试支持主要包括以下一些目标：
* 在测试执行时管理Spring IoC容器缓存（[Spring IoC container caching](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-ctx-management)），ApplicationContext加载一次，即可在同一组的测试case中重复使用。
* 支持测试实例的依赖注入（[Dependency Injection of test fixture instances](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-fixture-di)），Spring上下文中的bean也可注入进测试类中。
* 在集成测试中支持事务管理([transaction management](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-tx))，默认情况下框架会为每一个测试都创建并回滚事务，从而达到不影响数据的效果。测试代码可以假设都有一个事务存在。
* 给开发者在写整合测试时提供一些 Spring基础类([Spring-specific base classes](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-support-classes))

## JDBC测试支持

‘org.springframework.test.jdbc’ 包下包含的 `JdbcTestUtils` 包含一些静态工具类方法（例如统计表行数，删除记录等）

注意：在 `AbstractTransactionalJUnit4SpringContextTests ` 和 `AbstractTransactionalTestNGSpringContextTests` 中也提供了这些基础操作。

`spring-jdbc`模块提供了对内嵌数据库的支持详情查看([Embedded database support](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#jdbc-embedded-database-support) 和 [Testing data access logic with an embedded database](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#jdbc-embedded-database-dao-testing))

## 注解

### Spring 测试类注解

`@BootstapWith` : 类级别的注解，用来配置 Spring测试框架的引导启动。需要注意的是，`@BootStrapWith` 是用来指定一个自定义的 `TestContextBootstrapper`的。详情查看 [Bootstrap the TestContext framework](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-bootstrapping)。

`@ContextConfiguration` ： 类级别注解，用来指定配置文件路径（如果是基于注解的容器启动，则指定起 Configuration的类），此类为可继承的，所以可以将其写在父类上。

`@WebAppConfiguration`: 类级别注解，用来指定集成测试的上下文应该是个 `WebApplicationContext`,默认使用路径为 `file:src/main/webapp`,也可自行指定。（注意 该 注解必须与 `@ContextConfiguration` 一起使用）

        @ContextConfiguration
        @WebAppConfiguration
        public class WebAppTests {
                // class body...
        }

`@ContextHierarchy` : 类级别注解，用来定义 ApplicationContext 的层级，这个注解应该要生命一个或多个 @ContextConfiguration 实例。（可用于父子bean实例覆盖），详情请看：[Context hierarchies](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-ctx-management-ctx-hierarchies)

        @ContextHierarchy({
                @ContextConfiguration("/parent-config.xml"),
                @ContextConfiguration("/child-config.xml")
        })
        public class ContextHierarchyTests {
                // class body...
        }

`@ActiveProfiles` 指定使用的环境，详情看：[ Context configuration with environment profiles ](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-ctx-management-env-profiles)

        @ContextConfiguration
        @ActiveProfiles("dev")
        public class DeveloperTests {
                // class body...
        }        


`@TestPropertySource` ： 类级别注解，用来指定额外的配置属性，将其加入 `PropertySources` 和  `Environment`中。

        @ContextConfiguration
        @TestPropertySource(properties = { "timezone = GMT", "port: 4242" })
        public class MyIntegrationTests {
                // class body...
        }

`@DirtiesContext`：类和方法级别注解，用来表示 在经过这个类或者方法是，ApplicationContext 就被污染了，不应该再被缓存，应该要移除。

`@TestExecutionListeners` : 类级别注解。用来向 `TestContextManager` 注册监听器。通常和 `@ContextConfiguration`联合使用。

`@Commit` ： 表示测试方法执行玩后，需要将事务提交

`@Rollback` : 配置测试方法执行玩后，是否需要将事务回滚。

`@BeforeTransaction` ： 标注在事务启动前执行的方法。

`@AfterTransaction` ： 标注在事务执行完成后执行的方法。

`@Sql` : 用来配置在测试开始之前或者结束之后需要执行的sql脚本，通过注册 SqlScriptsTestExecutionListener 实现。

`@SqlConfig` ： 用来定义如何解析 `@Sql`指定的sql脚本

          @Test
          @Sql(
                  scripts = "/test-user-data.sql",
                  config = @SqlConfig(commentPrefix = "`", separator = "@@")
          )
          public void userTest {
                  // execute code that relies on the test data
          }

`@SqlGroup` ： 定义 sql 组：

          @Test
          @SqlGroup({
                  @Sql(scripts = "/test-schema.sql", config = @SqlConfig(commentPrefix = "`")),
                  @Sql("/test-user-data.sql")
          )}
          public void userTest {
                  // execute code that uses the test schema and test data
          }

并且 测试框架支持所有的标准注解 （@Autowired （通过注册 DependencyInjectionTestExecutionListener 实现）, @Resource 等）.    有些注解，例如 `@PostConstruct`有些需要特别说明的，看这里：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#integration-testing-annotations-standard) 。

### Spring JUnit4的测试注解

`@IfProfileValue` 表示只有在 属性 name 的值等于 value 时才执行此测试，否则忽略：

          @IfProfileValue(name="test-groups", values={"unit-tests", "integration-tests"})
          @Test
          public void testProcessWhichRunsForUnitOrIntegrationTestGroups() {
                  // some logic that should run only for unit and integration test groups
          }   

`@ProfileValueSourceConfiguration` ： 类级别注解，和 `@IfProfileValue`  配合使用，如果未配置，则默认使用 `SystemProfileValueSource`

`@Timed` 设置测试超时时间

`@Repeat` 设置重试次数

还有些不咋用的注解，看这里：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#integration-testing-annotations-junit-jupiter).

所有的注解都支持继承，所以可像如下方式，简写配置：

            @RunWith(SpringRunner.class)
            @ContextConfiguration({"/app-config.xml", "/test-data-access-config.xml"})
            @ActiveProfiles("dev")
            @Transactional
            public class OrderRepositoryTests { }

            @RunWith(SpringRunner.class)
            @ContextConfiguration({"/app-config.xml", "/test-data-access-config.xml"})
            @ActiveProfiles("dev")
            @Transactional
            public class UserRepositoryTests { }

将注解全部合并为一个注解进行使用：

          @Target(ElementType.TYPE)
          @Retention(RetentionPolicy.RUNTIME)
          @ContextConfiguration({"/app-config.xml", "/test-data-access-config.xml"})
          @ActiveProfiles("dev")
          @Transactional
          public @interface TransactionalDevTestConfig { }

则可将上的配置简写为：

        @RunWith(SpringRunner.class)
        @TransactionalDevTestConfig
        public class OrderRepositoryTests { }

        @RunWith(SpringRunner.class)
        @TransactionalDevTestConfig
        public class UserRepositoryTests { }


关于测试框架的一些扩张说明，[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-framework).


#### Working with Web Mocks

为了提供综合的web测试支持，Spring3.2默认提供并开启 `ServletTestExecutionListener` 进行支持。在测试 `WebApplicationContext` 的时候，`TestExecutionListener` 会设置一个默认的thread-localde的 `RequestContextHolder`，在每个测试方法运行前，创建一个 `MockHttpServletRequest`，`MockHttpServletResponse`和 `ServletWebRequest`.

一旦你在测试中加载了 `WebApplicationContext` 则其他相关的mock类也都可使用，例如在设置一些在web组件中的前置校验。以下是一个使用的例子，需要注意的是 `WebApplicationContext` 和 `MockServletContext` 是在整个测试组中缓存起来的。其他的mock对象由 `ServletTestExecutionListener`管理。

          @WebAppConfiguration
          @ContextConfiguration
          public class WacTests {

                  @Autowired
                  WebApplicationContext wac; // cached

                  @Autowired
                  MockServletContext servletContext; // cached

                  @Autowired
                  MockHttpSession session;

                  @Autowired
                  MockHttpServletRequest request;

                  @Autowired
                  MockHttpServletResponse response;

                  @Autowired
                  ServletWebRequest webRequest;

                  //...
          }

#### 上下文的缓存

多个 test 之间的 ApplicationContext 或者 WebApplicationContext 的复用是基于 配置文件地址，activeProfiles 等配置的唯一性约束确定的，及相同配置的上下文，可进行复用。具体信息看这里：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-ctx-management-caching)          

#### 上下文的继承

[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-ctx-management-ctx-hierarchies)

#### 测试作用域为request或者session级别的bean

[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-web-scoped-beans)


### 事务管理

在测试框架中，事务由 `TransactionTestExecutionListener` 管理，如果在测试类上没有申明 `@TestExecutionListeners`, 则它是默认开启的。但是，你必须要在 `ApplicationContext` 中配置了 `PlatformTransactionManager`，此外，在测试类或者方法上使用了 `@Transactional`

事务管理测试抽象类: AbstractTransactionalJUnit4SpringContextTests, 使用：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-tx-programmatic-tx-mgt)

### 执行SQL脚本

`spring-jdbc`模块提供初始化内嵌数据库([Embedded database support](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#jdbc-embedded-database-support))或者执行SQL脚本的支持([Testing data access logic with an embedded database](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#jdbc-embedded-database-dao-testing))。 也可使用 `@Sql` 执行脚本

spring提供以下方法可在集成测试中使用编程的方式执行脚本初始化

* org.springframework.jdbc.datasource.init.ScriptUtils
* org.springframework.jdbc.datasource.init.ResourceDatabasePopulator
* org.springframework.test.context.junit4.AbstractTransactionalJUnit4SpringContextTests
* org.springframework.test.context.testng.AbstractTransactionalTestNGSpringContextTests

使用 `@Sql` 执行脚本时的路径语意：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-executing-sql-declaratively)

### 并行执行测试

Spring5.0 支持并行测试 ：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-parallel-test-execution)

### 测试框架支持的类

@RunWith(SpringJUnit4ClassRunner.class) 和 @RunWith(SpringRunner.class) 是一样的。如果想使用其他还有一些 第三方框架提供的测试runner(例如：MockitoJUnitRunner)，可查看 Spring提供的JUnit支持进行替换（[Spring’s support for JUnit rules](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-junit4-rules)）。

在包 `org.springframework.test.context.junit4.rules` 下，spring提供了一些Junit4的一些规则支持

* SpringClassRule
* SpringMethodRule

可使用这样方式，选择其他的非Spring提供的runner：

            // Optionally specify a non-Spring Runner via @RunWith(...)
            @ContextConfiguration
            public class IntegrationTest {

               @ClassRule
               public static final SpringClassRule springClassRule = new SpringClassRule();

               @Rule
               public final SpringMethodRule springMethodRule = new SpringMethodRule();

               @Test
               public void testMethod() {
                  // execute test logic...
               }
            }

使用 JUnit Jupiter的一些扩展：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-junit-jupiter-extension)

## Spring MVC 测试框架

测试框架提供一套可以在JUnit，TestNG 或者其他测试框架中使用的流式编程接口 ,它都是建立在 spring-test模块中mock对象上的 ([ Servlet API mock objects](https://docs.spring.io/spring-framework/docs/5.0.0.RELEASE/javadoc-api/org/springframework/mock/web/package-summary.html)).因此不要运行Servlet容器。

### 服务端的测试

测试的例子，请查看这里 : [传送门](https://github.com/spring-projects/spring-framework/blob/master/spring-test/src/test/java/org/springframework/test/web/servlet/samples/context/XmlConfigTests.java).

使用详情：[传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#spring-mvc-test-server)

Html请求的单测 ： [传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#spring-mvc-test-server)

# 其他资源

一些测试框架的资源说明: [传送门](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-resources)
