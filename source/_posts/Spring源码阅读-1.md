---
title: Spring源码阅读(1)
date: 2019-10-08 14:13:17
categories: 源码
tags: Spring
---

## 前言

经过十几年的发展Spring已经从一个单一的Bean容器框架，演化成了覆盖开发方方面面的框架族，从单机开发到云、微服务，各项功能纷繁复杂，但最核心还是IOC和AOP。本系列文章将分析Spring的IOC和AOP功能及其实现。

{% asset_img Spring5.png Spring5结构图 %}

## 容器初始化

容器初始化入口在 **AbstractApplicationContext.refresh()**，代码如下：

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

以上所有方法均由protected修饰，这是典型的*模板方法*设计模式，父类定义基本步骤和默认实现，子类实现特殊逻辑。

Spring先后出现了XML based、Java Config based两种配置方式。典型的类结构如下：
{% asset_img ClasspathXmlApplicationContext.png XML式 %}
{% asset_img AnnotationConfigApplicationContext.png Java Config式 %}

通过对比类层次结构可以发现：两个类都继承自AbstractApplicationContext，这意味着主体的refresh()步骤一致的；XML容器继承了AbstractRefreshableApplicationContext，从名字也能猜到这是支持刷新的容器，而注解容器没有继承，推断其不具备重复刷新的功能。
下面逐个查看refresh()中的调用：

1. prepareRefresh()
刷新前的准备工作，设置启动、关闭标志位，**初始化PropertySource**，校验属性中的Required。

```java
/**
	 * Prepare this context for refreshing, setting its startup date and
	 * active flag as well as performing any initialization of property sources.
	 */
	protected void prepareRefresh() {
		// Switch to active.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

1.1 initPropertySources()
默认实现是空，几个子类实现大同小异，创建一个Environment，调用初始化方法。

*AbstractRefreshableWebApplicationContext*

```java
@Override
	protected void initPropertySources() {
		ConfigurableEnvironment env = getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
```

*GenericWebApplicationContext*

```java
@Override
	protected void initPropertySources() {
		ConfigurableEnvironment env = getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, null);
		}
	}
```


2. refreshBeanFactory()不同

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

启示：基于XML的容器在refreshBeanFactory()中关闭已有的内部容器并获取一个的，重新加载BeanDefinition；而基于注解的容器则不允许重复刷新。
