### 1. IOC高级特性介绍
>通过IOC容器源码分析，基本了解IOC容器对Bean定义资源的定位、读入和解析过程，清楚了getBean方法向IOC容器获取被管理Bean时，IOC容器对Bean进行初始化和依赖注入过程——基本功能特性。IOC容器高级特性，eg.lazy-init属性对Bean预初始化、FactoryBean产生者修饰Bean对象的生成、IOC容器初始化Bean过程中使用BeanPostProcessor后置处理器对Bean声明周期事件管理和IOC容器的autowiring自动装配功能等。
### 2. lazy-init属性实现预实例化
>IOC容器的初始化过程就是对Bean定义资源的定位、载入和注册，此时容器对Bean的依赖注入并没有发生，依赖注入主要是在应用程序第一次向容器索取Bean时，通过getBean方法调用完成。  
当<Bean>元素配置了lazy-init属性时，容器在初始化时对所配置的Bean预实例化，依赖注入在容器初始化时已完成。当应用程序第一次向容器索取被管理Bean时，就不用再初始化和对Bean进行依赖注入了，直接从容器中获取已完成依赖注入的现成Bean，可提高应用第一次向容器获取Bean的性能。
- refresh()
>先从IOC容器的初始过程开始，通过前面文章分析，IOC容器读入已定位Bean定义资源是从refresh方法开始的，从AbstractApplicationContext类的refresh方法入手分析

```java 
// org.springframework.context.support.AbstractApplicationContext
//容器初始化的过程，读入Bean定义资源，并解析注册
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		//调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		//告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动  
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		//为BeanFactory配置容器特性，例如类加载器、事件处理器等
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			//为容器的某些子类指定特殊的BeanPost事件处理器
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			//调用所有注册的BeanFactoryPostProcessor的Bean
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			//为BeanFactory注册BeanPost事件处理器.  
	        //BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件 
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			//初始化信息源，和国际化相关.
			initMessageSource();

			// Initialize event multicaster for this context.
			//初始化容器事件传播器
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			//调用子类的某些特殊Bean初始化方法
			onRefresh();

			// Check for listener beans and register them.
			//为事件传播器注册事件监听器.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			//初始化所有剩余的单例Bean.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			//初始化容器的生命周期事件处理器，并发布容器的生命周期事件
			finishRefresh();
		}

		catch (BeansException ex) {
			// Destroy already created singletons to avoid dangling resources.
			//销毁以创建的单态Bean
			destroyBeans();

			// Reset 'active' flag.
			//取消refresh操作，重置容器的同步标识.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}
	}
}
```
>refresh()方法中
ConfigurableListableBeanFactorybeanFactory=obtainFreshBeanFactory();启动了Bean定义资源的载入、注册过程，而finishBeanFactoryInitialization是对注册后的Bean定义中的预实例化(lazy-init=false,默认是预实例化,即为true)的Bean进行处理的地方。
- finishBeanFactoryInitialization处理预实例化Bean
>当Bean定义资源被载入IOC容器后，容器将Bean定义资源解析为容器内部数据结构BeanDefinition注册到容器中，AbstractApplicationContext类中的finishBeanFactoryInitialization方法对配置了预实例化属性的Bean进行预初始化过程：

```java 
// org.springframework.context.support.AbstractApplicationContext
//对配置了lazy-init属性的Bean进行预实例化处理 
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	//这是Spring3以后新加的代码，为容器指定一个转换服务(ConversionService)  
    //在对某些Bean属性进行转换时使用  
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	//为了类型匹配，停止使用临时的类加载器  
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	//缓存容器中所有注册的BeanDefinition元数据，以防被修改
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	//对配置了lazy-init属性的单态模式Bean进行预实例化处理
	beanFactory.preInstantiateSingletons();
}
```
>ConfigurableListableBeanFactory是一个接口，其preInstantiateSingletons方法由其子类DefaultListableBeanFactory提供。

- DefaultListableBeanFactory对配置lazy-init属性单态Bean的预实例化

```java 
//对配置lazy-init属性单态Bean的预实例化
public void preInstantiateSingletons() throws BeansException {
	if (this.logger.isInfoEnabled()) {
		this.logger.info("Pre-instantiating singletons in " + this);
	}
	List<String> beanNames;
	//在对配置lazy-init属性单态Bean的预实例化过程中，必须多线程同步，以确保数据一致性
	synchronized (this.beanDefinitionMap) {
		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		beanNames = new ArrayList<String>(this.beanDefinitionNames);
	}
	for (String beanName : beanNames) {
		//获取指定名称的Bean定义
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		//Bean不是抽象的，是单态模式的，且lazy-init属性配置为false
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			//如果指定名称的bean是创建容器的Bean
			if (isFactoryBean(beanName)) {
				//FACTORY_BEAN_PREFIX=”&”，当Bean名称前面加”&”符号  
                //时，获取的是产生容器对象本身，而不是容器产生的Bean.  
                //调用getBean方法，触发容器对Bean实例化和依赖注入过程  
				final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
				//标识是否需要预实例化  
				boolean isEagerInit;
				if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
					//一个匿名内部类
					isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
						public Boolean run() {
							return ((SmartFactoryBean<?>) factory).isEagerInit();
						}
					}, getAccessControlContext());
				}
				else {
					isEagerInit = (factory instanceof SmartFactoryBean &&
							((SmartFactoryBean<?>) factory).isEagerInit());
				}
				if (isEagerInit) {
					//调用getBean方法，触发容器对Bean实例化和依赖注入过程 
					getBean(beanName);
				}
			}
			else {
				//调用getBean方法，触发容器对Bean实例化和依赖注入过程
				getBean(beanName);
			}
		}
	}
}
```
>若设置lazy-init属性，则容器在完成Bean定义的注册后，会通过getBean方法，触发对指定Bean的初始化和依赖注入过程，这样当应用第一次向容器索取所需的Bean时，容器不再需要对Bean进行初始化和依赖注入，直接从已完成实例化和依赖注入的Bean中取一个现成的Bean，提高了第一次获取Bean的性能。

### 3. FactoryBean的实现