---
title: '聊聊Spring[Spring IOC 源码剖析]'
tags:
  - java
  - spring
  - ioc
date: 2020-06-02 09:44:24
---


- 本篇文章从下面几点聊聊spring核心值IOC
	- 什么是Spring IOC
	- Spring IOC 能够帮助我们做什么事情
	- 源码分析
	- 总结

<!-- more -->

#### 概览
- 什么是Spring IOC
- Spring IOC 能够帮助我们做什么事情
- 源码分析
- 总结
- 参考

#### 什么是Spring IOC
##### IOC （Inverse Of Control） 控制反转/反转控制
1. 主要是涉及到 java开发领域对象的创建以及管理
2. 是一种思想
3. 控制：对象的创建以及管理
4. 翻转：控制权交给其他的IOC容器
5. 实际对比 {% asset_img image-20200521092115676.png %}

###### 举个例子
- 场景
	- 有A，B两个类。A依赖于B
- 解决办法
		1. 传统思想解决：在A中实例一个B对象。 
		2. 使用IOC思想开发: 不通过new 关键字来创建对象，而是通过IOC容器(例如Spring IOC) 来帮助我们实例化对象。

###### 解决了什么问题
1. 对象之间的耦合度会变得很低
	- 例如如果Model 接口的实现是`ModelImplA`.  如果使用传统服务，那么所有使用到Model接口的实现都需要实例化一个`ModelImplA`.  当有一天需要更换其他的实现`ModelImplA`，那么就需要更改所有`new ModelImplA`->`new ModelImplB`. 
	- 如果使用IOC， 只需要将Model 的实现指向ModelImplB, 其他的就不需要更改了
2. 资源更容易管理。只需要负责定义，实际使用直接用就可以了。

##### IOC VS DI(Dependency Injection) 傻傻分不清楚
1. IOC 是一种思想
2. DI 是最常见的实现方式


#### 源码分析
带着下面两个问题看源码：
-  IOC 是如何创建Bean 容器
- IOC 是如何初始化Bean

##### ClassPathXmlApplicationContext 构造函数
```

public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		// 创建父类的applicationContext
		super(parent);
		// 设置配置路径
		setConfigLocations(configLocations);
		if (refresh) {
			// 核心
			refresh();
		}
	}
```
##### ClassPathXmlApplicationContext.refresh
- 提供功能
	- 找到所有的BeanDefinition
	- 加载额外的FactoryBean
	- 生成非lazy-init 的singletonsInstance. 
- 运行图
	- {% asset_img image-20200529084655527.png %} 	
- 源码
```
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 初始化开始时间，以及标记`active=true`已启动
			// 校验前置的需要的参数
			prepareRefresh();

			// 核心之一 
			// 初始化BeanFactory,加载Bean，注册Bean。 但是Bean并没有初始化。 
			
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 设置 BeanFactory 的类加载器，添加几个BeanPostProcessor, 手动注册几个特殊的Bean
			prepareBeanFactory(beanFactory);

			try {
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

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
##### AbstractApplicationContext.obtainFreshBeanFactory
- 主要功能: 
	1. 创建新的BeanFactory
	2. 刷新当前的BeanFactory中的配置信息(BeanDefinition)  重新加载Bean
	3. 注册配置信息
- 注意：
	1. 并没有实例化 
- {% asset_img image-20200528093521309.png %}
```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		// 核心逻辑在这里，刷新了会放在内置的属性beanFactory里面
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
###### AbstractRefreshableApplicationContext.refreshBeanFactory
- 功能：
	1. 如果存在beanFactory 销毁并关闭
	2. 创建新的 beanFactory
	3. 加载bean配置信息
