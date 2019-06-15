### 1 基于XML的依赖注入
#### 1.1 依赖注入发生的时间
>当SpringIOC容器完成Bean定义资源的定位、载入和解析注册后，IOC容器已经管理类Bean定义的相关数据，此时IOC容器还没对所管理的Bean进行依赖注入，依赖注入在以下两种情况发生：  
>- (1).用户第一次getBean(向IOC索要Bean)时，IOC容器触发依赖注入。
>- (2).当用户在Bean定义资源中为<bean>元素配置了lazy-init属性，即让容器在解析注册Bean定义时进行预实例化，触发依赖注入。  
BeanFactory接口定义了SpringIOC容器的基本功能规范，是SpringIOC容器所应遵守的最底层和最基本的编程规范。BeanFactory接口中定义了几个getBean方法，就是用户向IOC容器索取管理Bean的方法，分析其子类具体实现，理解IOC容器在用户索取Bean时如何完成依赖注入。

...

>在BeanFactory中我们看到getBean（String...）函数，它的具体实现在AbstractBeanFactory中

#### 1.2 AbstractBeanFactory的getBean
>AbstractBeanFactory通过getBean向IOC容器获取被管理的Bean。

```java 
//org.springframework.beans.factory.support.AbstractBeanFactory
//获取IOC容器中指定名称的Bean 
public Object getBean(String name) throws BeansException {
	//doGetBean才是真正向IoC容器获取被管理Bean的过程
	return doGetBean(name, null, null, false);
}

//获取IOC容器中指定名称和类型的Bean
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
	//doGetBean才是真正向IoC容器获取被管理Bean的过程
	return doGetBean(name, requiredType, null, false);
}

//获取IOC容器中指定名称和参数的Bean
public Object getBean(String name, Object... args) throws BeansException {
	//doGetBean才是真正向IoC容器获取被管理Bean的过程 
	return doGetBean(name, null, args, false);
}

//获取IOC容器中指定名称、类型和参数的Bean
public <T> T getBean(String name, Class<T> requiredType, Object... args) throws BeansException {
	//doGetBean才是真正向IoC容器获取被管理Bean的过程
	return doGetBean(name, requiredType, args, false);
}

//真正实现向IOC容器获取Bean的功能，也是触发依赖注入功能的地方
protected <T> T doGetBean(
		final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {

	//根据指定的名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖  
    //如果指定的是别名，将别名转换为规范的Bean名称
	final String beanName = transformedBeanName(name);
	Object bean;

	// Eagerly check singleton cache for manually registered singletons.
	//先从缓存中取是否已经有被创建过的单态类型的Bean
    //对于单例模式的Bean整个IOC容器中只创建一次，不需要重复创建
	Object sharedInstance = getSingleton(beanName);
	 //IOC容器创建单例模式Bean实例对象
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			//如果指定名称的Bean在容器中已有单例模式的Bean被创建
            //直接返回已经创建的Bean
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
	    //获取给定Bean的实例对象，主要是完成FactoryBean的相关处理  
        //注意：BeanFactory是管理容器中Bean的工厂，而FactoryBean是  
        //创建创建对象的工厂Bean，两者之间有区别
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		//缓存没有正在创建的单例模式Bean  
        //缓存中已经有已经创建的原型模式Bean
        //但是由于循环引用的问题导致实 例化对象失败
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		//对IOC容器中是否存在指定名称的BeanDefinition进行检查，首先检查是否  
        //能在当前的BeanFactory中获取的所需要的Bean，如果不能则委托当前容器  
        //的父级容器去查找，如果还是找不到则沿着容器的继承体系向父级容器查找
		BeanFactory parentBeanFactory = getParentBeanFactory();
		//当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			//解析指定Bean名称的原始名称
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				// Delegation to parent with explicit args.
				//委派父级容器根据指定名称和显式的参数查找
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				//委派父级容器根据指定名称和类型查找
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		//创建的Bean是否需要进行类型验证，一般不需要
		if (!typeCheckOnly) {
			//向容器标记指定的Bean已经被创建
			markBeanAsCreated(beanName);
		}

		try {
			//根据指定Bean名称获取其父级的Bean定义
	        //主要解决Bean继承时子类合并父类公共属性问题 
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			//获取当前Bean所有依赖Bean的名称
			String[] dependsOn = mbd.getDependsOn();
			//如果当前Bean有依赖Bean
			if (dependsOn != null) {
				for (String dependsOnBean : dependsOn) {
					//递归调用getBean方法，获取当前Bean的依赖Bean
					getBean(dependsOnBean);
					//把被依赖Bean注册给当前依赖的Bean
					registerDependentBean(dependsOnBean, beanName);
				}
			}

			// Create bean instance.
			//创建单例模式Bean的实例对象
			if (mbd.isSingleton()) {
				//这里使用了一个匿名内部类，创建Bean实例对象，并且注册给所依赖的对象
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					public Object getObject() throws BeansException {
						try {
							//创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的定义
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
				//获取给定Bean的实例对象
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			//IOC容器创建原型模式Bean实例对象
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				//原型模式(Prototype)是每次都会创建一个新的对象
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

			//要创建的Bean既不是单例模式，也不是原型模式，则根据Bean定义资源中  
	        //配置的生命周期范围，选择实例化Bean的合适方法，这种在Web应用程序中  
	        //比较常用，如：request、session、application等生命周期 
			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				//Bean定义资源中没有配置生命周期范围，则Bean定义不合法
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
				}
				try {
					//这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
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
					//获取给定Bean的实例对象
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; " + "consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	//对创建的Bean实例对象进行类型检查
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to convert bean '" + name + "' to required type [" + ClassUtils.getQualifiedName(requiredType) + "]", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```
>从上面可看出Bean定义的单例模式(Singleton)，容器在创建前先从缓存查找，确保整个容器只存在一个实例。若Bean定义的是原型模式(Prototype)，容器每次会创建一个新的实例。此外，Bean定义还可扩展为指定其生命周期范围。  
源码部分只定义了根据Bean定义的模式，采取不同创建Bean实例的策略，具体Bean实例的创建过程由实现ObejctFactory（使用委派模式，具体的Bean实例创建过程交由其实现类AbstractAutowireCapableBeanFactory完成）接口的匿名内部类createBean完成。  
继续分析AbstractAutowireCapableBeanFactory的createBean源码，理解创建Bean实例的具体实现过程。  

