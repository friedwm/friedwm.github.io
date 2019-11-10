---
title: Spring源码阅读之四类注解及实现原理
date: 2019-10-24 17:25:02
categories: 源码
tags: Spring
---

# 前言
早期Spring IOC对象间的依赖关系完全通过XML表达，很好的实现了依赖翻转，但也导致了成吨的难以维护的XML配置文件。利用Java5引入的注解([Annotation][2])Spring有效的缓解了这个问题，即通过扫描代码中的注解来识别依赖关系，既简洁又直观。~~懒~~ 简单的力量是如此强大，乃至最近几年全注解式的SpringBoot成为了主要的开发方式。

# Spring中的四类注解
Spring中的注解主要分类4类：
	1. 用于标识组件的 @Component系列（根据用途的不同派生出: @Controller, @Service, @Repository等）
	2. 用于Java注解配置的 Annotation Config系列：@Configuration, @ComponentScan, @Profile, @Import, @ImportResource, @PropertySource @Bean, @Lazy, @Scope, @Primary等
	3. 用于启动特定类型功能的 @EnableXXX系列 （严格来说也属于第二类注解配置，但其代码实现有所区别）
	4. 用于注入依赖的：@Autowired, @Qualifier, @Inject, @Resource(后俩为javax标准注解)

需要注意的是，注解只是一个能包含少量数据的标记，本身并不能直接改变程序运行流程，必须由相应的处理代码实现。下面分别看看这四类注解的用法及相关处理代码。

## @Component系列
除了基本的@Component，其余的注解都有特定业务含义：

- @Controller用来标注在mvc的控制器上
- @Service放置在业务逻辑实现对象上
- @Repository放置在数据访问对象(DAO)上
- @ControllerAdvice放置在控制器横截处理器类上
- @Configuration比较特殊，用来标记配置类

这里用到了Java5的[元注解][1]，即注解上的注解，例如：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {

	@AliasFor(annotation = Component.class)
	String value() default "";

}
```

@Service上标注了@Component，元注解也是注解，功能也是标记，在Java语法上并不直接代表@Service就是@Component了，它的含义由解析代码也就是Spring赋予。

元注解唯一的语法要求是 @Target必须配置成：ElementType.TYPE或ElementType.ANNOTATION_TYPE。

### 识别并注册BeanDefinition

就@Component系注解的解析来说，容器启动时利用 ClassPathBeanDefinitionScanner 扫描指定package，把标注了该类注解的.class文件作为候选BeanDefinition。
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

实际扫描由doScan()完成，并且如果包含注解配置（默认是）就注册相应的处理器，包括：

- ConfigurationClassPostProcessor
- AutowiredAnnotationBeanPostProcessor
- CommonAnnotationBeanPostProcessor
- PersistenceAnnotationBeanPostProcessor(类存在时)

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
                // 在@Configuration标注的配置类用中@Bean定义的Bean，解析@Primary, @Lazy, @DependsOn, @Role, @Description等注解上的值设置到BeanDefinition中
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

两个值得分析的点：

1. 查找指定package中的组件，也就是findCandidateComponents(basePackage)
2. scoped-proxy功能

下面分别来看看:

### 查找指定package中的组件

```java
	private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
			// 默认 resourcePattern = **/*.class，即所有的class
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
					// 由ResourceLoader解析出指定package下所有Resource
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

	// 判断是否组件
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		// 先检查黑名单
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return false;
			}
		}

		// 后检查白名单
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return isConditionMatch(metadataReader);
			}
		}
		return false;
	}

```

ClassPathBeanDefinitionScanner构造方法中通过registerDefaultFilters()默认支持三个注解@Component, @ManagedBean, @Named。

这里再简单看一下TypeFilter体系：
{% asset_img AnnotationTypeFilter.png AnnotationTypeFilter继承关系 %}

```java
@FunctionalInterface
public interface TypeFilter {

