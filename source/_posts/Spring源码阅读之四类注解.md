---
title: Spring源码阅读之四类注解
date: 2019-10-24 17:25:02
categories:
tags:
---

## 前言
早期的Spring IOC对象间的依赖关系完全通过XML表达，很好的实现了依赖翻转，但也导致了成吨的难维护的XML配置文件。Spring利用Java5引入的注解([Annotation][2])有效的缓解了这个问题，即通过扫描代码中的注解来识别依赖关系，既简洁又直观。~~懒~~简单的力量是如此强大，乃至最近几年全注解式的SpringBoot成为了主要的开发方式。

## Spring中的四类注解
Spring中的注解主要分类4类：
    1. 用于标识组件的：@Component系列（根据用途的不同派生出: @Controller, @Service, @Repository等）
    2. 用于Java Config的：@ComponentScan, @Profile, @Import, @ImportResource, @PropertySource @Bean, @Lazy, @Scope, @Primary等
    3. 用于启动特定类型功能的：@EnableXXX系列 （单独拎出来是因为自称一系，其实主要利用了@Import元注解）
    4. 用于表示依赖关系的：@Autowired, @Qualifier, @Inject, @Resource(后俩为javax标准)

值得注意的是，注解只是一个包含少量数据的标记，本身并不能直接改变程序运行流程，必须由相应的处理代码实现。

### @Component系列
除了基本的@Component，其余的注解都有特定业务含义：

- @Controller用来标注在mvc的控制器上
- @Service放置在业务逻辑实现对象上
- @Repository放置在数据访问对象(DAO)上
- @ControllerAdvice放置在控制器横截处理器类上
- @Configuration比较特殊，用来标记配置类

这里用到了Java5的[元注解][1]，即注解上的注解，例如：

{%asset_img AtService.png @Service的定义%}

元注解唯一的语法要求是 @Target必须配置成：ElementType.TYPE或ElementType.ANNOTATION_TYPE。

就@Component系的注解来说，容器启动时利用 ClassPathBeanDefinitionScanner 扫描指定package，把标注了该注解的类作为候选BeanDefinition。
{%asset_img ClasspathBeanDefinitionScanner.png %}

```java 
    // 扫描BeanDefinition入口
    public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
        // 默认是true
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
```

doScan()完成实际扫描功能，如果包含注解配置（默认是）就注册相应的处理器，包括：

* ConfigurationClassPostProcessor
* AutowiredAnnotationBeanPostProcessor
* CommonAnnotationBeanPostProcessor
* PersistenceAnnotationBeanPostProcessor(类存在时)

#### doScan()逻辑

```java
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
            // 在这里扫描出候选BeanDefinition
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
                // 解析bean scoped元数据，包括scope名，scoped-proxy类型: NO/DEFAULT(不创建scoped-proxy)，INTERFACES（创建Jdk代理型），TARGET_CLASS（创建cglib代理型）
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
                // 生成beanName
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                // 常规的Bean，补全默认值
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
                // 在@Configuration类用中@Bean定义的Bean，解析@Primary, @Lazy, @DependsOn, @Role, @Description等注解上的值设置到BeanDefinition中
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
                // 检查BeanName与BeanDefinition是否可以注册
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    // 适时替换成scoped-proxy BeanDefinition
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

doScan有两个值得分析的点：
1. 具体扫描逻辑 findCandidateComponents(basePackage)
2. scoped-proxy的实现
   
下面分别来看看:

#### findCandidateComponents(basePackage)

```java
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
			return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
		}
		else {
			// 重点关注这里
			return scanCandidateComponents(basePackage);
		}
	}
```

```java
	private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
					// 解析出指定package下所有Resource（这里是 .class)
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						// 根据Resource获取相应的MetadataReader
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
						// 初步检查是否符合候选组件
						if (isCandidateComponent(metadataReader)) {
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						else {
							if (traceEnabled) {
								logger.trace("Ignored because not matching any filter: " + resource);
							}
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because not readable: " + resource);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}

```

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