#### 1.3 AbstractAutowireCapableBeanFactory创建Bean实例  
>AbstractAutowireCapableBeanFactory类实现了ObejctFactory接口，创建容器指定的Bean实例，对创建Bean实例进行初始化。源码如下：

```java 
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
// 创建Bean实例对象
@Override
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) throws BeanCreationException {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	// Make sure bean class is actually resolved at this point.
	// 判断需要创建的Bean是否可实例化，即是否可通过当前的类加载器加载
	resolveBeanClass(mbd, beanName);

	// Prepare method overrides.
	// 校验和准备Bean中的方法覆盖
	try {
		mbd.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbd.getResourceDescription(),beanName, "Validation of method overrides failed", ex);
	}

	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		// 如果Bean配置了初始化前和初始化后的处理器，则试图返回一个需要创建Bean的代理对象
		Object bean = resolveBeforeInstantiation(beanName, mbd);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,"BeanPostProcessor before instantiation of bean failed", ex);
	}

	// 创建Bean的入口
	Object beanInstance = doCreateBean(beanName, mbd, args);
	if (logger.isDebugEnabled()) {
		logger.debug("Finished creating instance of bean '" + beanName + "'");
	}
	return beanInstance;
}

//真正创建Bean的方法 
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
	// Instantiate the bean.
	// 封装被创建的Bean对象
	BeanWrapper instanceWrapper = null;
	// 单例模式Bean，先从容器缓存中获取同名Bean
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		//创建实例对象
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	// 获取实例化对象的类型
	Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

	// Allow post-processors to modify the merged bean definition.
	// 调用PostProcessor后置处理器
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			mbd.postProcessed = true;
		}
	}

	// Eagerly cache singletons to be able to resolve circular references
	// even when triggered by lifecycle interfaces like BeanFactoryAware.
	// 向容器中缓存单例模式的Bean对象，以防循环引用
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isDebugEnabled()) {
			logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
		}
		//这里是一个匿名内部类，为了防止循环引用，尽早持有对象的引用
		addSingletonFactory(beanName, new ObjectFactory<Object>() {
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}

	// Initialize the bean instance.
	//Bean对象的初始化，依赖注入在此触发  
    //这个exposedObject在初始化完成之后返回作为依赖注入完成后的Bean
	Object exposedObject = bean;
	try {
		//将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
			//初始化Bean对象 
			//在对Bean实例对象生成和依赖注入完成以后，开始对Bean实例对象  
            //进行初始化 ，为Bean实例对象应用BeanPostProcessor后置处理器
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

	if (earlySingletonExposure) {
		//获取指定名称的已注册的单例模式Bean对象
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			//根据名称获取的已注册的Bean和正在实例化的Bean是同一个
			if (exposedObject == bean) {
				//当前实例化的Bean初始化完成
				exposedObject = earlySingletonReference;
			}
			//当前Bean依赖其他Bean，并且当发生循环引用时不允许新创建实例对象
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
				//获取当前Bean所依赖的其他Bean 
				for (String dependentBean : dependentBeans) {
					//对依赖Bean进行类型检查
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,"Bean with name '" + beanName + "' has been injected into other beans [" +StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	//注册完成依赖注入的Bean
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	//为应用返回所需要的实例对象
	return exposedObject;
}
```
>具体的依赖注入实现在以下两方法：  
(1)createBeanInstance：生成Bean所包含的java对象实例；  
(2)populateBean：对Bean属性的依赖注入进行处理。

