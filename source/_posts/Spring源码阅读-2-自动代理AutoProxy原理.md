---
title: Spring源码阅读(2) 自动代理AutoProxy原理
date: 2019-11-10 11:12:26
categories:
tags:
---

Spring AOP在编程式代理的基础上提供了自动代理机制，在需要创建大量代理的场景下非常用，主要利用了Bean钩子：BeanPostProcessor系列接口，在Bean实例化/初始化阶段介入，返回代理对象。

# 基类 AbstractAutoProxyCreator

{% asset_img AbstractAutoProxyCreator.png 基类 %}

BeanPostProcessor包括两个方法，在Bean**初始化**前后调用，其子类**InstantiationAwareBeanPostProcessor**增加了两个方法，用来在Bean**实例化**前后调用。
自动代理基类 AbstractAutoProxyCreator ，实现了 BeanPostProcessor系列接口，因此可以在 Bean实例化、初始化前后介入，适时返回代理对象。

实例化Bean前尝试做代理

```java
@Override
    // InstantiationAwareBeanPostProcessor接口的实例化前回调
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        // 构造cacheKey
		Object cacheKey = getCacheKey(beanClass, beanName);

        // bean没有名字，或者targetSourcedBeans中没有该bean
		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
            // 处理过的都会缓存，存在的话就不处理
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
            // 基础设施 或 应该跳过(默认实现是bean符合原始bean惯例时跳过)
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// 若能获取到一个TargetSource，则创建代理
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
            // 不管返回什么值都会创建代理（因为customTargetSource非空，也有用处)
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}
```

