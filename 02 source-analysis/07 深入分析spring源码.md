ListAbleBeanFactory
HierarchicaleanFactory
>ClassPathXmlApplication  
资源定位->配置文件定位  
载入->读取配置文件  
注册->把加载后的配置文件，解释成BeeanDefinition  

- 依赖注入  
>1 读取BeanDefinition的信息，获取依赖关系
BeanWapper；  
2 实例化，代理对象。注入：设值

>createBeanInstance：创建实例，放入到IOC容器
populateBean注入方法，list、map父类接口

- 高级特性主要用来做优化
>ClassPathXmlApplication，本身是一个工厂，这个类是我们自己new出来的，意味着有可能在多线程环境中使用。  
对象被代理后，相当于把类名都改了。  
后置、前置处理器，类似于模板，得到一个回调，
为AOP的实现做了铺垫，有了基础。

>最核心的两个jar包，spring-bean   spring-context定义的是规范。  
工厂的实现，di的实现  
spring-core是最顶层，所有项目都要依赖
spring-aop依赖spring-aspects，是其上层建筑。  
定义接口 targetClass MethodInvoker  
从IOC中取得的代理后的对象，对每个方法进行重写
加入一些切面调用所需要的东西  
do开头的方法，都是具体干活的方法  
AOP通知，与触发器类似，是主动触发，和数据库专门监听的定时器不一样。  
自动注入，注解编程@AutoWiring的功能，自动识别依赖，自动转型。声明的是接口，自动找到该接口实现类（前提：接口只有一个实现）  
spring的单例是用Map实现的，一定要保证线程安全。除非手动声明scope，否则默认单例。  
prototype，每get一次就new一个。  
IOC判断，若被代理的类实现了一个接口，默认用JDK；若被代理的类没实现任何接口，默认用cglib。  
cglib继承子类，所以类型不用强转。  
IOC结束。
--------------------------------------------
2WH（what、why、how）  

AOP设计原理及具体实践  
AOP主线：拆分、合并、解耦  
AOP如何制定规则  

一个切面就代表着N个Bean的一个集合，这N个Bean他们都拥有共同点，组成一个切面。  
事务管理时，就用到切面的定义，eg.com.xxx.service.impl.*  

bean切面是可交叉的，eg.service.impl 做事务管理、service.impl.*API做日志监听。  

归类了在切面中的bean之间的关联点  

事务管理：连接点——开启一个事务->执行事务->提交事务|事务回滚->事务提交  

异常可自定义，只会监听SQL异常。  
说的代理一般是指普通的bean，AOP不是代理。  

异常要通知，利用了IOC中的后置处理（熟悉后置处理的方法）。  

方法拦截器，是在AOP里独有的。  

切入点：切面中，某一个具体的Bean中的，某一个具体的方法。  
先找包名，再找类名，再找方法名。  

代理技术实现的前提：JDK代理和Cglib，本质是持有被代理对象的引用，在调用被代理的方法时，在调用之前加点东西，在调用后加点东西，中间就用自己保持的引用去调用它。  
目标对象，就是代理对象所持有的引用。  

通知机制：  

连接点：规定切面中方法调用的一些规则。   
切入点：进入切面内部的一个入口（主动调用方法）  

连接点是规则，是抽象，切入点是具体方法。    
切面是大规则，切入点是比较详细的规则。    
一旦调用过程中，满足连接点的规则，就会触发一个通知：调用代理写的代码（对用户无感知）    

返回后通知：方法调用完成，并且该方法拥有返回值，触发。  
环绕通知：类似一个拦截器链，doFilter方法。  
权限不够，抛出异常后通知。  

AOP编程，spring帮我们实现了，完全解耦，通知机制的设计，完美解决代码耦合问题。  

事件机制，先注册，后触发（异步执行）。  


---

