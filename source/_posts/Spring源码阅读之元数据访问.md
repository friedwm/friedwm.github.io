---
title: Spring源码阅读之元数据访问
date: 2019-10-24 17:25:02
categories:
tags:
---

## 前言
早期的Spring IOC对象间的依赖关系完全通过XML表达，很好的实现了依赖翻转，但也导致了成吨的难维护的XML配置文件。Spring利用Java5引入的注解([Annotation][2])有效的缓解了这个问题，即通过扫描代码中的注解来识别依赖关系，既简洁又直观。~~偷懒~~简单的力量是如此强大，乃至最近几年全注解式的SpringBoot成为了主要的开发方式。

## Spring中的四类注解
Spring中的注解主要分类4类：
    1. 用于标识组件的：@Component系列（根据用途的不同派生出: @Controller, @Service, @Repository, @Configuration, @ControllerAdvice）
    2. 用于Java Config的：@ComponentScan, @Profile, @Import, @ImportResource, @PropertySource @Bean, @Lazy, @Scope, @Primary等
    3. 用于启动特定类型功能的：@EnableXXX
    4. 用于表示依赖关系的：@Autowired, @Qualifier

### @Component系列
除了基本的@Component，其余的注解都有特定含义：

- @Controller用来标注在mvc的控制器上
- @Service放置在service上
- @Repository放置在数据访问对象上(DAO)
- @ControllerAdvice放置在控制器横截处理器类上
- @Configuration比较特殊，用来标记配置类

以上注解都使用了java的[元注解][1]功能，即注解上的注解，例如：
{%asset_img AtService.png @Service的定义%}
元注解唯一要求是 @Target必须配置成：ElementType.TYPE或ElementType.ANNOTATION_TYPE。特别需要注意的是，不论是注解还是元注解自身都不会直接改变程序运行流程，他们都只是一个标记，必须配合相应处理代码才能发挥作用。

就@Component系的注解来说，容器启动时利用 ClassPathBeanDefinitionScanner 扫描指定package，将由@Component标注的类识别为Bean。
{%asset_img ClasspathBeanDefinitionScanner.png %}

// TODO 说明@Component

并且默认注册几个处理注解配置类的处理器，包括：

* ConfigurationClassPostProcessor
* AutowiredAnnotationBeanPostProcessor
* CommonAnnotationBeanPostProcessor
* PersistenceAnnotationBeanPostProcessor(类存在时)


## 解析元数据
Spring通过**ClassMetadata**, **AnnotatedTypeMetadata**, **AnnotationMetadata**三个接口来封装元数据的访问。

{% asset_img StandardAnnotationMetadata.png %}

其中ClassMetadata提供访问*目标类*自身信息的方法，包括是否接口、是否注解类型、是否抽象类等常用判断、是否存在父类、是否包含其他类等信息
{% asset_img ClassMetadata.png %}

AnnotatedTypeMetadata接口提供关于*目标类*的注解相关信息，包括：目标类或者[元注解][1]是否存在指定注解(全路径名，不含@)，指定注解的属性值Map，多个同名注解的复合值
{% asset_img AnnotatedTypeMetadata.png %}

AnnotationMetadata
{% asset_img AnnotationMetadata.png %}



[1]: https://dzone.com/articles/what-are-meta-annotations-in-java
[2]: https://docs.oracle.com/javase/specs/jls/se11/html/jls-9.html#jls-9.6