```
@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			// 这里是对beanFactory 配置下（是否允许重载覆盖，是否允许循环依赖(默认允许)）
			customizeBeanFactory(beanFactory);
			// 加载bean的配置信息 具体的加载逻辑可以自行查看下源码，本质上对于xml就是解析bean标签中的内容，封装进BeanDefinition里面
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
###### DefaultListableBeanFactory
1. 先看下继承关系
	1. 红框就是AbstractApplicationContext.obtainFreshBeanFactory() 出参的类型
	{% asset_img image-20200527094513181.png %}
	
2. 再看下里面的核心属性
	```
		/** Map from serialized id to factory instance */
		// beanFactory Map
	private static final Map<String, Reference<DefaultListableBeanFactory>> serializableFactories =
			new ConcurrentHashMap<String, Reference<DefaultListableBeanFactory>>(8);

	/** Optional id for this factory, for serialization purposes */
	private String serializationId;

	/** Whether to allow re-registration of a different definition with the same name */
	// 是否允许覆盖，默认是允许的
	private boolean allowBeanDefinitionOverriding = true;

	/** Whether to allow eager class loading even for lazy-init beans */
	// 是否允许以饥饿模式加载
	private boolean allowEagerClassLoading = true;

	/** Optional OrderComparator for dependency Lists and arrays */
	// 对于依赖项列表和数组  可选的OrderComparator
	private Comparator<Object> dependencyComparator;

	/** Resolver to use for checking if a bean definition is an autowire candidate */
	// 自动注入的resolver
	private AutowireCandidateResolver autowireCandidateResolver = new SimpleAutowireCandidateResolver();

	/** Map from dependency type to corresponding autowired value */
	// 处理注入依赖的map
	private final Map<Class<?>, Object> resolvableDependencies = new ConcurrentHashMap<Class<?>, Object>(16);

	/** Map of bean definition objects, keyed by bean name */
	// beanName map  definition
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);

	/** Map of singleton and non-singleton bean names, keyed by dependency type */
	// 单例或者是非单例的bean
	private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<Class<?>, String[]>(64);

	/** Map of singleton-only bean names, keyed by dependency type */
	// 单例的bean信息
	private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<Class<?>, String[]>(64);

	/** List of bean definition names, in registration order */
	// 类加载进来的顺序
	private volatile List<String> beanDefinitionNames = new ArrayList<String>(256);

	/** List of names of manually registered singletons, in registration order */
	// 手动注入的单例
	private volatile Set<String> manualSingletonNames = new LinkedHashSet<String>(16);

	/** Cached array of bean definition names in case of frozen configuration */
	// 缓存bean信息，以防某一时候不允许修改配置
	private volatile String[] frozenBeanDefinitionNames;

	/** Whether bean definition metadata may be cached for all beans */
	// 是否需要缓存所有的definition metadata  默认为false
	private volatile boolean configurationFrozen = false;
	```

###### BeanDefinition （bean 的信息）
- 记录Bean的信息
	- Depends-on 依赖关系
	- 是否可以注入
	- 是否是由FactoryBean 生成
	- 是否是primary
	- 构造器参数
	- ...等等
- BeanDefinition 继承关系
	- {% asset_img image-20200528212343084.png %}
```
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	/**
	 * 支持的类型 单例 
	 */
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

	/**
	 * 支持的类型 原型
	 */
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	/**
	 * Role hint indicating that a {@code BeanDefinition} is a major part
	 * of the application. Typically corresponds to a user-defined bean.
	 */
	int ROLE_APPLICATION = 0;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is a supporting
	 * part of some larger configuration, typically an outer
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 * {@code SUPPORT} beans are considered important enough to be aware
	 * of when looking more closely at a particular
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition},
	 * but not when looking at the overall configuration of an application.
	 */
	int ROLE_SUPPORT = 1;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is providing an
	 * entirely background role and has no relevance to the end-user. This hint is
	 * used when registering beans that are completely part of the internal workings
	 * of a {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 */
	int ROLE_INFRASTRUCTURE = 2;


	// Modifiable attributes

	/**
	 * 设置父BeanDefinition. bean继承，会继承父bean的信息（Class信息）。
	 */
	void setParentName(String parentName);

	/**
	 * get.
	 */
	String getParentName();

	/**
	 * 设置bean的类名
	 */
	void setBeanClassName(String beanClassName);

	/**
	 * 获取bean的类型，英文注释中表示这个运行中并不一定是实际bean类型（factoryBean之类的）
	 */
	String getBeanClassName();

	/**
	 */
	void setScope(String scope);

	/**
	 */
	String getScope();

	/**
	 * 是否需要lazeinit
	 */
	void setLazyInit(boolean lazyInit);

	boolean isLazyInit();

	/**
	 * bean依赖的所有bean。注意点并不是属性依赖@Autowire标记的，而是depends-on="" 属性设置的值
	 */
	void setDependsOn(String... dependsOn);


	String[] getDependsOn();

	// 是否可以注入到其他的bean
	void setAutowireCandidate(boolean autowireCandidate);

	boolean isAutowireCandidate();

	/**
	 * 是否是主要的。多个实现类，会有一个主要的实现。@Primary
	 */
	void setPrimary(boolean primary);


	boolean isPrimary();

	/**
	指定factoryBean. 通过factoryBean 生成
	 */
	void setFactoryBeanName(String factoryBeanName);

	String getFactoryBeanName();

	/**
		指定factoryBean中的方法
	 */
	void setFactoryMethodName(String factoryMethodName);


	String getFactoryMethodName();

	// 构造器参数
	ConstructorArgumentValues getConstructorArgumentValues();

	// bean中的属性值
	MutablePropertyValues getPropertyValues();


	// Read-only attributes

	//是否singleton
	boolean isSingleton();

	// 是否 prototype
	boolean isPrototype();

	// 是否是抽象基类
	boolean isAbstract();

	/**
	 * Get the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools with an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	int getRole();

	/**
	 * Return a human-readable description of this bean definition.
	 */
	String getDescription();

	/**
	 * Return a description of the resource that this bean definition
	 * came from (for the purpose of showing context in case of errors).
	 */
	String getResourceDescription();

	/**
	 * Return the originating BeanDefinition, or {@code null} if none.
	 * Allows for retrieving the decorated bean definition, if any.
	 * <p>Note that this method returns the immediate originator. Iterate through the
	 * originator chain to find the original BeanDefinition as defined by the user.
	 */
	BeanDefinition getOriginatingBeanDefinition();

}
```
##### AbstractApplicationContext.finishBeanFactoryInitialization
- 实例化所有非lazy-init的singletons
- 具体流转图如下
	- {% asset_img image-20200602093355878.png %}
- 后面具体的逻辑可以参见文章最下面的参考，这里就不在赘述了。
- 需要注意的：
	- 实例化类是根据反射来实例化的。
	- 对于实例类中的属性，是使用populateBean方法进行属性赋值的
	- 对于`autowire`的属性，会先查找/创建对应的实例，然后再进行赋值 
```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		// 加载conversionService 
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
				@Override
				public String resolveStringValue(String strVal) {
					return getEnvironment().resolvePlaceholders(strVal);
				}
			});
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		// 实际加载所有的singtons
		beanFactory.preInstantiateSingletons()
```
#### 总结
- IOC实际是为了更好的对实例，依赖更好的管理
- IOC是一种思想，DI是常用的实现，springIOC就是DI的一种
- Spring IOC 实现的思路大致可以分为
	- 先找到所有的bean，记录所有的bean信息（包含，父类信息，是否需要override，是否允许循环依赖） 等
	- 然后再实例化所有非lazy-init的是单例
	- 最后通过对外getBean函数来实例需要的实例
	
#### 参考大佬
- [面试被问了几百遍的 IoC 和 AOP ，还在傻傻搞不清楚？](https://mp.weixin.qq.com/s/9_lUOU2tgVUf5VMZImfWJA)
- [Spring IOC 容器源码分析
](https://www.javadoop.com/post/spring-ioc)