- AOP相关类的关系  
![image](https://raw.githubusercontent.com/nanphonfy/note-images/master/promote-2019/source-analysis/07/spring-aop.png)

>- 切面（Aspect）："一个关注点的模块化，该关注点可能会横切多个对象”。切面在ApplicationContext的<aop:aspect>中配置；
>- 连接点（Joinpoint）：程序执行中的某一行为，eg.UserService.get或UserService.delete抛出异常等行为;
>- 通知（Advice）：切面对某连接点产生的动作。一个切面可包含多个Advice;  
>- 切入点（Pointcut）：匹配连接点的断言，在AOP中通知和一个切入点表达式关联。切面中所有通知所关注的连接点，都由切入点表达式来决定;
>- 目标对象（TargetObject）：被一个或多个切面通知的对象。eg.AServcieImpl和BServiceImpl实际运行时，AOP用代理实现，实际AOP操作的是TargetObject的代理对象；
>- AOP代理（AOPProxy）：AOP中有两种代理：JDK动态代理、CGLIB代理。默认下，TargetObject实现接口时，采用JDK动态代理；反之，采用CGLIB代理。  
强制使用CGLIB代理需将<aop:config>的proxy-target-class属性设为true。
>- 通知（Advice）类型：  
>>①前置通知（Beforeadvice）：某连接点（JoinPoint）执行前的通知，但该通知不能阻止连接点前的执行。ApplicationContext中在<aop:aspect>里用<aop:before>声明。eg.TestAspect中的doBefore方法；  
②后置通知（Afteradvice）：当某连接点退出时执行的通知（正常返回|异常退出）。ApplicationContext在<aop:aspect>里用<aop:after>声明。eg.ServiceAspect的returnAfter方法，抛出异常时returnAfter方法仍执行；  
③返回后通知（Afterreturnadvice）：在某连接点正常完成后执行的通知，不包括抛出异常情况。ApplicationContext中<aop:aspect>里用<after-returning>声明；  
④环绕通知（Aroundadvice）：包围一个连接点的通知，类似Web中Servlet中Filter的doFilter方法。可在方法的调用前后完成自定义行为，也可选择不执行。ApplicationContext中<aop:aspect>里用<aop:around>声明；  
⑤抛出异常后通知（Afterthrowingadvice）：在方法抛出异常退出时执行的通知。ApplicationContext在<aop:aspect>里用<aop:after-throwing>声明；  

>注：可将多个通知应用到一个目标对象上，即可将多个切面织入到同一目标对象。  
AOP可基于两种方式：注解方式、xml配置方式。
使用注解AOP分两步：第一步在xml中声明激活自动扫描组件功能，同时激活自动代理功能（来测试AOP的注解功能），第二步为Aspect切面类添加注解。

```java 
// org.springframework.aop.framework.JdkDynamicAopProxy
/**
 * 获取代理类要实现的接口，除Advised对象中配置的，还会加上SpringProxy,Advised(opaque=false)，检查上面得到的接口中有没有定义equals或hashcode的接口，调用Proxy.newProxyInstance创建对象（ClassLoader loaderClass<?>[] interfaces,InvocationHandler h）
 * @param classLoader the class loader to create the proxy with
 * (or {@code null} for the low-level proxy facility's default)
 * @return
 */
public Object getProxy(ClassLoader classLoader) {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
	}
	Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
>代理对象生成了，那切面如何织入？
InvocationHandler是JDK动态代理的核心，生成的代理对象的方法调用都会委托到InvocationHandler.invoke()方法。而JdkDynamicAopProxy类也实现了InvocationHandler——分析该类实现的invoke()方法——AOP是如何织入切面的。

`final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable`

```java 
// org.springframework.aop.framework.JdkDynamicAopProxy
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Class targetClass = null;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
		if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		// advised接口或其父接口中定义的方法，直接反射，不应用通知
		if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// May be null. Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		// 获取目标对象类
		target = targetSource.getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}

		// Get the interception chain for this method.
		// 获取可映射到此方法上的Interception列表
		// 拦截器链是AOP加上去的，目的：为环绕通知做准备
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		// 若没可应用到此方法的通知(Interceptor)，直接反射调用method.invoke(target，args)
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
		}
		else {
			// We need to create a method invocation...
			// 创建MethodInvocation
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		} else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```
>主流程：获取可应用到此方法的通知链（InterceptorChain）。若有,则应用通知并执行joinpoint;若没有,则直接反射执行joinpoint。关键：通知链如何获取、执行，逐一分析下。  
>>①通知链通过Advised.getInterceptorsAndDynamicInterceptionAdvice()方法获取，其实现:

```java 
// org.springframework.aop.framework.AdvisedSupport
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class targetClass) {
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
				this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}
