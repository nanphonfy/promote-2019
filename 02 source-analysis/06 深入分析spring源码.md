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
>容易混淆的类：BeanFactory和FactoryBean。
>- BeanFactory：Bean工厂，IOC容器最顶层接口，作用：管理Bean，即实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。  
>- FactoryBean：工厂Bean，作用：产生其他bean实例。通常，这种bean没特别要求，仅需提供一个工厂方法（返回其他bean实例）。通常，bean无须自己实现工厂模式，Spring容器担任工厂角色；但少数情况，容器中的bean本身就是工厂，作用：产生其它bean实例。  

>当用户使用容器本身时，可使用转义字符”&”来得到FactoryBean本身，以区别通过FactoryBean产生的实例对象和FactoryBean对象本身。在BeanFactory中通过如下代码定义了该转义字符：  
StringFACTORY_BEAN_PREFIX="&";  
若myJndiObject是一个FactoryBean，则使用&myJndiObject得到myJndiObject对象，而不是myJndiObject产生出来的对象。
-  FactoryBean的源码

```java 
// org.springframework.beans.factory.FactoryBean
// 工厂Bean，用于产生其他对象
public interface FactoryBean<T> {
	// 获取容器管理的对象实例
	T getObject() throws Exception;

	Class<?> getObjectType();

	// Bean工厂创建的对象是否是单态模式，若是单态模式，则整个容器只有一个实例对象，每次请求都返回同一个实例对象
	boolean isSingleton();
}
```

- AbstractBeanFactory.getBean调用FactoryBean
>getBean方法触发容器实例化Bean时会调用AbstractBeanFactory的doGetBean方法进行实例化的过程。

