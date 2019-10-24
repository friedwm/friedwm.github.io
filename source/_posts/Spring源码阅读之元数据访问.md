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
    1. 用于标识组件的：@Component（根据用途的不同利用[元注解][1]能力，派生出: @Controller, @Service, @Repository, @Configuration, @ControllerAdvice）
    2. 用于Java Config的：@ComponentScan, @Profile, @Import, @ImportResource, @PropertySource @Bean, @Lazy, @Scope, @Primary等
    3. 用于启动特定类型功能的：@EnableXXX
    4. 用于表示依赖关系的：@Autowired, @Qualifier

### 标识组件的 @Component
扫描组件(BeanDefinition)发生在容器启动早期


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