```
>实际执行方法：AdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice()，获到结果会被缓存。其实现：

```java 
// org.springframework.aop.framework.DefaultAdvisorChainFactory
/**
 *  从提供的配置实例config获取advisor列表，遍历。若是introductionadvisor，则判断此advisor能否应用到targetClass（目标类）；
 *  若是pointcutadvisor，则判断此advisor能否应用到method（目标方法），将满足条件的advisor通过advisoradaptor转化成interceptor列表返回
 * @param config the AOP configuration in the form of an Advised object
 * @param method the proxied method
 * @param targetClass the target class
 * @return
 */
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
		Advised config, Method method, Class targetClass) {

	// This is somewhat tricky... we have to process introductions first,
	// but we need to preserve order in the ultimate list.
	List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
	// 是否包含IntroductionAdvisor
	boolean hasIntroductions = hasMatchingIntroductions(config, targetClass);
	// 注册一系列AdvisorAdapter，将Advisor转化为MethodInterceptor
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	for (Advisor advisor : config.getAdvisors()) {
		if (advisor instanceof PointcutAdvisor) {
			// Add it conditionally.
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
				// 将Advisor转换成Interceptor
				MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
				// 检查当前advisor的pointcut是否可匹配当前方法
				MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				if (MethodMatchers.matches(mm, method, targetClass, hasIntroductions)) {
					if (mm.isRuntime()) {
						// Creating a new object instance in the getInterceptors() method
						// isn't a problem as we normally cache created chains.
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			if (config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}
	return interceptorList;
}
```
>该方法执行后，Advised中配置能够应用到连接点或目标类的Advisor全部被转化成了MethodInterceptor。
- 得到拦截器链怎么起作用

```java 
// org.springframework.aop.framework.JdkDynamicAopProxy
// 若没可应用到此方法的通知(Interceptor)，直接反射调用method.invoke(target，args)
if (chain.isEmpty()) {
	// We can skip creating a MethodInvocation: just invoke the target directly
	// Note that the final invoker must be an InvokerInterceptor so we know it does
	// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
	retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
}
else {
	// We need to create a method invocation...
	// 创建MethodInvocation
	invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
	// Proceed to the joinpoint through the interceptor chain.
	retVal = invocation.proceed();
}
```
>若拦截器链为空，则直接反射调用目标方法，否则创建MethodInvocation，调用其proceed方法，触发拦截器链的执行。

```java 
// org.springframework.aop.framework.ReflectiveMethodInvocation
public Object proceed() throws Throwable {
    // 若Interceptor执行完了，则执行joinPoint
    if(this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return this.invokeJoinpoint();
    } else {
        // 若要动态匹配joinPoint
        Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        L
        if(interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
            return dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)?dm.interceptor.invoke(this):this.proceed();
        } else {
            // 动态匹配失败时，略过当前intercepor，调用下一个intercepttor
            return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
        }
    }
}
```
>AOP到底能干嘛？  
BOP编程，由Factory执行动作，IOC容器
AOP工厂生产出来的bean是可以放入IOC容器中的
看源码最重要的是入口-Factory。  
getProxy方法用来获取一个代理后的bean，跟IOC容器的getBean有异曲同工之妙。  
FactoryBean统一了标准，IOC容器定位、加载、注册、初始化、注入。  
而AOP的Factory依赖于IOC容器，先等IOC容器启动后，进行二次操作。  
AOP不用执行创建对象的操作（定位、加载、注册、初始化、注入），直接从IOC容器中取。  
AOP的bean没有继承BeanFactory。  
IOC已经是代理类了，那么AOP中，是对代理类的二次操作（深操作），监听每个动作都要被管控。
除了原型模式外，其余都是被代理的。  
AOP最核心的东西：代理。主要两种：JDK代理、Cglib。  
MethodInterceptor[] interceptors = registry.getInterceptors(advisor);  
由切点进入切面，切点实际上就是Method。通知的动作通知前、后置处理器。切点，转换为MethodInterceptor，保存到一个容器里，该容器是链表结构，一定是有顺序的，它知道上一个和下一个是谁。   
有了这种结构，AOP就可以应用自如。  
AOP为什么能用来做权限控制，得益于这种拦截器链表的设计。  
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, Class targetClass)   
传进去的config可能是xml，也可能是注解，最终都会转换为AopConfig，只是注解定位时读的是class。
链表结构操作的一种较经典的方式  
Filter  
Filter Chain  

>before->open开启事务  
throwing->rollback回滚  
after->commit提交事务  
>>1、加载配置信息，解析成AopConfig；  
2、交给AopProxyFactory，调用一个createAopProxy方法；  
JdkDynamicAopProxy调用AdvisedSupport的getInterceptorsAndDynamicInterceptionAdvice方法得到方法拦截器链，并保存到一个容器（List，IOC容器是Map），递归执行拦截器方法proceed方法。

>MethodInterceptor容器是List，IOC容器是一个ConcurrentHashMap。  
最终就是由一个Advisor来调用切面中的方法。  
Cglib调用链：getCallbacks、advised.getInterceptorsAndDynamicInterceptionAdvice
JDK代理和Cglib有何区别？AOP有自动判断，若实现interface默认用JDK，否则使用Cglib，最终目的都是扫描所有切点，形成拦截器链。  
JDK性能更低。