```java 
// org.springframework.beans.factory.support.AbstractBeanFactory
// 真正实现向IOC容器获取Bean的功能，触发依赖注入的地方
protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
	// 根据指定名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖，若指定的是别名，将别名转换为规范的Bean名称
	final String beanName = transformedBeanName(name);
	Object bean;

	// 先从缓存中取是否已有被创建过的单态类型的Bean，对于单例模式的Bean整个IOC容器中只创建一次，不需重复创建
	Object sharedInstance = getSingleton(beanName);
	 // IOC容器创建单例模式Bean实例对象
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			// 若指定名称的Bean在容器中已有单例模式的Bean被创建，直接返回已经创建的Bean
			if(isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
	    // 获取给定Bean的实例对象，主要完成FactoryBean的相关处理。注意：BeanFactory是管理容器中Bean的工厂，而FactoryBean是创建创建对象的工厂Bean，两者之间有区别
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// 缓存没正在创建的单例模式Bean,缓存中已有创建的原型模式Bean,但由于循环引用问题导致实例化对象失败
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// 对IOC容器中是否存在指定名称的BeanDefinition进行检查，首先检查是否能在当前的BeanFactory中获取的所需Bean，若不能则委托当前容器的父级容器去查找，若还是找不到则沿着容器的继承体系向父级容器查找
		BeanFactory parentBeanFactory = getParentBeanFactory();
		// 当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// 解析指定Bean名称的原始名称
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		// 创建的Bean是否需进行类型验证，一般不需要
		if (!typeCheckOnly) {
			// 向容器标记指定的Bean已被创建
			markBeanAsCreated(beanName);
		}

		try {
			// 根据指定Bean名称获取其父级的Bean定义，主要解决Bean继承时子类合并父类公共属性问题 
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);
			
			// 获取当前Bean所有依赖Bean的名称
			String[] dependsOn = mbd.getDependsOn();
			// 若当前Bean有依赖Bean
			if (dependsOn != null) {
				for (String dependsOnBean : dependsOn) {
		    	// 递归调用getBean方法，获取当前Bean的依赖Bean
					getBean(dependsOnBean);
					// 把被依赖Bean注册给当前依赖的Bean
					registerDependentBean(dependsOnBean, beanName);
				}
			}

			// 创建单例模式Bean的实例对象
			if (mbd.isSingleton()) {
				// 这里使用了一个匿名内部类，创建Bean实例对象，并且注册给所依赖的对象
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					public Object getObject() throws BeansException {
						try {
							// 创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的定义
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							//显式地从容器单例模式Bean缓存中清除实例对象
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				// 获取给定Bean的实例对象
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			// IOC容器创建原型模式Bean实例对象
			else if (mbd.isPrototype()) {
				// 原型模式(Prototype)是每次都会创建一个新的对象
				Object prototypeInstance = null;
				try {
					//回调beforePrototypeCreation方法，默认的功能是注册当前创建的原型对象
					beforePrototypeCreation(beanName);
					//创建指定Bean对象实例
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
		    	//回调afterPrototypeCreation方法，默认的功能告诉IOC容器指定Bean的原型对象不再创建了
					afterPrototypeCreation(beanName);
				}
				//获取给定Bean的实例对象
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			//要创建的Bean既不是单例模式，也不是原型模式，则根据Bean定义资源中，配置的生命周期范围，选择实例化Bean的合适方法，这种在Web应用程序中比较常用，如：request、session、application等生命周期 
			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				// Bean定义资源中没有配置生命周期范围，则Bean定义不合法
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
				}
				try {
					// 这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
					Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
						public Object getObject() throws BeansException {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						}
					});
					// 获取给定Bean的实例对象
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; " +
							"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// 对创建的Bean实例对象进行类型检查
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to convert bean '" + name + "' to required type [" +
						ClassUtils.getQualifiedName(requiredType) + "]", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}

// 获取给定Bean的实例对象，主要完成FactoryBean的相关处理
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {
	// 容器已得到Bean实例对象，可能是一个普通Bean，也可能是一个工厂Bean，若是工厂Bean，则使用它创建一个Bean实例对象，若调用本身就想获得一个容器的引用，则指定返回这个工厂Bean实例对象，若指定的名称是容器的解引用(dereference，即是对象本身而非内存地址)， 且Bean实例也不是创建Bean实例对象的工厂Bean
	if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
		throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
	}

	// 如果Bean实例不是工厂Bean，或者指定名称是容器的解引用，调用者向获取对容器的引用，则直接返回当前的Bean实例
	if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		return beanInstance;
	}

	// 处理指定名称不是容器的解引用，或根据名称获取的Bean实例对象是一个工厂Bean  
	// 使用工厂Bean创建一个Bean的实例对象  
	Object object = null;
	if (mbd == null) {
		// 从Bean工厂缓存中获取给定名称的Bean实例对象 
		object = getCachedObjectForFactoryBean(beanName);
	}
	// 让Bean工厂生产给定名称的Bean对象实例
	if (object == null) {
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// 如果从Bean工厂生产的Bean是单态模式的，则缓存
		if (mbd == null && containsBeanDefinition(beanName)) {
			// 从容器中获取指定名称的Bean定义，如果继承基类，则合并基类相关属性
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		// 如果从容器得到Bean定义信息，并且Bean定义信息不是虚构的，则让工厂Bean生产Bean实例对象
		boolean synthetic = (mbd != null && mbd.isSynthetic());
        // 调用FactoryBeanRegistrySupport类的getObjectFromFactoryBean方法，实现工厂Bean生产Bean对象实例的过程 
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```
>getObjectForBeanInstance方法会调用FactoryBeanRegistrySupport类的getObjectFromFactoryBean方法，该方法实现了Bean工厂生产Bean实例对象。  
>>Dereference(解引用)：一个在C/C++中应用比较多的术语，在C++中，”*”是解引用符号，而”&”是引用符号，解引用是指变量指向的是所引用对象的本身数据，而不是引用对象的内存地址。
- AbstractBeanFactory生产Bean实例对象