#### 1.4 createBeanInstance方法创建Bean的java实例对象
>createBeanInstance方法根据指定的初始化策略，使用静态工厂、工厂方法或容器的自动装配特性生成java实例对象.

```java 
//org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
//创建Bean的实例对象
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
	// Make sure bean class is actually resolved at this point.
	// 检查确认Bean是可实例化的
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	// 使用工厂方法对Bean进行实例化
	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

	if (mbd.getFactoryMethodName() != null)  {
		// 调用工厂方法实例化
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// Shortcut when re-creating the same bean...
	// 使用容器的自动装配方法进行实例化
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			//配置自动装配属性，用容器的自动装配实例化，容器的自动装配是根据参数类型匹配Bean的构造方法
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			// 使用默认的无参构造方法实例化
			return instantiateBean(beanName, mbd);
		}
	}

	// Need to determine the constructor...
	// 使用Bean的构造方法进行实例化
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR || mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
		// 使用容器的自动装配特性，调用匹配的构造方法实例化 
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// No special handling: simply use no-arg constructor.
	// 使用默认的无参构造方法实例化
	return instantiateBean(beanName, mbd);
}
```
>对使用工厂方法和自动装配特性的Bean的实例化相当比较清楚，调用相应的工厂方法或者参数匹配的构造方法即可完成实例化对象的工作，但是对于我们最常使用的默认无参构造方法就需要使用相应的初始化策略(JDK的反射机制或者CGLIB)来进行初始化了，在方法getInstantiationStrategy().instantiate()中就具体实现类使用初始策略实例化对象。

#### 1.5 SimpleInstantiationStrategy类的无参构造方法创建Bean对象
>使用默认无参构造方法创建Bean时，getInstantiationStrategy().instantiate()调用了SimpleInstantiationStrategy类中的实例化Bean的方法：

```java 
//使用初始化策略实例化Bean对象
public Object instantiate(RootBeanDefinition beanDefinition, String beanName, BeanFactory owner) {
	// Don't override the class with CGLIB if no overrides.
	// 若Bean定义中无方法覆盖，则不需CGLIB父类类的方法
	if (beanDefinition.getMethodOverrides().isEmpty()) {
		Constructor<?> constructorToUse;
		synchronized (beanDefinition.constructorArgumentLock) {
			// 获取对象的构造方法或工厂方法
			constructorToUse = (Constructor<?>) beanDefinition.resolvedConstructorOrFactoryMethod;
			
			// 若无构造方法且无工厂方法 
			if (constructorToUse == null) {
				// 使用JDK反射，判断要实例化Bean是否是接口
				final Class clazz = beanDefinition.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						// 这里是一个匿名内置类，使用反射获取Bean的构造方法
						constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor>() {
							public Constructor run() throws Exception {
								return clazz.getDeclaredConstructor((Class[]) null);
							}
						});
					}
					else {
						constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
					}
					beanDefinition.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Exception ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		// 使用BeanUtils实例化，通过反射机制调用”构造方法.newInstance(arg)”来进行实例化
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// Must generate CGLIB subclass.
		// 使用CGLIB来实例化对象
		return instantiateWithMethodInjection(beanDefinition, beanName, owner);
	}
}
```
>从上面看到：若Bean有方法被覆盖，则使用JDK反射机制实例化，否则使用CGLIB实例化。  
instantiateWithMethodInjection方法调用SimpleInstantiationStrategy的子类CglibSubclassingInstantiationStrategy使用CGLIB来进行初始化：

```java 
// org.springframework.beans.factory.support.CglibSubclassingInstantiationStrategy
// 使用CGLIB进行Bean对象实例化
public Object instantiate(Constructor ctor, Object[] args) {
	// CGLIB中的类
	Enhancer enhancer = new Enhancer();
	// 将Bean本身作为其基类
	enhancer.setSuperclass(this.beanDefinition.getBeanClass());
	enhancer.setCallbackFilter(new CallbackFilterImpl());
	enhancer.setCallbacks(new Callback[] {
			NoOp.INSTANCE,
			new LookupOverrideMethodInterceptor(),
			new ReplaceOverrideMethodInterceptor()
	});

	// 使用CGLIB的create方法生成实例对象
	return (ctor == null) ?
			enhancer.create() :
			enhancer.create(ctor.getParameterTypes(), args);
}
```
>CGLIB是常用的字节码生成器的类库，提供一系列API实现java字节码的生成和转换功能。JDK的动态代理只能针对接口，若一个类没实现任何接口，要对其进行动态代理只能使用CGLIB。

#### 1.6 populateBean方法对Bean属性的依赖注入  
