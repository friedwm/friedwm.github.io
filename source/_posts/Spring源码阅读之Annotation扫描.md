---
title: Spring源码阅读之Annotation扫描
date: 2019-10-24 17:25:02
categories:
tags:
---

# 前言
发展初期的Spring倡导纯粹的IOC，对象间的依赖关系完全通过XML表达，很好的实现了依赖翻转，但也导致了成吨的难维护的XML配置文件。Spring利用Java5引入的注解(Annotation)有效的缓解了这个问题，即通过扫描代码中的注解来识别依赖关系，既简洁又直观。简单的(懒)力量是如此强大，乃至最近几年全注解式的SpringBoot占据了主要地位。

# Spring中的四类注解
Spring中的注解主要分类4类
    1. 用于标识组件的：@Component（根据用途的不同派生出了 @Controller, @Service, @Repository, @Configuration, @ControllerAdvice）
    2. 用于Java Config的：@ComponentScan, @Profile, @Import, @ImportResource, @PropertySource @Bean, @Lazy, @Scope, @Primary等
    3. 用于启动特定类型功能的：@EnableXXX
    4. 用于表示依赖关系的：@Autowired, @Qualifier