```java 
// org.springframework.beans.factory.support.AbstractBeanFactory
// Bean工厂生产Bean实例对象
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
	// Bean工厂是单态模式，并且Bean工厂缓存中存在指定名称的Bean实例对象
	if (factory.isSingleton() && containsSingleton(beanName)) {
		// 多线程同步，以防止数据不一致
		synchronized (getSingletonMutex()) {
			// 直接从Bean工厂缓存中获取指定名称的Bean实例对象
			Object object = this.factoryBeanObjectCache.get(beanName);
			// Bean工厂缓存中没有指定名称的实例对象，则生产该实例对象
			if (object == null) {
				// 调用Bean工厂的getObject方法生产指定Bean的实例对象
				object = doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
				// 将生产的实例对象添加到Bean工厂缓存中
				this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
			}
			return (object != NULL_OBJECT ? object : null);
		}
	}
	// 调用Bean工厂的getObject方法生产指定Bean的实例对象
	else {
		return doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
	}
}

//调用Bean工厂的getObject方法生产指定Bean的实例对象
private Object doGetObjectFromFactoryBean(
		final FactoryBean<?> factory, final String beanName, final boolean shouldPostProcess) throws BeanCreationException {
	Object object;
	try {
		if (System.getSecurityManager() != null) {
			AccessControlContext acc = getAccessControlContext();
			try 	// 实现PrivilegedExceptionAction接口的匿名内置类  
                // 根据JVM检查权限，然后决定BeanFactory创建实例对象 
				object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
					public Object run() throws Exception {
							// 调用BeanFactory接口实现类的创建对象方法
							return factory.getObject();
						}
					}, acc);
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
			// 调用BeanFactory接口实现类的创建对象方法
			object = factory.getObject();
		}
	}
	catch (FactoryBeanNotInitializedException ex) {
		throw new BeanCurrentlyInCreationException(beanName, ex.toString());
	}
	catch (Throwable ex) {
		throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
	}

	// 创建出来的实例对象为null，或者因为单态对象正在创建而返回null
	if (object == null && isSingletonCurrentlyInCreation(beanName)) {
		throw new BeanCurrentlyInCreationException(
				beanName, "FactoryBean which is currently in creation returned null from getObject");
	}

	// 为创建出来的Bean实例对象添加BeanPostProcessor后置处理器
	if (object != null && shouldPostProcess) {
		try {
			object = postProcessObjectFromFactoryBean(object, beanName);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Post-processing of the FactoryBean's object failed", ex);
		}
	}

	return object;
}
```
>BeanFactory接口调用其实现类的getObject方法来实现创建Bean实例对象的功能。
- 工厂Bean的实现类getObject方法创建Bean实例对象
>FactoryBean的实现类非常多，eg.Proxy、RMI、JNDI、ServletContextFactoryBean等，FactoryBean接口为Spring容器提供了一个很好的封装机制，具体的getObject有不同的实现类根据不同的实现策略来具体提供，分析最简单的AnnotationTestFactoryBean的实现源码：
```java 
// org.springframework.jmx.export.annotation.AnnotationTestBeanFactory
public class AnnotationTestBeanFactory implements FactoryBean<FactoryCreatedAnnotationTestBean> {
	private final FactoryCreatedAnnotationTestBean instance = new FactoryCreatedAnnotationTestBean();

	public AnnotationTestBeanFactory() {
		this.instance.setName("FACTORY");
	}

	@Override
	public FactoryCreatedAnnotationTestBean getObject() throws Exception {
		return this.instance;
	}

    // AnnotationTestBeanFactory产生bean实例对象的实现
	@Override
	public Class<? extends IJmxTestBean> getObjectType() {
		return FactoryCreatedAnnotationTestBean.class;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}
}
```
>其他的Proxy，RMI，JNDI等，都是根据相应的策略提供getObject的实现。不是Spring的核心功能。
### 4. BeanPostProcessor后置处理器的实现
>IOC容器经常使用的一个特性，Bean后置处理器是一个监听器，可监听容器触发的Bean声明周期事件。后置处理器向容器注册后，容器中管理的Bean就具备了接收IOC容器事件回调的能力。  
BeanPostProcessor的使用非常简单，只需提供一个实现接口BeanPostProcessor的实现类，然后在Bean的配置文件中设置即可。

- BeanPostProcessor的源码