	/**
	 * Determine whether this filter matches for the class described by
	 * the given metadata.
	 * @param metadataReader the metadata reader for the target class 目标类的元数据读取器
	 * @param metadataReaderFactory a factory for obtaining metadata readers 元数据读取器工厂
	 * for other classes (such as superclasses and interfaces)
	 * @return whether this filter matches
	 * @throws IOException in case of I/O failure when reading metadata
	 */
	boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException;

}
```

AbstractTypeHierarchyTraversingFilter提供了向上遍历并匹配的能力

```java
	@Override
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException {

		// 当前类是否满足条件
		if (matchSelf(metadataReader)) {
			return true;
		}
		ClassMetadata metadata = metadataReader.getClassMetadata();
		// 当前类名是否满足条件
		if (matchClassName(metadata.getClassName())) {
			return true;
		}

		// 若考虑继承
		if (this.considerInherited) {
			String superClassName = metadata.getSuperClassName();
			if (superClassName != null) {
				// 快速匹配父类名(不用load Class的方法)
				Boolean superClassMatch = matchSuperClass(superClassName);
				if (superClassMatch != null) {
					if (superClassMatch.booleanValue()) {
						return true;
					}
				}
				else {
					// 否则读取class进行匹配
					try {
						if (match(metadata.getSuperClassName(), metadataReaderFactory)) {
							return true;
						}
					}
					catch (IOException ex) {
						if (logger.isDebugEnabled()) {
							logger.debug("Could not read super class [" + metadata.getSuperClassName() +
									"] of type-filtered class [" + metadata.getClassName() + "]");
						}
					}
				}
			}
		}

		// 如果还考虑接口，注意到这里使用metadata来获取接口名
		if (this.considerInterfaces) {
			for (String ifc : metadata.getInterfaceNames()) {
				// 快速匹配接口名
				Boolean interfaceMatch = matchInterface(ifc);
				if (interfaceMatch != null) {
					if (interfaceMatch.booleanValue()) {
						return true;
					}
				}
				else {
					// 读取接口class进行匹配
					try {
						if (match(ifc, metadataReaderFactory)) {
							return true;
						}
					}
					catch (IOException ex) {
						if (logger.isDebugEnabled()) {
							logger.debug("Could not read interface [" + ifc + "] for type-filtered class [" +
									metadata.getClassName() + "]");
						}
					}
				}
			}
		}

		// 默认false 不匹配
		return false;
	}

```

AnnotationTypeFilter继承了AbstractTypeHierarchyTraversingFilter，实现匹配注解的功能。

```java
	protected AbstractTypeHierarchyTraversingFilter(boolean considerInherited, boolean considerInterfaces) {
		this.considerInherited = considerInherited;
		this.considerInterfaces = considerInterfaces;
	}

	public AnnotationTypeFilter(Class<? extends Annotation> annotationType) {
		// 默认不匹配接口
		this(annotationType, true, false);
	}

	// 根据要检查的注解是否有标注@Inherited来决定是否探查父类，因为不能继承的话就没必要检查父类了。
	public AnnotationTypeFilter(
			Class<? extends Annotation> annotationType, boolean considerMetaAnnotations, boolean considerInterfaces) {
		// 如果注解包含@Inherited元注解，则匹配父类
		super(annotationType.isAnnotationPresent(Inherited.class), considerInterfaces);
		this.annotationType = annotationType;
		this.considerMetaAnnotations = considerMetaAnnotations;
	}

```

**@Inherited元注解**
用来启用注解能被子类继承的功能，例如:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface CanInheritAnno {

}

@CanInheritAnno
public class Animal {

}

public class Dog extends Animal {

}

public class Main {
  private static final Logger logger = LoggerFactory.getLogger(Main.class);
  public static void main(String[] args) {
	// 1 获取类上直接注解 （空）
    Annotation[] declaredAnnotations = Dog.class.getDeclaredAnnotations();
	// 2 获取所有注解（能获取到 @CanInheritAnno 注解）
    Annotation[] annotations = Dog.class.getAnnotations();
  }
}
```

通过class.getAnnotations()就能获取到继承的注解；但这与解析@Service的情况不同，@Component是元注解而不是@Service的"父类"，这里没有继承关系，所以探查@Component还得另外实现。

AnnotationTypeFilter继承了AbstractTypeHierarchyTraversingFilter，实现了matchSelf()，利用metadataReader判断是否包含指定注解。

```java
@Override
	// 匹配当前metadataReader是否含有指定注解
	protected boolean matchSelf(MetadataReader metadataReader) {
		AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();
		// 包含注解或元注解
		return metadata.hasAnnotation(this.annotationType.getName()) ||
				(this.considerMetaAnnotations && metadata.hasMetaAnnotation(this.annotationType.getName()));
	}
```

