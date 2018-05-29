---
layout: post
title: effective java 读书笔记
category: 技术
tags: java
keywords:
description:
---

# 创建和销毁对象

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
          private final String name;
          private final String password;

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

        public class SingletonClass {
          private static final SingletonClass INSTANCE = new SingletonClass();

          private SingletonClass() {}

          public static SingletonClass getInstance() {
            return INSTANCE;
          }
        }

除了放射，序列化后再反序列化成对象，也会破坏单例模式（反序列化会调用构造器实例化一个新对象），可通过 在单例中加入 readResolve() 方法解决

## 通过私有构造器强化不可实例化的能力

        public class UtilityClass{
          private UtilityClass() {
            throw new AssertionError();
          }

          // 其他static 工具类静态方法
        }

## 避免创建不必要的对象

* 将 重复创建的局部变量提取为类属性
* 注意基本数据类型的自动装箱(下面的代码，由于sum是Long包装类型，所以会做相加操作时，都会创建一个多余的Long对象

        Long sum=0L;
        for (long i=0; i<Integer.MAX_VALUE; i++) {
          sum += i;
        }

并非提倡使用对象池，小对象的创建和回收是非常廉价的，通过创建附加对象，提升程序清晰性，简洁性和功能性，通常是好的。

## 清除过期的对象引用

        public class Stack{
          private Object[] elements;

          public void publish(Object s) {
            // ensure capacity
            elements[size++] = s;
          }

          public Object pop() {
            return elements[--size];
          }
        }

上面的pop方法，虽然将大小减了一，但是其实他仍然持有最后一个元素的引用，导致其无法被垃圾回收，应将其置为null，清除过期的对象，使其可以被回收：

        public Object pop() {
          Object o = elements[--size];
          elements[size] = null; // help gc
          return o;
        }

第二个内存泄漏常见来源：缓存

第三个内存泄漏常见来源：监听器和回调


## 避免使用终结方法

* 从对象变为不可达到其 finalizer 方法被执行的时间，可能是无限长的。（在 Finalizer 类中可以看到执行 finalizer方法 的线程优先级被置为了 Thread.MAX_PRIORITY - 2，所以你并不知道它啥时候会被执行）

* finaizer方法会使得创建和销毁对象都更慢

最好还是使用 try finally 模块进行资源释放等操作


# 对于所有对象都通用的方法

## 覆盖equals时请遵守通用约定

满足以下任意一个条件，就不要覆盖equals方法

* 类的每个实例本质上都是唯一的（例如Thread）
* 不关心类是否提供了“逻辑相等”的测试功能
* 超类已覆盖equals，并且从超类继承过来的行为对子也是合适的（例如大多数Set实现都是从AbstractSet继承equals实现）
* 类是私有的或者包级私有的，可以确定它的equals方法永远不会被调用（通过以下方式防止意外调用）

        public boolean equals(Object o) {
          throw new AssertionError();
        }

覆盖equals方法必须遵循的约定（来自JavaSE6）
* 自反性，非null引用， x.equals(x) 必须为 true
* 对称性，非null引用x和y，当且仅当 x.equals(y) 为 true 时，y.equals(x) 必须返回 true
* 传递性，非null引用x、y、z, 如果 x.equals(y) 和 y.equals(z) 都返回true，则 x.equals(z) 必须返回true
* 一致性，非null引用x和y，只要equals的比较操作所用的信息没有被修改，多次调用 x.equals(y)就必须一致的返回true，或者全部返回false

高质量equals诀窍
* 使用 == 检查 是否为相同引用
* 使用 instanceof 检查 是否为正常的类型
* 把参数转换为正确的类型
* 编写完成equals后，反问自己，它是否是自反的、对称的、传递的、一致的？（通过单测来保证）

## 覆盖equals时总要覆盖hashCode

如果覆盖equals不覆盖hashCode，则在与HashMap一起使用的时候，会出现 逻辑相等的两个对象，作为key，无法正常替换或者查询。

hashCode的简单实现：
1. 将一个常量存入名为result的int类型变量总
2. boolean类型，值为 (f?1:0)
3. byte、char、short或者int类型，都将其转为int类型
4. long型，则 (int)(f^(f>>>32)), 高32位异或低32位
5. double类型，使用 Double.doubleToLongBits(f), 然后再使用上面的long的计算方法，高32位异或低32位
6. float类型，使用 Float.floatToIntBits(f)
7. 如果是 对象引用，则使用上述规则递归调用其hashCode
8. 数组，则将其每个元素使用上述规则单独处理
9. 最后使用 result = result * 31 + c (c为上述规则计算出来的值)

## 始终要覆盖toString

toString方法应该返回对象中包含的所有值得关注的信息，使类用起来更加舒适

## 谨慎使用clone

当类实现了 cloneable 接口后，调用从 Object继承下来的clone() 方法，会将对象的域逐个拷贝，但是如果域是另外一个对象或者数组，那么clone出来的对象里，的这些域和原始对象的域指向 的其实是同一个对象，所以实在要使用clone的时候，要考虑深层拷贝

## 考虑实现Comparable接口

实现Comparable接口，可以方便的使用JDK中的方法，注意：强烈建议 Comparable的实现满足 (x.compareTo(y) == 0) == (x.equals(y)), 如果子类不满足此条件一定要加以说明，例如 BigDecimal （new BigDecimal("1.0") 和 new BigDecimal("1.00") 在 HashSet中会存在两个元素，因为在HashSet中使用的是 equals方法进行比较，而在 TreeSet 中 只会存在一个元素，在TreeSet中是使用compareTo()方法进行比较的）

# 类和接口

## 使类和成员的可访问性最小

这个叫 信息隐藏 或 封装（软件设计的基本原则之一），使一个模块无需知道其他模块的内部工作情况，可以有效的解除组成系统的各个模块之间的耦合关系，使得这些模块可以独立的开发、测试、优化、使用、理解和修改

## 在共有类中使用访问方法而非公有域

如果公有类暴露了它的数据域，要想在将来改变其内部表示法是不可能的，因为公有类的客户端代码已经遍布各处了。所以应使用访问方法，而非公有域

## 使可变性最小化

不可变类比可变类更加易于设计、实现和使用。不可变类的5个规则：
* 不要提供任何会修改对象状态的方法
* 保证类不会被扩展
* 使所有的域都是final的
* 使所有的域都成为私有的
* 确保对任何可变组件的互斥访问

## 复合优于继承

使用复合和转发机制来代替继承，尤其是当存在是当存在适当的接口可以实现包装类的时候，包装类不仅比子类更加健壮，而且功能也更加强大。

## 要么为继承而设计，并提供稳定说明，要么就禁止继承

例如为继承而设计的集合类（List, AbstractList）

## 接口优于抽象类

接口的优势：
* 现有的类可以很容易被更新，已实现新的接口
* 接口是定义mixin(混合类型)的理想选择
* 接口允许我们构造非层次结构的类型框架（例如可以将某些行为抽象成接口）

## 接口只用于定义类型

接口应该只被用来定义类型，它们不应该被用来导出常量（接口中不应该定义常量）

## 类层次优于标签类

例如 Figure 图形类，用用来同时表示长方形和圆的实例，内部使用一个字段来标记它是长方形还是圆，更正确的方式是定义 Rectangle和Circle 类，分别用来表示 长方形和圆

## 用函数对象表示策略

例如 使用字符串长度进行排序，则可以使用一个 实现 Comparable 接口的 StringLengthComparator 类来表示 使用长度进行排序的策略

## 优先考虑静态成员类

如果要使用内部类，应优先考虑使用静态成员类，这样在实例化内部类的时候，无需先实现外部类（有些时候外围类并不是必须的）。


# 泛型

## 请不要在新代码中使用原生态类型

泛型可以在编译期执行类型检查，而原生类型是在运行时执行强制转换时才能发现错误，原生类型只是为了与引入泛型之前的遗留代码进行兼容和互用而提供的。

## 消除非受检警告

应该尽量消除警告，有些无法消除的警告，但如果可以证明引起警告的代码是类型安全的，则可以使用 `@SuppressWarnning("unchecked")` 来禁止警告

## 列表优于数组

列表比数组更易于在编译器发现问题，不用将列表和数组混用，例如定义 List<String>[] 的列表数组

## 优先考虑泛型

当在类定义，要考虑类元素的扩展性时，应该使用泛型，而不是 将元素类型定义为Object。

## 优先考虑泛型方法

静态方法也应该优先考虑泛型方法

## 利用有限制通配符来提升API的灵活性

利用 <Number>  、<? extend E> 、<? super E>来提示API的灵活性

## 优先考虑类型安全的异构容器

使用泛型使类变为类型安全（有编译器类型检查）的类， 例如 ：

          public class Favorites {
            private Map<Class<?>, Object> favorites = new HashMap<>();

            public <T> void put(Class<T> type, T instance) {
              if (type != null) {
                favorites.put(type, instance);
              }
            }

            public <T> T get(Class<T> type) {
              return type.cast(favorites.get(type));
            }
          }

# 枚举和注解

## 用enum代替int常量

使用int的问题
* 多个int常量类之间，是没有命名空间的，也就是能把APPLE常量类中的常量传递到想要ORANGE常量类的方法中。
* int是编译时常量，会被编译到使用它们的客户端中，若果枚举常量关联的int发生了变化，客户端就必须重新编译。（没有重新编译，也能运行，但是行为是不确定的）

      public enum Apple {ONE, TWO}
      public enum Orange {ONE, TWO}

使用枚举的好处

* 枚举提供了编译时的类型安全，如果参数类型为Apple，就可以保证，任何被传到此参数上的任何非null对象引用的一定是Apple枚举中定义的枚举之一（针对上面提出的的第一个问题）
* 可以增加或者重新排列枚举类型中常量，而无需重新编译客户端，因为导出常量的域在枚举类型和它的客户端直接提供了一个隔离层（针对上面的第二个问题）
* 枚举还可以添加额外的域和方法，提供了更大的灵活性

## 用实例域代替序数

如果需要序数应该定义一个实例域，而不是使用 ordinal(), 因为如果使用了 ordinal() 根本就无法再对常量进行重排序，因为重排序之后，ordinal()的值也就跟着变化了。


        public enum Ensemble {
          SOLO(1), DUET(2),TRIO(3);

          private static final int nunmberOfMusicians;

          Ensemble(int size) {
            this.nunmberOfMusicians = size;
          }

          public int nunmberOfMusicians() {
            return nunmberOfMusicians;
          }
        }


## 用EnumSet代替位域

如果要使用位域(每一个二进制位表示一种类型，后续使用 位或 运算，进行多个类型合并)，则使用EnumSet会更加方便，且功能更加强大。（元素个数小于等于64个时，真个EumSet就是用单个long表示的，所以性能也比的上位域的性能）

## 用EnumMap代替序数索引

不要使用序数来索引数组，而要使用EnumMap。

将 `Set<Herb>[] herbByType = new Set[Herb.Type.valuse().length]`; 这种代码，使用 `Map<Herb.Type, Set<Herb>> hersByType = new EnumMap(Herb,Type.class)` 来代替

## 用接口模拟可伸缩的枚举

例如下面这样一个可用来计算来个数相加操作的枚举，这样的枚举作为api发布到客户端，是无法进行扩展的，无法支持其他的 相乘，相减等操作。

        public enum BasicOperation {
          PLUS("+") {
            public double apply(double x, double y) {return x + y;}
          }
          public abstract double apply(double x, double y);
        }

可以在这个枚举的基础上抽象出一个接口：

        public interface Operation {
          double apply(double x, double y);
        }

然后将上面的枚举改写为：

        public enum BasicOperation implements Operation {
          PLUS("+") {
            public double apply(double x, double y) {return x + y;}
          }
        }

虽然此枚举是无法扩展的，但是接口是可以的，如果客户端需要增加 相乘和相减的操作，则可以定义一个如下的枚举:

        pubic enum ExtendOperation implements Operation {
          MINUS("-") {
            public double apply(double x, double y) {return x - y;}
          }
          TIMES("*") {
            public double apply(double x, double y) {return x * y;}
          }
        }

然后保证在对两个数做操作的地方都是使用 Operation 接口，而非具体的枚举类即可。

接口模拟可伸缩枚举的不足：无法实现从一个枚举类型继承到另一个枚举类型。（在代码较少的情况下可以考虑直接将 BasicOperation中的代码复制到 ExtendOperation中，在代码较多的情况下，可以将它封装在一个辅助类或者静态辅助方法中，来避免代码的复制工作）

## 注解优于命名模式

命名模式会造成错误的安全感（即：命名错误，但是程序也不会报错），并且命名模式没有提供将参数值与程序元素关联起来的好方法。而注解可以方便的解决这些问题，所以既然有了注解，就完全没有理由再使用用命名模式了

## 坚持使用Override注解

public class Bigram {
  private int id;
  public boolean equals(Bigram b) {
    return id == b.id;
  }
}

以上的 Bigram 类原本是想要覆盖 equals 方法(这里省略了 hashCode 方法)，但是由于 参数类型是 Bigram 而非 Object ，所以这个方法其实是对equals进行了重载，而 使用 @Override 就可以在编译器发现这种问题

## 用标记接口定义类型

标记接口 是不包含任何方法的接口，只是指明一个类实现了具有某种属性的接口。例如 Serializable 接口

标记接口比标记注解的好处：
* 标记接口定义的类型由被标记类的实例实现，所以可以允许你在编译时捕捉在使用标记注解的情况下要到运行时才能捕捉到的错误
* 标记接口比标记注解的另外一个优点是，它可以被更加精确的进行锁定（如果注解使用了 @Target(ElementType.TYPE),则其可以用到任何地方）

什么时候用标记接口，什么时候用标记注解？

如果标记应用到任何程序元素而不是类或者接口，就必须使用注解，如果标记只给应用或者接口，则就要考虑：是否要编写只接受有这种标记的方法？如果是，则使用标记接口。

# 方法

## 检查参数的有效性

对参数的校验，应在方法最开始就进行，校验失败，快速报错，不应继续往下执行，避免出现一些诡异的异常，也可避免一些不需要的操作。


## 必要是进行保护性拷贝

如果累具有从客户端得到或者返回到客户端的可变组件，类就必须保护性的拷贝这些组件，因为客户端可能会不恰当的修改组件。


## 谨慎设计方法签名

* 谨慎选择方法名称（应始终遵循标准命名规则）
* 不要过于追求提供便利的方法（方法太多，反而是类难以学习，使用，文档化，测试和维护）
* 避免过长的参数列表（可使用Builder模式）

## 慎用重载

以下代码，将打印出两个 unknown，而不是一个 set 和 一个 unknown,因为重载方法的选择，是在编译是决定的，多以在编译时，两个集合的类型都是Collection,所以在执行classify时，都是选择的 classify(Collection<?> s) ，*_安全而保守的策略是，永远不要到处两个具有相同参数数目的重载方法，如果方法使用了可变参数，保守的策略是不要重载它。_*

          public class CollectionCLassifier {
            public static String classify(Set<?> s) {
              return "set";
            }
            public static String classify(Collection<?> s) {
              return "unknown";
            }
            public static void main(String[] args) {
              Collection<?>[] collections = {new HashSet<Integer>(), new ArrayList<Integer>()};
              for (Collection<?> c : collections) {
                System.out.println(classify(c));
              }
            }
          }




## 慎用可变参数

在定义参数数目不定的方法时，可变参数方法是一种很方便的方式，但是它们不应该被过度滥用，如果使用不当，会产生混乱的结果

## 返回零长度的数组或者集合，而不是null

可避免客户端的空指针问题，而且对象的创建也不会造成性能问题


## 为所有导出的API元素编写文档注释

为API编写文档，文档注释是最好，最有效的途径


# 通用程序设计

## 将局部变量的作用域最小化

将局部变量作用于最小化，可以增强代码的可读性和可维护性，并降低出错的可能性。局部变量应在第一次使用的地方声明它。

## for-each优于传统for循环

for-each循环在简洁性和预防bug方面有着传统的for循环无法比拟的优势，并且没有性能损失，所以应尽可能的使用for-each循环。（但在 过滤，等时候无法使用for-each）


## 了解和使用类库

不要重复发明轮子，尽可能使用类库

## 如果需要精确的答案，避免使用float和double

float 和 double 有精度问题，应使用 BigDecimal ， int， long 代替


## 基本类型优于装箱基本类型

装箱类型的问题：
* 装箱类型具有与他们值不同的同一性
* 装箱类型可能为null
* 基本类型通常比装箱类型更节省时间和空间

注意装箱类型不能使用 == 进行等值比较

## 如果其他类型更合适，则尽量避免使用字符串

* 字符串不适合代替枚举类型
* 字符串不适合代替聚集类型
* 字符串也不适合代替能力表

## 单选字符串连接的性能问题

使用StringBuilder代替String的多次连接

## 通过接口引用对象

使用接口来引用对象，可以方便后续的替换与修改


## 接口优于反射机制

反射的坏处：
* 丧失了编译时类型检查的好处
* 执行反射访问所需要的代码非常笨拙和冗长
* 性能损失

## 谨慎使用本地方法

本地方法有些严重的缺点，它不是安全的。也不提倡使用本地方法提高性能的做法。

## 谨慎的进行优化

要努力编写好的程序，而不是快的程序。努力避免那些限制性能的设计决策

## 遵守普遍接受的命名惯例

应该把标准的命名惯例当做一种内在的机制来看待，并学着用他们的第二特性。



# 异常

## 只针对异常的情况才使用异常

不要将异常用于普通的控制流，也不要编写迫使它们这么做的API


## 对可恢复的情况使用受检异常，对编程错误使用运行时异常

对可恢复的情况使用受检异常，对编程错误使用运行时异常


## 避免不必要的使用受检异常

避免不必要的使用受检异常


## 优先使用标准的异常

更方便也更易于让人理解

## 抛出与抽象相对应的异常

方法传递由底层抽象抛出的异常时，如果异常与它所执行的任务没有明显关系时，会比较令人困惑，所以高层实现应该捕获底层的异常，同时抛出高层抽象进行解释的异常。（这叫异常转译）

## 每个方法抛出的异常都要有文档

要为你编写的每个方法所能抛出的每个异常建立文档，对于未首检和首检异常，以及对于抽象的和具体的方法也一样

## 在细节消息中包含能捕获失败的信息

失败信息中应该包含 对该异常有贡献 的参数和域的值，使异常捕获能更加方便的指定问题出错的地方。

## 努力使失败保存原子性

应尽可能使方法失败保存原子性，对数据重试和处理都更加方便

## 不要忽略异常

不要吃掉异常，不利于问题分析和排查


# 并发

## 同步访问共享的可变数据

当多个线程共享可变数据的时候，每个读或者写数据的线程都必须执行同步。

## 避免过度同步

过度同步可能会导致性能降低，死锁，甚至不确定的行为。为避免死锁和数据破坏，千万不要从同步区域内部调用外来方法，更为一般的讲，要尽量限制同步区域内部的工作量。

## executor 和 task 优于线程

尽量避免不要编写自己的工作队列，而且还应该尽量不直接使用线程。

## 并发工具优于wait和notify

直接使用wait和notify就像用“并发汇编语言”进行飙车一样，而 java.lang.concurrent 则提供了更高级的语言。没有理由在新代码中使用wait和notify，即使有，也是极少的。

## 线程安全性的文档化

线程安全级别：
* 不可变的
* 无条件的线程安全
* 有条件的线程安全
* 非线程安全
* 线程对立的 （这个类不能安全的被多个线程并发使用）

每个类都应该利用字斟句酌的说明或者线程安全注解，清楚的在文档中说明它的线程安全属性。

## 慎用延迟初始化

延迟初始化就像一把双刃剑，它降低了初始化类或者创建实例的开销，却增加了访问被延迟初始化的域的开销。大多数类的域都应该正常的进行初始化，而不是延迟初始化。


## 不要依赖线程调度器

任何依赖于线程调度器来达到正确性或者性能要求的程序，很有可能都是不可移植的。不要通过 Thead.yield 来‘修正’该程序。简而言之，不要让应用程序的正确性依赖有线程调度器

## 避免使用线程组

线程组并没有提供太多有用的功能，而且它们提供的许多功能还都是有缺陷的。我们最好把线程组看作是一个不成功的实验，可以忽略它们，就当它们根本不存在一样。



# 序列化

## 谨慎的实现 Serializable 接口

* 实现 Serializable 接口而付出的最大代价是，一旦一个类被发布，就大大降低了“改变这个类的实现”的灵活性。（如果没有显示指定 UID，则系统会自动生成，但是它收到类名称，类实现的接口名称，以及所有公有和受保护成员的名称影响，所以如果你修改了这些信息，例如，增加了一个工具方法，UID也会随之变化，从而导致兼容性被破坏，运行时产生 InvalidClassException）
* 实现Serializable的第二个代价是，它增加了出现Bug和安全漏洞的可能性
* 实现Serializable的第三个代价是，随着类发现新的版本，相关的测试负担也增加了。

如果一个专门为了继承而设计的类不是可序列化的，就不可能编写出可序列化的子类。特别是，如果超类没有提供可访问的无参构造器，子类也不可能做到可序列化。

## 考虑使用自定义的序列化形式

当决定要将一个雷做成可序列号的时候，请仔细考虑应该采用审核样的序列化形式。只有默认的序列化形式能够合理的描述对象的逻辑状态时，才能使用默认的序列化形式。通过 重写 writeObject 和 readObject 来自定义序列化形式。

## 保护性的编写readObject方法

对于不可变的域，在readObject 中要进行保护性拷贝

* 对于对象引用域必须保持为私有的类，要保护性的拷贝这些域中的每个对象，不可变类的可变组件就属于这一类别
* 对于任何约束条件，如果检查失败，则抛出一个 InvalidObjectException异常，这些检查动作应该跟在所有的保护性拷贝之后
* 如果真个对象图在被反序列化之后必须进行验证，就应该使用ObjectInputValidation接口
* 无论是直接方法还是间接方法，都不要调用任何类中任何可被覆盖的方法。

## 对于实例控制，枚举类型优于readResolve

如果类定义了一个readResolve方法，并且具备正确的声明，那么在反序列化之后，新建对象上的readResolve就会被调用。

总而言之，你应该竟可能的使用枚举类型来实施实例控制的约束条件，如果做不到，同时又需要一个即可序列化又是实例受控的类，就必须提供一个readResolver方法，并确保该类的所有实例域都为基本类型，或者transient的

## 考虑用序列化代理代替序列化实例

序列化代理可以极大的减少风险，以下为Period编写了一个 SerializableProxy 的序列化代理类，在外围类中 使用 writeReplace 将序列化的类替换为代理类，并编写 readObject 方法，不允许直接反序列化Period类，最后在 SerializableProxy 代理类中添加 readResolve 类将代理类转为外围类。

        public class Period implements Serializable {
          private final Date start;
          private final Date end;

          public Period(Date start, Date end) {
            this.start = start;
            this.end = end;
          }

          private Object writeReplace() {
            return SerializableProxy(this);
          }

          private Object readObject() {
            throw new InvalidObjectException("proxy required");
          }

          public static class SerializableProxy implements Serializable {
            private final Date start;
            private final Date end;

            SerializableProxy(Period p) {
              this.start = p.start;
              this.end = p.end;
            }

            private Object readResolve() {
              return new Period(start, end);
            }
          }
        }


简而言之，每当你发现自己必须在一个不能被客户端扩展的类上编写 readObject 或者 writeObject 方法的时候，就应该考虑使用序列化代理模式。想要稳健的将带有重要约束条件的对象序列化时，这种模式可能是最容易的方法。
