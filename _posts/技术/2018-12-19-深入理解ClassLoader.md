---
layout: post
title: 深入理解ClassLoader
category: 技术
tags: Java
keywords:
description:
---

# 说明

本文为翻译文，原文地址：[传送门](https://zeroturnaround.com/rebellabs/rebel-labs-tutorial-do-you-really-get-classloaders/)

# 第一部分：ClassLoader 概述，委派和常见问题

这一部分，我们会对ClassLoader做一个概述，解释 委派是如何工作的，以及ClassLoader常见问题的解决方案。

### 引言: 为什么我们应该知道和敬畏 ClassLoader

类加载器 是 Java语言的核心，Java EE 容器，OSGI ，各种Web框架 和 其他的工具 都大量的使用了ClassLoader，在类加载失败的时候，你知道怎么解决它么？

我们将从 JVM 和 开发者两方面来讲述 Java的类加载机制。还有和类加载的常见问题以及如何解决他们。NoClassDefFoundError，LinkageError 和 一些其他的错误，你通常都能解决它。每个问题，我们都提供了一个 例子和相关的解决方案。我们也会看下为什么类加载器会泄漏，以及如何解决它。

最后我们会回顾下如何使用一个动态类加载器 重新载入一个Java类。为了达到这个目的，我们会看到 对象，类和类加载器是如何互相关联的，以及为了修改需要作出的改动。我们开始先从宏观视角解释重载过程，然后再从一个详细的例子来说明典型的问题和解决方案。

### 进入 java.lang.ClassCloader

现在让我们开启类加载机制的美妙世界吧。

非常重要的一点是意识到 每一个类加载器 其实都是 java.lang.ClassCloader 的子类的一个实例。每个类都是有这些实例之一进行加载的，并且开发者可以通过继承 java.lang.ClassCloader 在JVM中加载类。

这里可能会有一个小困惑：如果类加载器都有一个类，并且每个类都是有类加载器进行加载的，那么先有谁呢？（像 先有鸡还是先有蛋的问题）. 要知道这个问题，我们必须弄明白类加载器的机制和JVM类加载器继承体系

下面是类加载器的API，省略了不相关的部分：


          package java.lang;

          public abstract class ClassLoader {

            public Class loadClass(String name);
            protected Class defineClass(byte[] b);

            public URL getResource(String name);
            public Enumeration getResources(String name);

            public ClassLoader getParent()
          }

到目前为止 java.lang.ClassLoader 最重要的方法是 loadClass，它通过一个类的全限定加载并返回一个 Class 对象。

defineClass 方法用来为JVM实例化一个类对象。方法的参数是一个从磁盘或者其他任意位置加载的 类 的字节数组

getResource 和 getResources 返回的是一个指定的资源。它们是类加载约定中重要的一部分，并且它们和类加载器采用一样的方式来处理委派，即：先委派给父类尝试加载，然后再尝试在本地查到资源。我们甚至可以将类加载简单的等价于 `defineClass(getResource(name).getBytes())`。

getParent方法返回的是父加载器。在下一部分我们将会描述它的更多细节。

java的惰性加载影响了类加载器的工作方式 - 任何事都是在尽可能的在最后一刻才执行。一个类只会在其以某种方式被引用时才进行加载（调用构造器，静态方法或者静态域）。

现在然我们看下面的例子，类A实例化类B

        public class A {
        	  public void doSomething() {
          	 B b = new B();
           	 b.doSomethingElse();
        	  }
        	}

声明 `B b = new B()` 在语义上等价于 `B b = A.class. getClassLoader().loadClass(“B”).newInstance()`。

我们可以看到，在Java中 每个对象都关联了它属于的类(A.class) 并且 每个 类 都关联了 加载它的那个类加载器(A.class.getClassLoader())。

当我们实例化一个类加载器，我们可以通过构造器指定它的父加载器。如果父加载器没有指定，则父加载器默认是虚拟机的 系统类加载器(System Classloader)，接下来我们更加仔细的探究JVM的类加载器体系。

### 类加载器层次结构

无论何时，一个新启动的JVM都会首先创建一个 启动类加载器(Bootstrap Classloader)， 用来将核心类(java.lang包下的类)和其他一些运行时的类加载到内存中。启动类加载是其他所有类加载器的父加载器，因此，它是唯一一个没有父类加载器的类加载器。

接下来是 [扩展类加载器](https://docs.oracle.com/javase/tutorial/ext/index.html), 它以 启动类加载器 作为父加载器，用来加载 所有 `java.ext.dirs` 里所配置目录下的 jar 包。

从开发人员的角度来看，第三个非常重要的类加载器是 系统类加载器(System Classloader)，它的父加载器是扩展类加载器，用于加载 `java.class.path`或者`-classpath`中配置的路径下的jar包。

![](/assets/picture/2018-12-20_1.png)

需要注意的是这个类加载器的层次结构并不是一个继承体系，而是一个委派模式体系。大多数类加载器在加载一个类时，会优先委派给父类加载器进行加载，如果加载不到，再尝试从本身的类路径下加载类。实际上，一个类加载器的责任是加载那些父加载器无法加载的类。在类加载层级结构中，由较高层次结构的类加载器加载的类是无法引用由较低层次的类加载器加载的类的。

上面所说的类加载器的委派行为，是为了避免同一个类被多次加载。在1995年，Java的主要应用被认为是在客户端的Web小程序(Applet),在当时连接是很慢的，所以在网络上惰性加载程序是非常重要的。类加载器通过抽象每个类的加载方式开启这种行为。

但是历史证明Java在服务端更加有用。而且在JavaEE里，这个查找顺序常常是颠倒的：一个类加载器可能会优先在本地加载类，然后再去父加载器加载类。

### JavaEE的委派模型


下图是一个典型的应用容器类加载器层次结构：有一个容器的类加载器，每个 ear (Enterprise Archive, 主要用于对JavaEE应用程序进行打包,包含一个或多个war) 有一个自己的类加载器，每个 war 也有自己的类加载器。

![](/assets/picture/2018-12-20_2.png)

Java的Servlet规范推荐一个web模块的类加载器优先在本地加载类，如果找不到，再去父类加载器进行加载。

在某些应用容器中是遵循了上面的建议的，但是另外一些web模块的类加载器配置为遵循了正常的加载模型，所以明智的方法时提前阅读你所使用的应用程序容器的文档。

应用程序容器反转类本地查找和委派查找的顺序的原因是：每个应用程序都包含了大量的库，这些库的发布周期可能并不适用于开发人员。一个经典的例子就是 log4j ，应用程序容器发布的时候回包含一个版本，但是应用程序绑定的是另外一个版本。

现在，让我们看几个我们可能会遭遇的类加载的问题，以及其解决方案。

### 当程序出错的时候

JavaEE的委派模型在类加载时会造成一些有趣的问题。NoClassDefFoundError, LinkageError, ClassNotFoundException, NoSuchMethodError, ClassCastException 等异常在部署JavaEE应用的时候，都是非常常见的。上面的这些问题我们可以对它们做各种假设，但是更重要的是要去验证它们，这样我们才不会像个“笨蛋”。

### NoClassDefFoundError

NoClassDefFoundError是一个在部署JavaEE应用时非常常见的问题。分析和解决这个问题的难度依赖于你所部署的JavaEE中间件环境的规模。特别是在各种JavaEE应用中存在着大量的类加载器。

Java文档上说，如果java虚拟机或者类加载器实例尝试加载一个类的是找不到它的定义，则会抛出 NoClassDefFoundError，也即是说 在编译时是能找到所查找的类的定义的，但是在运行时无法找到它的定义。

这也是你不能完全认为IDE说没问题就是没问题 的原因之一，这是一个运行时的问题，IDE无法在这里帮助你。

让我们看个例子：

        public class HelloServlet extends HttpServlet {
           protected void doGet(HttpServletRequest request,
                                            HttpServletResponse response)
                                            throws ServletException, IOException {
               PrintWriter out = response.getWriter();
               out.print(new Util().sayHello());     
        }

HelloServlet实例化了一个Util的实例，用来支持信息打印，当请求执行时我们可能会看到如下的错误信息：

        java.lang.NoClassdefFoundError: Util
        	HelloServlet:doGet(HelloServlet.java:17)
        	javax.servlet.http.HttpServlet.service(HttpServlet.java:617)
        	javax.servlet.http.HttpServlet.service(HttpServlet.java:717)

我们应该怎么来解决这个问题呢？显然，你可能会去检查 Util 这个类是否包含在包里面。

这里有一个技巧，我我们可以让容器的类加载器说出它是从哪个资源去加载类的。那就是 我们需要将 HelloServlet 的类加载器转为 URLCLassLoader 并查询它的 classpath。

        public class HelloServlet extends HttpServlet {
           protected void doGet(HttpServletRequest request,
                                            HttpServletResponse response)
                                            throws ServletException, IOException {
               PrintWriter out = response.getWriter();
               out.print(Arrays.toString(
                   ((URLClassLoader)HelloServlet.class.getClassLoader()).getURLs()));
        }

上面的输出结果会像下面这样：

>file:/Users/myuser/eclipse/workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/demo/WEB-INF/classes, file:/Users/myuser/eclipse/workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/demo/WEB-INF/lib/demo-lib.jar        

`file:/Users/myuser/eclipse/workspace/.metadata/` 这个资源的路径 揭示了 容器是由 Eclipse 启动的，以及 IDE 解压文档进行部署的位置。现在我们可以检查缺少的 Util 类是否包含在 demo-lib.jar 或者 WEB-INF/classes 目录下了。

上面这个特定的例子中，出现错误的原因是：我们期望 Util 被打包到 demo-lib.jar 中，但是我们没有触发构建，所以这个类并不包含在上一个已经存在的包中。

URLClassLoader这个技巧可能并不是在所有的应用服务器上都可用，所以另一个查问题的方式是使用 jconsole等工具查到JVM加载资源的真实路径。下图是使用 jconsole 连接 JBoss 应用服务端进程查询classpath的的地方。

![](/assets/picture/2018-12-21_1.png)

## NoSuchMethodError

使用上一个例子，在另外一个场景下，我们可能会遭遇下面的异常：

        java.lang.NoSuchMethodError: Util.sayHello()Ljava/lang/String;
        	HelloServlet:doGet(HelloServlet.java:17)
        	javax.servlet.http.HttpServlet.service(HttpServlet.
        	java:617)
        	javax.servlet.http.HttpServlet.service(HttpServlet.

NoSuchMethodError 代表的是另一个不问题。在这个例子中，我们的引用使存在的，但是加载的是不正确的版本，因此请求的这个方法找不到。为了解决这个问题，我们首先要找到这个类是从哪里加载的，简单的方式是增加 `-verbose:class` JVM启动参数，也可以使用 `getResource()` 来查到类路径：

        public class HelloServlet extends HttpServlet {
           protected void doGet(HttpServletRequest request,
                                            HttpServletResponse response)
                                            throws ServletException, IOException {
               PrintWriter out = response.getWriter();
            out.print(HelloServlet.class.getClassLoader().getResource(
                   Util.class.getName.replace(‘.’, ‘/’) + “.class”));  

        }

假设上面这个例子的请求执行结果如下：
> file:/Users/myuser/eclipse/workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/demo/WEB-INF/lib/demo-lib.jar!/Util.class

现在我们需要去校验我们关于类的版本不正确的假设。我们可以使用 javap 工具来反编译类，然后我们可以看到我们请求的方法是否存在：

        $ javap -private Util
        Compiled from “Util.java”
        public class Util extends java.lang.Object {
           public Util();
        }

可以看见，在这个版本的`Util`类中并没有 `sayHello` 这个方法，可能使我们没有重新对 demo-lib.jar 进行打包，所以它不包含这个新方法。

### 错误的变体

NoClassdefFoundError 和 NoSuchMethodError 是在处理JavaEE应用时，非常典型的问题。它要求开发人员明白这些问题的原因，以及拥有解决这些问题的技能。

这两个错误还有非常多的变体，例如: AbstractMethodError, ClassCastException, IllegalAccessError。基本上遇到这些问题的时候，我们都可以认为是应用使用的版本和真实加载的类的版本不一致。

### ClassCastException

然我们来论证一个 `ClassCastException` 的例子。我们修改初始化的那个例子，增加 工厂类来获取 Util 类。

        public class HelloServlet extends HttpServlet {
           protected void doGet(HttpServletRequest request,
                                            HttpServletResponse response)
                                            throws ServletException, IOException {
               PrintWriter out = response.getWriter();
            out.print(((Util)Factory.getUtil()).sayHello());
        }

        class Factory {
             public static Object getUtil() {
                  return new Util();
             }
        }


这个方法，当请求过来是，可能会得到 ClassCastException：

        java.lang.ClassCastException: Util cannot be cast to Util
        	HelloServlet:doGet(HelloServlet.java:18)
        	javax.servlet.http.HttpServlet.service(HttpServlet.			java:617)
        	javax.servlet.http.HttpServlet.service(HttpServlet.

这个的意思是 HelloServlet 和 Factory 类在不同的上下文进行操作。我们已经明白了类是如何加载的。让我们使用 `-verbose:class` 来搞清楚 HelloServlet 和 Factory 都是从哪里加载的Util类。

>[Loaded Util from file:/Users/ekabanov/Applications/ apache-tomcat-6.0.20/lib/cl-shared-jar.jar]
[Loaded Util from file:/Users/ekabanov/Documents/workspace-javazone/.metadata/.plugins/org.eclipse.wst. server.core/tmp0/wtpwebapps/cl-demo/WEB-INF/lib/cl-demo-jar.jar]

所以 Util 类是由两个不同的类加载器从不同的地方进行加载的。一个是web应用类加载器，另一个是应用容器类加载器。

![](/assets/picture/2018-12-21_2.png)

但是为什么它们是不兼容的呢？在最开始的时候每个java中的每个类都是使用一个唯一的全限定名来定义的。但是在1997年发布的一个论文暴露出了这种仅使用名字的方式来定义类的严重的安全问题：它导致沙箱应用(例如applet)可以定义包括 `java.lang.String`在内的任何类，并将它们从沙箱外部注入到沙箱里。

这个问题的解决方案是通过全限定名和类加载器联合来定义一个类！也就是说类加载器A加载的Util类和类加载器B加载的Util对JVM来说是不一样的，所以它无法进行转换。

这个问题的根本原因在于web类加载器的加载反转行为(反双亲委派模型)。如果web类加载器的类加载行为和其他类加载器一致，那Util追呗应用容器类加载器加载一次，也就不会发送 ClassCastException 了。

### LinkageError

事情还能更加糟糕，让我们稍微修改上一个例子中的Factory类，使它返回Util而不是Object：

        class Factory {
             public static Util getUtil() {
                  return new Util();
             }
        }

这样做的结果就是，执行时会发生 LinkageError：

        ClassCastException:
        java.lang.LinkageError: loader constraint violation: when resolving method Factory.getUtil()LUtil; <…> HelloServlet:doGet(HelloServlet.java:18) javax.servlet.http.HttpServlet.service(HttpServlet. java:617) javax.servlet.http.HttpServlet.service(HttpServlet. java:717)

这个问题和 ClassCastException 唯一的差别是我们不再强转object对象了，但是由于加载器的约束就会发生LinkageError。

在处理类加载器的问题是有一个重要的原则是，类加载的行为往往会与你的常识不同，所以验证你的假设非常重要。例如，发生LinkageError时，查看代码或者构建过程反而会阻碍你发现问题的真相。这个问题的关键是你需要确切的知道类具体是从哪里加载的，它为何会从那里加载，以及如何预防它再次发生。

一个常见的原因是，当在不同地方依赖了同一个包的不同版本时，相同的类出现在了不止一个类加载器中（例如应用服务器和web应用）。比较经典的发生情况就是例如log4j和hibernate这个中标准库。在上面那个例子中的解决方案是 要么从web应用中去掉包的绑定，要么就是异常小心不要使用那些出现父类加载器中使用的类。

### IllegalAccessError

事实证明，不仅类是由全限定名和类加载器来定义的，包(packages)也是如此。为了证明它，让我们将 Factory.getUtil 方法修改回默认的样子：

        class Factory {
             static Object getUtil() {
                  return new Util();
             }
        }

假设 HelloServlet 和 Factory 在同一个包下，则HelloServlet中是可以访问 getUtil 的。不幸的是，如果我们在运行时访问它，我们会得到 IllegalAccessError异常：

        java.lang.IllegalAccessError: tried to access method Factory.getUtil()Ljava/lang/Object;
        HelloServlet:doGet(HelloServlet.java:18)
        javax.servlet.http.HttpServlet.service(HttpServlet.java:617)
        javax.servlet.http.HttpServlet.service(HttpServlet.java:717)        

虽然上面修改的的访问方式在编译时是正确的，但是两个类在运行时由不同的类加载器加载，所以在运行时会失败。造成失败的原因和类一样，为了一些安全的原因，包也由它的全限定名和类加载器来唯一确定。

ClassCastException,LinkageError 和 IllegalAccessError 之间的实现有一些细微的差别，但是造成它们的根本原因是一样的：类由不同的类加载器进行了加载。


### 类加载器菜单

列出了常见的问题，以及帮助定位的方法：

| | 找不到类(No class found) | 找到了错误的类(Wrong class found) | 找到了不止一个类(More than one class found) |
|--|---|--|---|
| 变体 | ClassNotFoundException <br> NoClassDefFoundError | IncompatibleClassChangeError <br> AbstractMethodError <br> NoSuch(Method &#124; Field)Error <br> ClassCastException <br> IllegalAccessError | LinkageError (class loading constraints violated) <br> ClassCastException <br>  IllegalAccessError |
| 定位方式 | 1. IDE class lookup (Ctrl+Shift+T in Eclipse) <br> 2. find *.jar -exec jar -tf ‘{}’\; &#124; grep MyClass  <br> 3. URLClassLoader.getUrls() Container specific logs | 1. -verbose:class <br> 2. ClassLoader.getResource() javap -private MyClass | 1. -verbose:class <br> 2. ClassLoader.getResource() |

# 第二部分：在web应用中的动态类加载器

在第二部分，我们将看到在web应用和模块系统中如果使用动态类加载器的，以及它们能造成多么令人头痛的问题，最后我们讨论如何做的更好。

### 动态类加载

在我们讨论动态类加载时，我们需要弄明白的第一件事就是类和对象的关系。所有的Java代码都和类中的方法关联。简单的来说，你可以认为类是方法的集合，这些方法都接受`this`作为它的第一个参数。类和他所有的方法都被加载到内存中，并接受一个唯一的身份。在Java API中，这个唯一的身份由一个 `java.lang.Class` 的实例来表示，并且你可以通过 MyObject.class 来访问它。

通过 `Object.getClass()`方法每个对象都可以创建一个范围这个唯一身份的引用。当调用某个对象的一个方法时，JVM会在类引用中查到并调用该特定类的方法。也就是说，当你调用 `mo.method()`(这里mo是MyObject的一个实例)时，在语义上等于 `mo.getClass().getDeclareMethod("method").invoke(mo)` （JVM的真实情况并不是这么做的，但是结果是一样的）。

每个类对象都是与其类加载器相关联的(`MyObject.class.getClassLoader()`)。类加载器主要的作用是定义类的范围，即：这个类在哪里是可见的，在哪里是不可见的。这个作用域允许相同全限定名的类存在，只要它们是在不同的类加载器中加载的。同时它也允许在不同的类加载器中加载一个新的版本。

![](/assets/picture/2018-12-22_1.png)

在Java中进行类重新载入的主要问题是，虽然你可以加载一个类的新版本，它拥有一个完全不同的身份，但是会有已经存在的对应持有了这个类上一个版本的引用。所以当这些对象在执行方法的时候，它实际上执行的是老版本的方法。

假设我们加载了 MyObject 类的一个新版本。老版本记为 MyObject_1，新版本记为 MyObject_2。同时也假设 MyObject_1 的 MyObject.method() 返回 “1”，MyObject_2 的 MyObject.method() 返回 “2”，现在如果有一个 MyObject_2 的实例 mo2，则：

* mo.getClass() != mo2.getClass()
* mo.getClass().getDeclareMethod("method").invoke(mo)
* mo2.getClass().getDeclareMethod("method").invoke(mo2)
* mo.getClass().getDeclaredMethod(“method”).invoke(mo2) 会抛出 ClassCastException，因为mo和mo2的身份是不匹配的。

### Down & Dirty

接下来让我们看看代码。记住，我们是在尝试在不同的类加载器中加载类的一个新版本。我们使用 Counter 的代码如下：

        public class Counter implements ICounter {
          private int counter;



        public String message() {
            return "Version 1";
          }
          public int plusPlus() {
            return counter++;
          }
          public int counter() {
            return counter;
          }
        }

我们使用一个 main() 方法无限循环的输出 Counter 类的信息。同时我们还需要两个Counter类的实例：counter1 在开始的位置创建一次，counter2在每次循环开始的位置创建。

        public class Main {
          private static ICounter counter1;
          private static ICounter counter2;

          public static void main(String[] args)  {
            counter1 = CounterFactory.newInstance();

            while (true) {
              counter2 = CounterFactory.newInstance();

              System.out.println("1) " +
                counter1.message() + " = " + counter1.plusPlus());
              System.out.println("2) " +
                counter2.message() + " = " + counter2.plusPlus());
              System.out.println();

              Thread.currentThread().sleep(3000);
            }
          }
        }

`ICounter` 是一个包含`Counter`所有方法的接口。这个接口是必须的，因为我们会在两个隔离的类加载器里加载Counter，所以Main不能直接使用它(否则我们会得到ClassCastException)。

        public interface ICounter {
          String message();
          int plusPlus();
        }

这个例子中，你可能会惊讶创建一个类加载器是如此的简单。如果我们移除移除处理部分，则可归结为如下这样：

        public class CounterFactory {
          public static ICounter newInstance() {
            URLClassLoader tmp =
              new URLClassLoader(new URL[] {getClassPath()}) {
                public Class loadClass(String name) {
                  if ("example.Counter".equals(name))
                    return findClass(name);
                  return super.loadClass(name);
                }
              };

            return (ICounter)
              tmp.loadClass("example.Counter").newInstance();
          }
        }

上面这段代码中 `getClassPath()` 这个方法的目的是返回一个硬编码的类路径。但是我们也可以使用 `ClassLoader.getResource()`API来自动化处理这个过程。现在让我们执行下 Main.main(), 然后看下输出。

>1) Version 1 = 3

>2) Version 1 = 0

作为我们期待的结果，第一个实例的计数器是被更新的，第二计数器一直是“0”。如果我们修改 `Counter.message()`使其返回“Version 2”。则输出会变为：

>1) Version 1 = 4

>2) Version 1 = 2

我们可以看到第一个实例继续增加了计数器，但是使用的是老版本的类。第二个实例的类被更新了，但是所有的状态都丢失了。

为了修正它，我们尝试为第二个实例重建状态。我可以通过从上一个计数器中复制状态来达到这个目的。

首先我们增加一个 `copy()` 方法到Counter类中，参数为对应的接口：

        public ICounter copy(ICounter other) {
            		if (other != null)
              		counter = other.counter();
            		return this;
          	   }

接下来，我们更新 Main.main() 方法创建第二个实例的地方为：

        counter2 = CounterFactory.newInstance().copy(counter2);

重新执行，等待几秒后，输出：

>1) Version 1 = 3

>2) Version 1 = 3

然后我们修改 Counter.message() 方法 使之返回 "Version 2", 可见输出：

>1) Version 1 = 4

>2) Version 2 = 4

你可以看到，尽管 第二个实例被更新了，但是它所有的状态都被保存下来了，它涉及了手动管理状态。不幸的是，Java API 无法更新已经存在的对象的类，也不能可靠的复制其状态，所以我们不得不求助于更复杂的解决方案。

### 类加载器如何发生泄漏（How Do ClassCloader Leaks Happen） ?

如果你使用java变成，那你肯定知道java有时会发送内存泄漏。通常泄漏发生的原因是：应该被清理掉的对象的集合(例如:listeners)没有清理掉。类加载器在这种情况中非常特殊，并且不幸的是，当前Java平台的状态为，这些泄漏是昂贵而又不可避免的：在几次重新载入后，通常就会发生 OutOfMemoryError 了。

我们可以看到之前的每个对象都引用了它所属的类，也就是说它引用着它的类加载器。但是我们还没说过每个类加载器都反过来引用着每个它所加载的类，每个类加载器都将它们持有在一个静态域中。

![](/assets/picture/2018-12-22_2.png)

这也就意味着：
1. 如果一个类加载器泄漏了，它会在静态域中持有所有它加载的类。静态域中通常持有缓存，单例对象和何种各样的配置和应用状态。即使你的应用没有任何比较大的静态缓存，也不意味着你所使用的框架不会为你持有(例如 Log4j 就是最长久的罪魁祸首，它通常会被放在还几个服务器的classpath下)。这也就解释了为什么类加载器的泄漏如此的昂贵。
2. 只要持有一个由某个类加载器加载的类实例化的实例引用，就足以使这个类加载器发送泄漏了。即使这个对象看起来是无害的(例如一个域也没有),它仍然会持有它的类加载器和应用的所有状态。应用在重新载入时只要有一个地方没有清理干净，则他就会发生泄漏。在一个经典的应用的，通常会有好几个这种地方，有些是无法被修复的，因为它们是第三方构建的包。因此类加载器泄漏是如此的常见。

### 泄漏的介绍

我是使用上面的Main类来做一个简单的泄漏的例子：

        public class Main {
        private static ICounter counter1;
          private static ICounter counter2;

          public static void main(String[] args)  {
            counter1 = CounterFactory.newInstance().copy();

            while (true) {
              counter2 = CounterFactory.newInstance().copy();

              System.out.println("1) " +
                counter1.message() + " = " + counter1.plusPlus());
              System.out.println("2) " +
                counter2.message() + " = " + counter2.plusPlus());
              System.out.println();

              Thread.currentThread().sleep(3000);
            }
          }
        }

CounterFactory和之前的是一样的。但是这里有其他东西来导致泄漏，下面介绍一个叫 Leak 的新类和它相关的接口 ILeak:

        interface ILeak {
        }

        public class Leak implements ILeak {
          private ILeak leak;

          public Leak(ILeak leak) {
            this.leak = leak;
          }
        }

你可以看到，这不是一个特别复杂的类：由一个对象链表组成，它仅仅持有了上一个对象的引用。我们修改Counter类，使之包含Leak对象，并且持有一个大数组使内存上升(模拟一个大缓存)。为了简单起见，我们省略掉前面相同的方法：

        public class Counter implements ICounter {
          private int counter;
          private ILeak leak;

          private static final long[] cache = new long[1000000];

          /* message(), counter(), plusPlus() impls */

          public ILeak leak() {
            return new Leak(leak);
          }

          public ICounter copy(ICounter other) {
            if (other != null) {
              counter = other.counter();
              leak = other.leak();
            }
            return this;
          }
        }

对于这个 Counter 需要注意的两件事情是：
1. Counter持有可一个Leak的引用，但是Leak并没有引用Counter。
2. 当Counter被复制时(调用copy()方法)。会创建并持有一个新的包含上一个Leak的Leak对象。

如果你尝试运行这段代码，则经过几个迭代之后，则会抛出 OutOfMemoryError：

        Exception in thread "main" java.lang.OutOfMemoryError: Java heap space at example.Counter.(Counter. java:8)

使用正确的工具，我们可以深入探究，看到它是如何发生的。


### 泄漏分析

java 5.0 之后可以使用JDK提供的 jmap这个命令行工具导出一个运行时应用的堆内存信息。但是如果应用崩溃，则我们需要另外一个 java 6.0 提供的特性：在 OutOfMemoryError 的时候转储堆内存信息，你只需要在命令行加入 `-XX:+HeapDumpOnOutOfMemoryError`即可：

        java.lang.OutOfMemoryError: Java heap space
        Dumping heap to java_pid37266.hprof ...
        Heap dump file created [57715044 bytes in 1.707 secs]
        Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at example.Counter.(Counter.java:8)

我们获取了堆信息之后就可以分析它了。这里有很多工具可以使用，包括JDK发布的jhat，这里作者使用的是 Eclipse Memory Analyzer (MAT) 来分析，其实还可以使用 JProfiler.

在上面那个内存泄漏的例子中：每一个Leak对象都泄漏了，并且它们还持有着它们的类加载器，每个类加载器也持有着它加载的Counter类：

![](/assets/picture/2018-12-24_1.png)

这个例子看起来略微显得不自然，它的主要想法是在java中移除一个对象，但是很容易造成泄漏。在类进行重新载入或者创建一个新类加载器时，leak对象隐式的泄漏了整个类加载器。预发内存泄漏是非常有挑战性的，并且事后解决这些泄漏问题，也不能保证不引入新的泄漏问题。

### JavaEE Web应用程序的动态类加载

为了运行JavaEE的web应用。需要将归档文件打包为一个 .war 的包，并且使用 像tamcat一样的servlet容器进行部署。这在生产环境中是有意义的，因为给出了一种收集和部署应用的简单方式，但是当部署应用的时候，你通常只想编辑应用的文件并且在浏览器中查看文件生效后的改变。

我们已经验证了如何动态的重新加载类。这一部分，我们将看到服务器和框架如何使用动态类加载加速部署周期。这里我们使用apache tomcat 进行演示。

为了使用动态类加载，我们必须先创建它们。部署应用时，服务器会为每个应用都创建一个类加载器。

在tomcat里面每个war包应用都是由 `StandardContext`类的实例来进行管理的，它会创建一个 `WebappClassLoader`来加载我们应用的类。当一个用户开始执行reload，tomcat处理器会发送如下事件：
* 调用 `StandardContext.reload()` 方法
* 使用一个 新的`WebappClassLoader`来替换上一个。
* 删除所有对servlet的引用。
* 创建新的servlet。
* 调用`Servlet.init()`方法。

![](/assets/picture/2018-12-24_2.png)

调用`Servlet.init()`会使用新类加载器加载的类重新创建已经被初始化的应用的状态。这种方式的主要问题是，我们要从头开始初始化，通常包括metadata/的配置处理，缓存预热，运行各种检查等。在一个足够大的应用中，通常要花费数分钟，但是在小型引用中，通常只需要花费几秒就可以了。

这和我们之前讨论的Counter的例子非常相似，并且也同时受到相同问题的困扰。大多数应用在重新载入几次之后就会花光内存。这是由于应用容器，库和Java核心类之间的许多连接是应用运行所必须的。即使只有一个连接没有主动的清理，应用程序就不会被垃圾回收掉。

### 现代类加载器

拥有基础体系的类加载器并不总是足够的。现代类加载器，包括OSGi 的每一个模块(通常是一个jar文件)模块都拥有一个类加载器。所有的类加载器都是平等的。一个可能会引用另外一个，并且共享一个中心仓库。每一个模块声明它导入和导出的包，通过包名，在中心仓库中可以查到相关的类加载器。

假设所有的包被必须被导入或者导出，则一个简单的模块类加载器就像下面这样：

        class MClassLoader extends ClassLoader {
         // Initialized during startup from imports
         Set<String> imps;

         public Class loadClass(String name) {
           String pkg = name.substring(0, name.lastIndexOf('.'));

           if (!imps.contains(pkg))
             return null;

           return repository.loadClass(name);
         }
        }

        class MRepository {
         // Initialized during startup from exports
         Map<String, List<MClassLoader>> exps;

         public Class loadClass(String name) {
           String pkg = name.substring(0, name.lastIndexOf('.'));

           for (MClassLoader cl : exps.get(pkg)) {
             Class result = cl.loadLocalClass(name);
             if (result != null)
               return result;
           }

           return null;
         }
        }

现代类加载器的故障排查依然和双亲委派模型的类加载器模型一样，使用 `ClassLoader.getResource()` 和 `-verbose:class` 。但是你需要像考虑类路径classpath一样考虑导入导出(export/import)

现代方法存在的一些问题，首先 导入是单向的，如果使用Hibernate，则你需要导入它，但是它不能访问你的类。第二，泄漏会成为更加严重的问题：更多的类加载器，在它们之间会有更多的引用，也就更加容易造成泄漏。第三，JVM在loadClass()获取全局锁时可能造成死锁(这个问题在Java SE 7 中已修复)

### 如何修复它？

我们必须意识到互相隔离的模块或者应用通过类加载器加载的一个幻觉：泄漏可以发生，也会发生。隔离的自然抽象是一个进程，这在Java之外的世界得到广泛的认识：.NET 动态语言和浏览器都是用进程来做隔离。

每个应用程序的单独进程是支持无泄漏更新的隔离的唯一方法。进程使用内存隔离，所以你在进程之间不能简单的进行引用。因此，如果一个新版本的应用时一个新的进程，则不能包含来自前一个版本的泄漏。

JavaEE生产应用程序的最先进的解决方案是每个应用程序在单独的Web应用程序上运行，并使用会话滚动重启来确保新版本不会被前一个版本影响。

# Too Long, Didn’t Read (TL;DR)

关于类加载器我们还有很多方面没有覆盖到，如果这篇文章太长，你记不住，则以下是几个关键点你必须了解：

1. 验证类加载猜想。讲真，当类加载出问题时，常识并不一定正确。所以需要确保对每种可能有进行测试，并验证每个类都是从哪里来的。上面的常见问题列表在此时会很有帮助。
2. 类加载器泄漏。泄漏一个类加载器，就已经足够泄漏由这个类加载器加载的任何一个类了。真相是类加载器不会给出任何隔离保证，因此大多数应用都会泄漏类加载。甚至像OSGi这种现代隔离模型也不能避免这种情况。
3. 使用进程隔离。进程通过单独的内存空间来保证应用隔离，其他java之外的平台来使用它来保证应用隔离。目前没有一个好的原因来说明我们为什么不这么做。
