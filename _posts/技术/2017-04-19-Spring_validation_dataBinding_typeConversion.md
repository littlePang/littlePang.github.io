---
layout: post
title: Spring MessageSource 使用
category: 技术
tags: spring
keywords:
description:
---

# 简介

spring 4.0 支持 Bean Validation 1.0 （JSR-303） and Beand Validation 1.1（JSR-349）同时也支持 spring的 `validator`接口,应用可以为每一个`DataBinder`接口注册一个`Validator`接口。

数据校验不应该与web层绑定，它应该可以插入任何需要校验的地方。spring提供`Validator`接口在应用中的每一层，都非常有用。

`Validator`接口和`Databinder`接口构成了一个 validation 的包，它不应该被局限在MVC框架中。

`BeanWrapper`是一个在spring框架中使用非常多的一个基础思想。但你可能不会直接使用。

# 使用 spring 的 Validator 接口

使用这个接口可以用来校验对象。`Validator`接口在校验失败时，使用`Errors`对象来说明错误。

## validator 的使用

考虑有这么个对象：

        public class Person {

            private String name;
            private int age;

            // the usual getters and setters...
        }

为这个类写一个这样的校验类 来对其实例进行校验：

      public class PersonValidator implements Validator {

            /**
             * This Validator validates *just* Person instances
             */
            public boolean supports(Class clazz) {
                return Person.class.equals(clazz);
            }

            public void validate(Object obj, Errors e) {
                ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
                Person p = (Person) obj;
                if (p.getAge() < 0) {
                    e.rejectValue("age", "negativevalue");
                } else if (p.getAge() > 110) {
                    e.rejectValue("age", "too.darn.old");
                }
            }
        }

其中`ValidationUtils`是一个校验帮助类。上面的静态方法 rejectIfEmpty 表示如果 name 属性为 null 或者 空字符串 则 拒绝。（查看 ValidationUtils 的文档获取更多信息）


### 错误码与错误信息的映射

输出错误信息是讨论数据绑定和校验的最后一步，如果我们想要使用`MessageSource`接口来处理，我们可以定义不同的错误码来处理。又或者可以使用`MessageCodesResolver`来处理。

### Bean 操作 和 BeanWrapper

在 beans 包下一个很重要的接口是`BeandWrapper`和它的一个相关实现`BeandWrapperImpl`,`BeandWrapper`提供了get和set一个属性，获取一个属性的描述，或者读取这个属性是否可写可读的方法，并且支持嵌套属性。`BeanWrapper`支持给一个标准的JavaBean（拥有默认无参构造器，每一个属性都有getter和setter）添加`PropertyChangeListeners`和`VetoableChangeListeners`,并且`BeanWrapper`还支持建立属性索引。

`BeanWapper`所做的事可已总结为：包装一个Bean来执行一些操作，例如设值和检索（恢复）属性。

#### 属性设值和嵌套属性

例子：

* name : 表示属性name，相关的方法为`getName()`,`isName()`和`setName()`
* account.name : 表示对象的account属性的name属性
* account[2] : 表示 属性account的第三个元素，acount可以是array，list或者其他自然序的集合
* account[companyName]:表示Map属性account的key为companyName的值。

如果你仅仅是使用了`DataBinder`和`BeanFactory`它们的一些开箱即用的实现，那你就可以不关心`PropertyEditors`的使用了。

下面是`PropertyEditors`的使用例子：

          BeanWrapper company = new BeanWrapperImpl(new Company());
          // 设置属性name的值为Some Company Inc.
          company.setPropertyValue("name", "Some Company Inc.");
          // 这个和上面是一样的结果
          PropertyValue value = new PropertyValue("name", "Some Company Inc.");
          company.setPropertyValue(value);

          // ok, let's create the director and tie it to the company:
          BeanWrapper jim = new BeanWrapperImpl(new Employee());
          jim.setPropertyValue("name", "Jim Stravinsky");
          company.setPropertyValue("managingDirector", jim.getWrappedInstance());

          // retrieving the salary of the managingDirector through the company
          Float salary = (Float) company.getPropertyValue("managingDirector.salary");


### 内建的属性编辑实现

`PropertyEditor`的实现用来处理从一个字符串转换为一个你所需要的对象。下面是内置的属性编辑器

* ByteArrayPropertyEditor
* ClassEditor
* CustomBooleanEditor
* CustomCollectionEditor
* CustomDateEditor
* CustomNumberEditor
* FileEditor
* InputStreamEditor
* LocaleEditor
* PatternEditor
* PropertiesEditor
* StringTrimmerEditor
* URLEditor

## spring的类型转换

### 类型转换接口定义：

          package org.springframework.core.convert.converter;

          public interface Converter<S, T> {

              T convert(S source);

          }

### 类型状态集中管理工厂类：

        package org.springframework.core.convert.converter;

        public interface ConverterFactory<S, R> {

            <T extends R> Converter<S, T> getConverter(Class<T> targetType);

        }