对注解的匹配委托给MetadataReader完成（可以参考第三部分元数据访问体系），它可以匹配目标类上是否有注解或元注解，再加上已有的匹配父类的能力，所以@Component系可以标记在父类上，子类支持组件自动扫描。

### scoped-proxy
扫描出候选BeanDefinition后，Spring通过`AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);`应用了scoped-proxy(如果启用了scoped-proxy的话)，这个功能主要用在多个不同scope的对象协作的场景，例如某单例对象依赖原型对象，由于单例对象只创建一次，通常来说当单例对象创建后，其依赖的原型对象也只会创建一次，与原型对象的定义不符。通过设置原型对象的scoped-proxy，让单例对象依赖原型对象的Proxy，调用时通过Proxy转发到实际的原型对象就能完美解决问题。

既然是通过Proxy，那么也就存在与Spring AOP着相同的问题，不能代理私有方法，final方法。

**配置**

- 注解式，在类/@Bean方法上标@Scope(proxyMode = XXX)
- XML

**实现**

通过把原bean改名为 scopedTarget.$beanName，并用一个ScopedProxyFactoryBean替换了原beanDefinition的targetClass。ScopedProxyFactoryBean利用了Spring内置的ProxyCreator体系和FactoryBean功能创建bean。

```java
public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
			BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		String originalBeanName = definition.getBeanName();
		BeanDefinition targetDefinition = definition.getBeanDefinition();
		// scopedTarget.原BeanName
		String targetBeanName = getTargetBeanName(originalBeanName);

		// Create a scoped proxy definition for the original bean name,
		// "hiding" the target bean in an internal target definition.
		// 用ScopedProxyFactoryBean替换了原名的BD
		RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
		// 记录了被代理的原BD
		proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
		proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
		proxyDefinition.setSource(definition.getSource());
		proxyDefinition.setRole(targetDefinition.getRole());

		// 被代理的原bean的新名字
		proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
		if (proxyTargetClass) {
			targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
		}
		else {
			proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
		}

		// Copy autowire settings from original bean definition.
		proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
		proxyDefinition.setPrimary(targetDefinition.isPrimary());
		if (targetDefinition instanceof AbstractBeanDefinition) {
			proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
		}

		// The target bean should be ignored in favor of the scoped proxy.
		targetDefinition.setAutowireCandidate(false);
		targetDefinition.setPrimary(false);

		// Register the target bean as separate bean in the factory.
		registry.registerBeanDefinition(targetBeanName, targetDefinition);

		// Return the scoped proxy definition as primary bean definition
		// (potentially an inner bean).
		return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
	}
```

## Annotation Config系列

优先介绍的是@Configuration，这是注解配置类上的标识开关，实际上它也被@Component元注解标注了，意味着组件扫描时自动识别为Bean。

### 实现原理

将配置类注册到内置容器是第一步，接下来 `ConfigurationClassPostProcessor` 会识别到@Configuration注解的配置类并识别出其中配置的BeanDefinition。

ConfigurationClassPostProcessor是一个 `BeanDefinitionRegistryPostProcessor`

