---
title: Spring源码阅读(1)
date: 2019-10-08 14:13:17
categories: 源码
tags: Spring
---

## 前言

Spring是进行Java开发时绕不开的一个常见框架，平时工作中只用到其一小部分功能，只知其用法而不知所以然，对于一款架构优秀的框架来说，不去了解其原理未免太可惜。所以从本文开始，我将尝试从源代码层面一探究竟。

{% asset_img spring5.png Spring5结构图 %}

## 容器初始化

容器初始化入口在 **AbstractApplicationContext.refresh()**，主要逻辑如下：

```java
    @Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);
            // Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
        }
    }
```

这是典型的*模板方法*设计模式，上层抽象类中定义基本步骤，下层实现类实现具体逻辑。

Spring先后出现了XML based、（过渡的Annotation Based）、Java Config based两种配置方式。典型的类结构如下：
{% asset_img ClasspathXmlApplicationContext.png XML式 %}
{% asset_img AnnotationConfigApplicationContext.png Java Config式 %}

通过对比层次结构可以发现：在AbstractApplicationContext及以上结构完全相同，下方的结构区别如下：
1. refreshBeanFactory()不同

**XML based:**

```java
/**
	 * This implementation performs an actual refresh of this context's underlying
	 * bean factory, shutting down the previous bean factory (if any) and
	 * initializing a fresh bean factory for the next phase of the context's lifecycle.
	 */
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

**Java Config based:**

```java
/**
	 * Do nothing: We hold a single internal BeanFactory and rely on callers
	 * to register beans through our public methods (or the BeanFactory's).
	 * @see #registerBeanDefinition
	 */
	@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}
```