### 更加灵活的类型转换接口，但就不是强类型签名了

这个接口更加灵活和复杂，建议在非必须的情况下，尽量使用 `Converter` 和 `ConverterFactory`

        package org.springframework.core.convert.converter;

        public interface GenericConverter {

            public Set<ConvertiblePair> getConvertibleTypes();

            Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

        }

### 带条件的类型转换

如果只想在某些条件下才执行转换，则可使用 `ConditionalGenericConverter` 接口，在 `matches（）`方法返回为true才会执行类型转换。


### ConversionService 接口

        package org.springframework.core.convert;

        public interface ConversionService {

            boolean canConvert(Class<?> sourceType, Class<?> targetType);

            <T> T convert(Object source, Class<T> targetType);

            boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);

            Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

        }

这个接口用来在执行时执行所有的 converter 实例。大部分`ConversionService`的实现都 实现了 `ConverterRegistry`接口，可注册自定义的 converter。在`core.convert.support`提供了一个粗暴的实现`GenericConversionService`。`ConversionServiceFactory`可用来方便的注册 converter 。


## ConversionService 的配置

在spring中可像如下方式注册一个 ConversionService

        <bean id="conversionService"
                class="org.springframework.context.support.ConversionServiceFactoryBean">
            <property name="converters">
                <set>
                    <bean class="example.MyCustomConverter"/>
                </set>
            </property>
        </bean>

## 使用 formatter spi 实现对象域的不同时区的格式化


          public interface Formatter<T> extends Printer<T>, Parser<T> {
          }

          public interface Printer<T> {
              String print(T fieldValue, Locale locale);
          }

          public interface Parser<T> {
              T parse(String clientValue, Locale locale) throws ParseException;
          }

## 通过自定义注解来实现对象域的格式化输出

        package org.springframework.format;

        public interface AnnotationFormatterFactory<A extends Annotation> {

            Set<Class<?>> getFieldTypes();

            Printer<?> getPrinter(A annotation, Class<?> fieldType);

            Parser<?> getParser(A annotation, Class<?> fieldType);

        }

## formatter格式器注册接口


        package org.springframework.format;

        public interface FormatterRegistry extends ConverterRegistry {

            void addFormatterForFieldType(Class<?> fieldType, Printer<?> printer, Parser<?> parser);

            void addFormatterForFieldType(Class<?> fieldType, Formatter<?> formatter);

            void addFormatterForFieldType(Formatter<?> formatter);

            void addFormatterForAnnotation(AnnotationFormatterFactory<?, ?> factory);

        }

## formater批量注册接口

        package org.springframework.format;

        public interface FormatterRegistrar {

            void registerFormatters(FormatterRegistry registry);

        }

## spring mvc 中的类型转换和格式化输出

[http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-config-conversion](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-config-conversion)

## 日期格式转换配置

可使用 `@DateTimeFormate`注解，或者使用如下方式自定义日期转换：

          <?xml version="1.0" encoding="UTF-8"?>
          <beans xmlns="http://www.springframework.org/schema/beans"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="
                  http://www.springframework.org/schema/beans
                  http://www.springframework.org/schema/beans/spring-beans.xsd>

              <bean id="conversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
                  <property name="registerDefaultFormatters" value="false" />
                  <property name="formatters">
                      <set>
                          <bean class="org.springframework.format.number.NumberFormatAnnotationFormatterFactory" />
                      </set>
                  </property>
                  <property name="formatterRegistrars">
                      <set>
                          <bean class="org.springframework.format.datetime.joda.JodaTimeFormatterRegistrar">
                              <property name="dateFormatter">
                                  <bean class="org.springframework.format.datetime.joda.DateTimeFormatterFactoryBean">
                                      <property name="pattern" value="yyyyMMdd"/>
                                  </bean>
                              </property>
                          </bean>
                      </set>
                  </property>
              </bean>
          </beans>

## JDR-303 数据校验概述

可使用如下方式进行校验

        public class PersonForm {

            @NotNull
            @Size(max=64)
            private String name;

            @Min(0)
            private int age;

        }

## 其他

      <bean id="validator"
          class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean"/>

`LocalValidatorFactoryBean`同时实现了`javax.validation.ValidatorFactory`和`javax.validation.Validator`接口，可在service中通过`@Autowired`获得.


通过注解来指定属性的校验器，@Constraint指定此注解的校验使用 MyConstraintValidator 校验器

        @Target({ElementType.METHOD, ElementType.FIELD})
        @Retention(RetentionPolicy.RUNTIME)
        @Constraint(validatedBy=MyConstraintValidator.class)
        public @interface MyConstraint {
        }

## 使用 DataBinder

          Foo target = new Foo();
          DataBinder binder = new DataBinder(target);
          binder.setValidator(new FooValidator());

          // bind to the target object
          binder.bind(propertyValues);

          // validate the target object
          binder.validate();

          // get BindingResult that includes any validation errors
          BindingResult results = binder.getBindingResult();