```java
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	int factoryId = System.identityHashCode(beanFactory);
	// 防止重复处理
	if (this.factoriesPostProcessed.contains(factoryId)) {
		throw new IllegalStateException(
				"postProcessBeanFactory already called on this post-processor against " + beanFactory);
	}
	this.factoriesPostProcessed.add(factoryId);
	if (!this.registriesPostProcessed.contains(factoryId)) {
		// 通常作为 BeanDefinitionRegstryPostProcessor 已经调用过processConfigBeanDefinitions()方法了，如果没有就在这里调一次，解析以BeanDefinition.
		processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
	}
	
	// 增强配置类
	enhanceConfigurationClasses(beanFactory);
	// 添加感知@Import的BeanPostProcessor
	beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
	// 从BeanDefinitionRegistry中查找配置类：包括@Configuration, 含@Bean标注的方法, @Component, @ComponentScan, @Import, @ImportResource注解的类，从中找到BeanDefinition
	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			// 解析过的BeanDefinition会有标记
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
					ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			// 如果没有解析过，就在这里尝试解析，符合候选的返回true
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		}

		// Sort by previously determined @Order value, if applicable
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});

		// Detect any custom bean name generation strategy supplied through the enclosing application context
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
				if (generator != null) {
					this.componentScanBeanNameGenerator = generator;
					this.importBeanNameGenerator = generator;
				}
			}
		}

		if (this.environment == null) {
			this.environment = new StandardEnvironment();
		}

		// Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		// 初始候选
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		// 递归解析直到没有新的候选
		do {
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			// 如果通过解析增加了新的BeanDefinition
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				// 这里是beanName
				for (String candidateName : newCandidateNames) {
					// 找出之前没有解析过的beanName
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						// 检查是否是配置类，且没有解析过对应的类
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				// 下一轮解析前把已有的beanName赋值给candidateNames
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
		}

		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			// Clear cache in externally provided MetadataReaderFactory; this is a no-op
			// for a shared cache since it'll be cleared by the ApplicationContext.
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}


// 处理容器中的配置类
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
	Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
	for (String beanName : beanFactory.getBeanDefinitionNames()) {
		BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
		// 判断是否已经处理过
		if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
			if (!(beanDef instanceof AbstractBeanDefinition)) {
				throw new BeanDefinitionStoreException("Cannot enhance @Configuration bean definition '" +
						beanName + "' since it is not stored in an AbstractBeanDefinition subclass");
			}
			else if (logger.isInfoEnabled() && beanFactory.containsSingleton(beanName)) {
				logger.info("Cannot enhance @Configuration bean definition '" + beanName +
						"' since its singleton instance has been created too early. The typical cause " +
						"is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor " +
						"return type: Consider declaring such methods as 'static'.");
			}
			configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
		}
	}
	if (configBeanDefs.isEmpty()) {
		// nothing to enhance -> return immediately
		return;
	}

	ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
	for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
		AbstractBeanDefinition beanDef = entry.getValue();
		// If a @Configuration class gets proxied, always proxy the target class
		beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
		try {
			// Set enhanced subclass of the user-specified bean class
			Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
			if (configClass != null) {
				Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
				if (configClass != enhancedClass) {
					if (logger.isTraceEnabled()) {
						logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
								"enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
					}
					beanDef.setBeanClass(enhancedClass);
				}
			}
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
		}
	}
}
```

# 元数据访问体系

Spring通过**ClassMetadata**, **AnnotatedTypeMetadata**, **AnnotationMetadata**三个接口来封装元数据的访问。 

{% asset_img StandardAnnotationMetadata.png %}

- 其中ClassMetadata提供访问*目标类*自身信息的方法，包括是否接口、是否注解类型、是否抽象类等常用判断、是否存在父类、是否包含其他类等信息
{% asset_img ClassMetadata.png %}

- AnnotatedTypeMetadata接口提供*目标类*与指定注解的关系信息（都通过注解名交互）
{% asset_img AnnotatedTypeMetadata.png %}

- AnnotationMetadata接口提供注解类本身的信息操作
{% asset_img AnnotationMetadata.png %}

标准实现类 `StandardAnnotationMetadata`，相关操作都委托给`AnnotatedElementUtils`工具类实现。

简单测试一下 AnnotatedElementUtils.isAnnotated()：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Component
@Inherited
public @interface CustomAnno {

}

@CustomAnno
public class Location {

}

public class PlayGround extends Location {

}

public class Main {

  private static final Logger logger = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    StandardAnnotationMetadata metadata = new StandardAnnotationMetadata(PlayGround.class);
    logger.info("annotated @Component:{}", metadata.isAnnotated("org.springframework.stereotype.Component"));
    logger.info("annotated @CompAnno:{}", metadata.isAnnotated("xyz.quxiao.playwith.spring.annotations.metaanno.CustomAnno"));
  }
}

结果：
annotated @Component:true
annotated @CompAnno:true

如果把@Inherited元注解去掉，则两个都是false
```

结论：isAnnotated()方法会检查目标类是否被直接标注或被元注解标注，但不会探查父类。因此结合 AbstractTypeHierarchyTraversingFilter搜索父类的功能，AnnotationTypeFilter能适应多种情况：
1. 类上直接标注或含有指定元注解
2. 父类上直接标注或含有指定元注解

[1]: https://dzone.com/articles/what-are-meta-annotations-in-java
[2]: https://docs.oracle.com/javase/specs/jls/se11/html/jls-9.html#jls-9.6