```java 
// org.springframework.beans.factory.config.BeanPostProcessor
public interface BeanPostProcessor {
	// 为在Bean的初始化前提供回调入口
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

	// 为在Bean的初始化之后提供回调入口
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
>这两个回调的入口都是和容器管理的Bean的生命周期事件紧密相关，可为用户提供在IOC容器初始化Bean过程中自定义的处理操作。

- AbstractAutowireCapableBeanFactory类对容器生成的Bean添加后置处理器
>BeanPostProcessor后置处理器的调用发生在对Bean实例的创建和属性的依赖注入之后。当第一次调用getBean方法(lazy-init预实例化除外)向IOC容器索取指定Bean时触发创建Bean实例并进行依赖注入的过程，真正实现创建Bean对象并进行依赖注入的方法是AbstractAutowireCapableBeanFactory类的doCreateBean方法。

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
// 真正创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
	//创建实例对象
	...

	// Bean对象初始化，依赖注入在此触发 这个exposedObject在初始化之后返回作为依赖注入完成后的Bean
	Object exposedObject = bean;
	try {
		// 将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
			//初始化Bean对象,在对Bean实例生成和依赖注入后，对Bean实例初始化 ，为Bean实例应用BeanPostProcessor后置处理器
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}
    ...

	//为应用返回所需要的实例对象
	return exposedObject;
}
```
>为Bean实例添加BeanPostProcessor后置处理器的入口的是initializeBean方法。
- initializeBean方法为Bean添加BeanPostProcessor后置处理器

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
// 初始Bean实例，为其添加BeanPostProcessor后置处理器
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	// JDK的安全机制验证权限
	if (System.getSecurityManager() != null) {
		// 实现PrivilegedAction接口的匿名内部类
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			public Object run() {
			    // 为Bean实例包装相关属性，eg.名称，类加载器，所属容器等
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	// 对BeanPostProcessor后置处理器的postProcessBeforeInitialization回调方法的调用，为Bean初始化前做一些处理
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	// 调用Bean初始化方法（在Spring Bean定义配置文件中通过init-method属性指定）
	try {
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),beanName, "Invocation of init method failed", ex);
	}

	// 对BeanPostProcessor后置处理器的postProcessAfterInitialization回调方法的调用，为Bean实例初始化后做一些处理
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}

// 调用BeanPostProcessor后置处理器实例初始化后的处理方法
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	// 遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
	for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
		// 调用Bean实例所有的后置处理中的初始化后处理方法，为Bean实例对象在初始化之后做一些自定义的处理操作
		result = beanProcessor.postProcessBeforeInitialization(result, beanName);
		if (result == null) {
			return result;
		}
	}
	return result;
}

// 调用BeanPostProcessor后置处理器实例对象初始化前的处理方法
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	// 遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
	for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
		// 调用Bean实例所有的后置处理中的初始化前处理方法，为Bean实例对象在初始化之前做一些自定义的处理操作
		result = beanProcessor.postProcessAfterInitialization(result, beanName);
		if (result == null) {
			return result;
		}
	}
	return result;
}
```
>BeanPostProcessor是一个接口，其初始化前的操作方法和初始化后的操作方法均委托其实现子类来实现，在Spring中，BeanPostProcessor的实现子类非常的多，分别完成不同的操作，eg.AOP面向切面编程的注册通知适配器、Bean对象的数据校验、Bean继承属性/方法的合并等，以最简单的AOP切面织入来简单了解其主要的功能。
- AdvisorAdapterRegistrationManager在Bean对象初始化后注册通知适配器
>AdvisorAdapterRegistrationManager是BeanPostProcessor的一个实现类，作用:为容器中管理的Bean注册一个面向切面编程的通知适配器，以便在Spring容器为所管理的Bean进行面向切面编程时提供方便。

```java
// org.springframework.aop.framework.adapter.AdvisorAdapterRegistrationManager
// 为容器中管理的Bean注册一个面向切面编程的通知适配器
public class AdvisorAdapterRegistrationManager implements BeanPostProcessor {
	// 容器中负责管理切面通知适配器注册的对象
	private AdvisorAdapterRegistry advisorAdapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();

	public void setAdvisorAdapterRegistry(AdvisorAdapterRegistry advisorAdapterRegistry) {
		this.advisorAdapterRegistry = advisorAdapterRegistry;
	}

	// BeanPostProcessor在Bean对象初始化前的操作
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// 没做任何操作，直接返回容器创建的Bean对象
		return bean;
	}

	// BeanPostProcessor在Bean对象初始化后的操作
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean instanceof AdvisorAdapter){
			//如果容器创建的Bean实例对象是一个切面通知适配器，则向容器的注册
			this.advisorAdapterRegistry.registerAdvisorAdapter((AdvisorAdapter) bean);
		}
		return bean;
	}
}
```
>其他BeanPostProcessor接口实现类也类似，都是对Bean对象使用到的一些特性进行处理，或向IOC容器中注册，为创建的Bean实例做一些自定义的功能增加，这些操作是容器初始化Bean时自动触发的，不需要人为干预。

### 5. autowiring实现原理
