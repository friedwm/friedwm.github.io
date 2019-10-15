---
title: Spring源码阅读之Environment
date: 2019-10-12 14:50:37
categories:
tags:
---

# 简介

Environment接口是Spring对Profile和Property两个方面的抽象，主要处理运行环境对应用程序的影响。

* Profile
    Profile是一个命名的，包含了一些BeanDefinition的逻辑分组，只有当这个profile处于active状态，包含的bean才会注册到容器中，容器通过Environment获取启用的Profile名称。

* Property
    Property就是键值对，来源很多：属性文件、JVM系统属性、系统环境变量、JNDI、servlet context参数、Maps等。Environment通过继承*PropertyResolver*获得解析Property的能力（获取Property、解析占位符）

# 实现
目前有两个实现类，StandardEnvironment和StandardWebEnvironment，前者支持把SystemProperty和SystemEnvironment作为Property解析源，后者直接继承前者并增加了initPropertySource(ServletContext, ServletConfig)，额外支持ServletContext和ServletConfig作为PropertySource，且优先级高于SystemProperty和SystemEnvironment。Env的主要逻辑在基类AbstractEnvironment中,其主要处理Profile，而对Property的解析是委托给内部的PropertyResolver来完成的：Property来源是AbsEnv持有的MutablePropertySources对象。

## 解析property
解析Property的实现类是PropertySourcesPropertyResolver，主要分为两步：获取string形式的解析值，调用ConversionService转换到指定类型。

```java
@Nullable
	protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		if (this.propertySources != null) {
            // 依次遍历PropertySources，返回第一个找到的值
			for (PropertySource<?> propertySource : this.propertySources) {
				if (logger.isTraceEnabled()) {
					logger.trace("Searching for key '" + key + "' in PropertySource '" +
							propertySource.getName() + "'");
				}
				Object value = propertySource.getProperty(key);
				if (value != null) {
					if (resolveNestedPlaceholders && value instanceof String) {
						value = resolveNestedPlaceholders((String) value);
					}
					logKeyFound(key, propertySource, value);
                    // 转换到指定类型
					return convertValueIfNecessary(value, targetValueType);
				}
			}
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Could not find key '" + key + "' in any property source");
		}
		return null;
	}
```
