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
目前有两个实现类，StandardEnvironment和StandardWebEnvironment，前者支持把SystemProperty和SystemEnvironment作为Property解析源，后者直接继承前者并增加了initPropertySource(ServletContext, ServletConfig)，额外支持ServletContext和ServletConfig作为PropertySource，且优先级更高。Env的主要逻辑在基类AbstractEnvironment，主要管理活跃Profile，而对Property的解析是委托给内部的PropertyResolver来完成的：Property来源是AbsEnv持有的MutablePropertySources对象。

# 类层次结构的设计
{%asset_img Environment类层次关系.png environment %}
从结构上能发现两个重要的设计决策：
1. Get、Set方法分离。（PropertyResolver和ConfigurablePropertyResolver、Environment和ConfigurableEnvironment）看起来很麻烦，但是存在一个明显的好处：通常框架代码功能丰富方法非常多，Get/Set分离后，简化了使用者需要关注的点(方法数)；并且存在不允许直接配置Set的情况，这时只需要把对象赋值给Get接口即可。
2. 同种类型的操作抽象到一个接口里，通过继承、组合等方式合并这些操作（PropertyResolver解析属性、Environment处理活跃Profile）。

## 解析property
Spring通过PropertySourcesPropertyResolver来解析Property，主要分为两步：获取string形式的解析值，调用ConversionService转换到指定类型。

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
