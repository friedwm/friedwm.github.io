---
title: Spring源码阅读之四类注解
date: 2019-10-24 17:25:02
categories:
tags:
---

## 前言
早期Spring IOC对象间的依赖关系完全通过XML表达，很好的实现了依赖翻转，但也导致了成吨的难以维护的XML配置文件。利用Java5引入的注解([Annotation][2])Spring有效的缓解了这个问题，即通过扫描代码中的注解来识别依赖关系，既简洁又直观。~~懒~~ 简单的力量是如此强大，乃至最近几年全注解式的SpringBoot成为了主要的开发方式。

## Spring中的四类注解
Spring中的注解主要分类4类：
	1. 用于标识组件的：@Component系列（根据用途的不同派生出: @Controller, @Service, @Repository等）
	2. 用于Java Config的：@ComponentScan, @Profile, @Import, @ImportResource, @PropertySource @Bean, @Lazy, @Scope, @Primary等
	3. 用于启动特定类型功能的：@EnableXXX系列 （单独拎出来是因为辨识度太高了，其实主要利用了@Import元注解）
	4. 用于表示依赖关系的：@Autowired, @Qualifier, @Inject, @Resource(后俩为javax标准注解)

需要注意的是，注解只是一个能包含少量数据的标记，本身并不能直接改变程序运行流程，必须由相应的处理代码实现。下面分别看看这四类注解的用法及相关处理代码。

### @Component系列
除了基本的@Component，其余的注解都有特定业务含义：

- @Controller用来标注在mvc的控制器上
- @Service放置在业务逻辑实现对象上
- @Repository放置在数据访问对象(DAO)上
- @ControllerAdvice放置在控制器横截处理器类上
- @Configuration比较特殊，用来标记配置类

这里用到了Java5的[元注解][1]的能力，即注解上的注解，例如：

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

* ConfigurationClassPostProcessor
* AutowiredAnnotationBeanPostProcessor
* CommonAnnotationBeanPostProcessor
* PersistenceAnnotationBeanPostProcessor(类存在时)

#### doScan() 识别并注册BeanDefinition

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
1. 识别@Component系注解的方式，也就是findCandidateComponents(basePackage)
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

```

关键在isCandidateComponent()方法：

```java
	// 
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

黑白TypeFilter有哪些呢？在ClassPathBeanDefinitionScanner构造方法中注册了一些默认的。

```java
	@SuppressWarnings("unchecked")
	protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```

可以看到白名单中注册了 AnnotationTypeFilter类型的: @Component, @ManagedBean, @Named 三个。
这里再简单看一下TypeFilter体系：

```java
@FunctionalInterface
public interface TypeFilter {

	/**
	 * Determine whether this filter matches for the class described by
	 * the given metadata.
	 * @param metadataReader the metadata reader for the target class
	 * @param metadataReaderFactory a factory for obtaining metadata readers
	 * for other classes (such as superclasses and interfaces)
	 * @return whether this filter matches
	 * @throws IOException in case of I/O failure when reading metadata
	 */
	boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException;

}
```

{% asset_img AnnotationTypeFilter.png AnnotationTypeFilter继承关系 %}

匹配逻辑封装在父类AbstractTypeHierarchyTraversingFilter.match()中，该方法提供了对父类及接口进行匹配的能力。

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

AnnotationTypeFilter继承了AbstractTypeHierarchyTraversingFilter，用途是匹配指定类是否存在某注解。

```java
	protected AbstractTypeHierarchyTraversingFilter(boolean considerInherited, boolean considerInterfaces) {
		this.considerInherited = considerInherited;
		this.considerInterfaces = considerInterfaces;
	}

	public AnnotationTypeFilter(Class<? extends Annotation> annotationType) {
		this(annotationType, true, false);
	}

	// 根据要检查的注解是否有标注@Inherited来决定是否探查父类，因为不能继承的话就没必要检查父类了。
	public AnnotationTypeFilter(
			Class<? extends Annotation> annotationType, boolean considerMetaAnnotations, boolean considerInterfaces) {

		super(annotationType.isAnnotationPresent(Inherited.class), considerInterfaces);
		this.annotationType = annotationType;
		this.considerMetaAnnotations = considerMetaAnnotations;
	}

```

#### 关于@Inherited
**@Inherited**是一个元注解，用来标识注解具备被子类继承的功能，例如:

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

通过class.getAnnotations()就能获取到继承的注解；这与解析@Service的情况不同，@Component是元注解而不是@Service的"父类"，这里没有继承关系，所以探查@Component还得另外实现。

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

## 元数据访问体系
Spring通过**ClassMetadata**, **AnnotatedTypeMetadata**, **AnnotationMetadata**三个接口来封装元数据的访问。 

{% asset_img StandardAnnotationMetadata.png %}

- 其中ClassMetadata提供访问*目标类*自身信息的方法，包括是否接口、是否注解类型、是否抽象类等常用判断、是否存在父类、是否包含其他类等信息
{% asset_img ClassMetadata.png %}

- AnnotatedTypeMetadata接口提供*目标类*与指定注解名的关系信息（都通过注解名交互）
{% asset_img AnnotatedTypeMetadata.png %}

- AnnotationMetadata接口提供注解类本身的信息操作
{% asset_img AnnotationMetadata.png %}


[1]: https://dzone.com/articles/what-are-meta-annotations-in-java
[2]: https://docs.oracle.com/javase/specs/jls/se11/html/jls-9.html#jls-9.6
