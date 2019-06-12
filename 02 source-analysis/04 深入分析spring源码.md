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
