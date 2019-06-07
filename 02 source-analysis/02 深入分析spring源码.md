[toc]

### 1.spring IOC核心体系结构

#### 1.1 BeanFactory
>Spring Bean的创建是典型的工厂模式，这些Bean工厂（IOC容器）在管理对象间的依赖关系提供了便利和基础服务，Spring中有许多IOC容器的实现，相互关系如下：

![image](https://github.com/nanphonfy/note-images/blob/master/promote-2019/source-analysis/DefaultListableBeanFactory.png?raw=true)

>BeanFactory是最顶层的接口类，定义了IOC容器的基本功能规范。  
它有三个子类：ListableBeanFactory、HierarchicalBeanFactory和AutowireCapableBeanFactory。  
上图最终的默认实现类是DefaultListableBeanFactory，它实现了所有接口。  
>>为何定义这么多层次的接口？每个接口都有使用场合，为了区分Spring内对象的传递和转化过程中，对对象的数据访问所做的限制。  
eg.ListableBeanFactory接口：可列表，HierarchicalBeanFactory：有继承关系（每个Bean有可能有父Bean），AutowireCapableBeanFactory：自动装配规则。这四接口定义了Bean的集合、Bean之间的关系、以及Bean行为。

- 最基本的IOC容器接口BeanFactory
```java  
//org.springframework.beans.factory.BeanFactory
public interface BeanFactory {
	/**使用转义符&来得到FactoryBean本身，用来区分通过容器获得FactoryBean本身和其产生的对象**/
	String FACTORY_BEAN_PREFIX = "&";

	//通过name来获取IOC容器指定的bean
	Object getBean(String name) throws BeansException;

	//根据名字和类型来得到bean实例，增加了类型安全验证机制
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	//通过类型获取bean实例
	<T> T getBean(Class<T> requiredType) throws BeansException;

	//增加更多获取的条件，同上方法
	Object getBean(String name, Object... args) throws BeansException;

	//判断IOC容器是否含有指定名字的bean
	boolean containsBean(String name);

	//查询指定名字的Bean是不是单例的Bean
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	//判断Bean是不是prototype类型的bean
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	//查询指定了名字的Bean的Class类型是否与指定类型匹配
	boolean isTypeMatch(String name, Class<?> targetType) throws NoSuchBeanDefinitionException;

	//获取指定名字bean的Class类型
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	//查询指定了名字的bean的所有别名，这些别名都是在BeanDefinition中定义的
	String[] getAliases(String name);
}
```
>只定义了IOC容器的基本行为，不关心Bean如何定义、加载。（类比我们只关心工厂产品，不关心生产流程）  
工厂如何产生对象？看具体IOC容器实现——eg.XmlBeanFactory，ClasspathXmlApplicationContext等。  
XmlBeanFactory实现最基本的IOC容器（可读取XML文件定义的BeanDefinition（XML文件中对bean的描述））。

- 类比：XmlBeanFactory——容器中的屌丝，ApplicationContext——容器中的高帅富。  
>ApplicationContext一个高级IOC容器，除了提供IOC容器的基本功能外，还提供如下附加服务：  
>>1.支持信息源，可实现国际化（MessageSource）；   
2.访问资源(ResourcePatternResolver)；  
3.支持应用事件(ApplicationEventPublisher)。

#### 1.2 BeanDefinition
>Spring IOC容器管理各种Bean对象及其相互关系，Bean对象在Spring实现中是以BeanDefinition来描述的，其继承体系如下：

![image](https://github.com/nanphonfy/note-images/blob/master/promote-2019/source-analysis/RootBeanDefinition.jpg?raw=true)

>Bean解析过程非常复杂，功能很细。需被扩展的很多，必须保证足够的灵活性。Bean的解析主要是对Spring配置文件的解析。这个解析过程主要通过下图中的类完成：

![image](https://github.com/nanphonfy/note-images/blob/master/promote-2019/source-analysis/XmlBeanDefinitionReader.jpg?raw=true)

---

### 2.IOC容器初始化
>IOC容器的初始化过程包括BeanDefinition的Resource定位、载入和注册。以ApplicationContext为例讲解（web项目中使用的XmlWebApplicationContext、ClasspathXmlApplicationContext等就属于这个继承体系），如下：

![image](https://github.com/nanphonfy/note-images/blob/master/promote-2019/source-analysis/ApplicationContext.jpg?raw=true)

>ApplicationContext允许上下文嵌套，通过保持父上下文可维持一个上下文体系。Bean可在上下文体系中查找：①检查当前上下文；②父上下文，逐级向上（为不同的Spring应用提供共享的Bean环境）。

#### 2.1 XmlBeanFactory(屌丝IOC)流程  

```java 
public class XmlBeanFactory extends DefaultListableBeanFactory {
    //this传的是factory对象
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}

	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
	}
}
```

- 用代码，直观表达源码意图：定位、载入、注册的全过程
```java 
//根据xml文件，创建resource资源对象（包含BeanDefinition信息）
ClassPathResource resource = new ClassPathResource("applicantion-context.xml");
//创造DefaultListableBeanFactory
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// 创建读取器，用于载入BeanDefinition
// 需要BeanDefinition做为参数——会将读取的信息回调配置给factory
// XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this)中的this传的是factory
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
// 执行载入的BeanDefinition方法，最后完成bean的载入和注册
// 完成后bean成功放置到IOC容器中
reader.loadBeanDefinitions(resource);
```
#### 2.2 FileSystemXmlApplicationContext的IOC容器流程   

##### 2.2.1 高富帅版IOC解剖
```java 
public FileSystemXmlApplicationContext(String... configLocations) throws BeansException {
	this(configLocations, true, null);
}
//实际调用构造函数
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
		throws BeansException {

	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
		refresh();
	}
}
```
##### 2.2.2 设置资源加载器和资源定位
>创建FileSystemXmlApplicationContext容器时，构造方法做两项重要工作：  
>>①调用父类容器的构造方法(super(parent)方法)为容器设置好Bean资源加载器； 
②调用父类AbstractRefreshableConfigApplicationContext的setConfigLocations(configLocations)方法设置Bean定义资源文件的定位路径。  

>通过追踪FileSystemXmlApplicationContext的继承体系，发现其父类的父类AbstractApplicationContext中初始化IOC容器所做的主要源码如下：

```java 
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
		//静态初始化块，在整个容器创建过程中只执行一次
	static {
		// Eagerly load the ContextClosedEvent class to avoid weird classloader issues
		// on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
		//为了避免应用程序在Weblogic8.1关闭时出现类加载异常加载问题，加载IOC容器关闭事件(ContextClosedEvent)类
		ContextClosedEvent.class.getName();
	}
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}

	//FileSystemXmlApplicationContext调用父类构造方法调用的就是该方法
	public AbstractApplicationContext(ApplicationContext parent) {
		this();
		setParent(parent);
	}
	//获取一个Spring Source的加载器用于读入Spring Bean定义资源文件
	protected ResourcePatternResolver getResourcePatternResolver() {
		//AbstractApplicationContext继承DefaultResourceLoader，因此也是一个资源加载器
        //Spring资源加载器，其getResource(String location)方法用于载入资源
		return new PathMatchingResourcePatternResolver(this);
	}
}
```
AbstractApplicationContext构造方法中调用PathMatchingResourcePatternResolver的构造方法创建Spring资源加载器

```java 
public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
	Assert.notNull(resourceLoader, "ResourceLoader must not be null");
	//设置Spring的资源加载器
	this.resourceLoader = resourceLoader;
}
```
>设置容器的资源加载器后，执行setConfigLocations方法
——调用其父类AbstractRefreshableConfigApplicationContext的方法对Bean定义资源文件的定位，源码如下：

```java 
//解析Bean定义资源文件的路径，处理多个资源文件字符串数组
public void setConfigLocations(String[] locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
			//resolvePath为同一个类中将字符串解析为路径的方法  
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}
```
>既可使用一个字符串来配置多个SpringBean定义资源文件，也可使用字符串数组：
>- ClasspathResource res=new ClasspathResource(“a.xml,b.xml,......”);多个资源文件路径之间可以是用”,;\t\n”等分隔。  
>- ClasspathResource res=new ClasspathResource(newString[]{“a.xml”,”b.xml”,......});  

>至此，SpringIOC容器在初始化时将配置的Bean定义资源文件定位为Spring封装的Resource。

##### 2.2.3 AbstractApplicationContext的refresh函数载入Bean定义过程
>refresh()是一个模板方法，作用是：创建IOC容器前，若容器已存在，则把已有容器销毁和关闭（保证refresh后IOC容器是新建的，类似对IOC容器重启）。在新建立的容器进行初始化，对Bean定义资源进行载入。  

- FileSystemXmlApplicationContext通过调用其父类AbstractApplicationContext的refresh()函数启动整个IOC容器对Bean定义的载入过程：
```java 
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
		}catch (BeansException ex) {
			// Destroy already created singletons to avoid dangling resources.
			// 销毁已创建的单态Bean
			destroyBeans();

			//Reset 'active' flag.
			//取消refresh操作，重置容器的同步标识.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}
	}
}
```
>refresh()方法为IOC容器Bean的生命周期管理提供条件。  
SpringIOC容器载入Bean定义资源文件从其子类容器的refreshBeanFactory()方法启动，所以整个refresh()中“ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();”这句以后的代码都是注册容器的信息源和生命周期事件，载入过程从该句开始。

##### 2.2.4 AbstractApplicationContext的obtainFreshBeanFactory()
- AbstractApplicationContext的obtainFreshBeanFactory()方法调用子类容器的refreshBeanFactory()方法，启动容器载入Bean定义资源文件的过程
```java 
//org.springframework.context.support.AbstractApplicationContext
//protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	//这里使用委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
	refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}

//org.springframework.context.support.AbstractRefreshableApplicationContext
//实现抽象类接口
protected final void refreshBeanFactory() throws BeansException {
    //如果已经有容器，销毁容器中的bean，关闭容器
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		//创建IOC容器
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		//对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
		customizeBeanFactory(beanFactory);
		//调用载入Bean定义的方法，这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}

//又使用了委托模式，调用子类获取bean定义资源定位的方法（在ClassPathXmlApplicationContext中实现，FileSystemXmlApplicationContext没使用该方法）
protected Resource[] getConfigResources() {
	return null;
}
```
>先判断BeanFactory是否存在，若存在则先销毁beans并关闭beanFactory，接着创建DefaultListableBeanFactory，并调用loadBeanDefinitions(beanFactory)装载bean定义。

##### 2.2.5 AbstractRefreshableApplicationContext子类的loadBeanDefinitions方法
>AbstractRefreshableApplicationContext中只定义了抽象的loadBeanDefinitions方法，容器真正调用的是其子类AbstractXmlApplicationContext对该方法的实现：
loadBeanDefinitions方法同样是抽象方法，是由其子类实现的，也即在AbstractXmlApplicationContext中。

```java 
//org.springframework.context.support.AbstractRefreshableApplicationContext
//protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory)
			throws BeansException, IOException;
//实现父类抽象的载入Bean定义方法 
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
	//创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容器使用该读取器读取Bean定义资源
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	//为Bean读取器设置Spring资源加载器，AbstractXmlApplicationContext的  
    //祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器
	beanDefinitionReader.setResourceLoader(this);
	//为Bean读取器设置SAX xml解析器
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	//当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制
	initBeanDefinitionReader(beanDefinitionReader);
	//Bean读取器真正实现加载的方法
	loadBeanDefinitions(beanDefinitionReader);
}

protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
	reader.setValidating(this.validating);
}

//Xml Bean读取器加载Bean定义资源
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	//获取Bean定义资源的定位
	//这里使用了一个委托模式，调用子类的获取Bean定义资源定位的方法  
    //该方法在ClassPathXmlApplicationContext中进行实现，对于我们  
    //举例分析源码的FileSystemXmlApplicationContext没有使用该方法
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		//Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的Bean定义资源
		reader.loadBeanDefinitions(configResources);
	}
	//如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		//Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的Bean定义资源
		reader.loadBeanDefinitions(configLocations);
	}
}

protected Resource[] getConfigResources() {
	return null;
}
```

##### 2.2.6 AbstractBeanDefinitionReader的loadBeanDefinitions方法
>AbstractBeanDefinitionReader读取Bean定义资源,在其抽象父类AbstractBeanDefinitionReader中定义了载入过程。
>loadBeanDefinitions方法做了两件事：①调用资源加载器的获取资源方法resourceLoader.getResource(location)，获取加载资源；②真正执行加载功能是其子类XmlBeanDefinitionReader的loadBeanDefinitions方法。


```java 
//org.springframework.beans.factory.support.AbstractBeanDefinitionReader
//重载方法，调用下面的loadBeanDefinitions(String, Set<Resource>);方法 
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}

public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
	//获取在IoC容器初始化过程中设置的资源加载器 
	ResourceLoader resourceLoader = getResourceLoader();
	if (resourceLoader == null) {
		throw new BeanDefinitionStoreException("Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
	}

	if (resourceLoader instanceof ResourcePatternResolver) {
		// Resource pattern matching available.
		try {
			//将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源  
            //加载多个指定位置的Bean定义资源文件  
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
			//委派调用其子类XmlBeanDefinitionReader、PropertiesBeanDefinitionReader的方法，实现加载功能 
			int loadCount = loadBeanDefinitions(resources);
			if (actualResources != null) {
				for (Resource resource : resources) {
					actualResources.add(resource);
				}
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
			}
			return loadCount;
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("Could not resolve bean definition resource pattern [" + location + "]", ex);
		}
	}
	else {
		// Can only load single resources by absolute URL.
		//将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源  
        //加载单个指定位置的Bean定义资源文件
		Resource resource = resourceLoader.getResource(location);
		//委派调用其子类XmlBeanDefinitionReader、PropertiesBeanDefinitionReader的方法，实现加载功能
		int loadCount = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
		}
		return loadCount;
	}
}

//重载方法，调用loadBeanDefinitions(String)
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int counter = 0;
	for (String location : locations) {
		counter += loadBeanDefinitions(location);
	}
	return counter;
}
```

>ResourceLoader与ApplicationContext继承系图，可知道其实际调用的是DefaultResourceLoader中的getSource()方法定位Resource，因为FileSystemXmlApplicationContext本身就是DefaultResourceLoader的实现类，故又回到了FileSystemXmlApplicationContext中来。

##### 2.2.7 DefaultResourceLoader的getResource方法
>资源加载器获取要读入的资源：XmlBeanDefinitionReader调用父类DefaultResourceLoader方法。

```java 
//org.springframework.core.io.DefaultResourceLoader
//获取Resource的具体实现方法
public Resource getResource(String location) {
	Assert.notNull(location, "Location must not be null");
	//如果是类路径的方式，那需要使用ClassPathResource来得到bean 文件的资源对象
	if (location.startsWith(CLASSPATH_URL_PREFIX)) {
		return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
	}
	else {
		try {
			// Try to parse the location as a URL...
			// 如果是URL 方式，使用UrlResource 作为bean 文件的资源对象
			URL url = new URL(location);
			return new UrlResource(url);
		}
		catch (MalformedURLException ex) {
			// No URL -> resolve as resource path.
			//如果既不是classpath标识，又不是URL标识的Resource定位，则调用容器本身的getResourceByPath方法获取Resource 
			return getResourceByPath(location);
		}
	}
}

//org.springframework.context.support.FileSystemXmlApplicationContext
@Override
protected Resource getResourceByPath(String path) {
	if (path != null && path.startsWith("/")) {
		path = path.substring(1);
	}
	return new FileSystemResource(path);
}
```
>DefaultResourceLoader的getResourceByPath没有具体实现，是继承父类的实现（super），FileSystemXmlApplicationContext容器提供了getResourceByPath的实现（处理既不是classpath标识，又不是URL标识），其中FileSystemResource负责从文件系统得到配置文件的资源定义，加载IOC配置文件（也可从其他地方加载）。
>>在Spring中提供了各种资源抽象，eg.ClassPathResource,URLResource,FileSystemResource等。上面我们看到的是定位Resource的一个过程，而这只是加载过程的一部分。

#### 2.2.8 XmlBeanDefinitionReader的loadBeanDefinitions
>XmlBeanDefinitionReader加载Bean定义资源,源码代表bean文件的资源定义以后的载入过程。


```java 
//org.springframework.beans.factory.xml.XmlBeanDefinitionReader
//XmlBeanDefinitionReader加载资源的入口方法
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	//将读入的XML资源进行特殊编码处理
	return loadBeanDefinitions(new EncodedResource(resource));
}

//这里是载入XML形式Bean定义资源文件方法
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    ...
    try {
    	//将资源文件转为InputStream的IO流
    	InputStream inputStream = encodedResource.getResource().getInputStream();
    	try {
    		//从InputStream中得到XML的解析源
    		InputSource inputSource = new InputSource(inputStream);
    		if (encodedResource.getEncoding() != null) {
    			inputSource.setEncoding(encodedResource.getEncoding());
    		}
    		//这里是具体的读取过程
    		return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
    	}
    	finally {
    		//关闭从Resource中得到的IO流
    		inputStream.close();
    	}
    }
    ...
}

//从特定XML文件中实际载入Bean定义资源的方法
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
		throws BeanDefinitionStoreException {
	try {
		int validationMode = getValidationModeForResource(resource);
		//将XML文件转换为DOM对象，解析过程由documentLoader实现
		Document doc = this.documentLoader.loadDocument(
				inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
		//这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
		return registerBeanDefinitions(doc, resource);
	}
	...
}
```
>载入Bean定义资源文件的最后一步是将Bean定义资源转换为Document对象，该过程由documentLoader实现。

#### 2.2.9 DocumentLoader的loadDocument方法
>DocumentLoader将Bean定义资源转换成Document对象源码：
```java 
//org.springframework.beans.factory.xml.DocumentLoader定义了接口
//org.springframework.beans.factory.xml.DefaultDocumentLoader实现了接口，如下
//使用标准的JAXP将载入的Bean定义资源转换成document对象
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
		ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
	//创建文件解析器工厂
	DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
	if (logger.isDebugEnabled()) {
		logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
	}
	//创建文档解析器
	DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
	//解析Spring的Bean定义资源
	return builder.parse(inputSource);
}	

//创建文档解析工厂
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
		throws ParserConfigurationException {
	DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
	factory.setNamespaceAware(namespaceAware);

	//设置解析XML的校验
	if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
		factory.setValidating(true);
		if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
			// Enforce namespace aware for XSD...
			factory.setNamespaceAware(true);
			try {
				factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
			}
			catch (IllegalArgumentException ex) {
				ParserConfigurationException pcex = new ParserConfigurationException(
						"Unable to validate using XSD: Your JAXP provider [" + factory +
						"] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
						"Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
				pcex.initCause(ex);
				throw pcex;
			}
		}
	}
	return factory;
}
```
>至此SpringIOC容器根据定位的Bean定义资源文件，将其加载读入并转换成为Document对象过程完成。接下来继续分析SpringIOC容器将载入的Bean定义资源文件转换为Document对象后，如何将其解析为SpringIOC管理的Bean对象并将其注册到容器中。

#### 2.2.10 XmlBeanDefinitionReader的registerBeanDefinitions方法
>XmlBeanDefinitionReader类中的doLoadBeanDefinitions方法是从特定XML文件中实际载入Bean定义资源的方法，该方法在载入Bean定义资源后将其转换为Document对象，接下来调用registerBeanDefinitions启动SpringIOC容器对Bean定义的解析过程。
```java 
//按照Spring的Bean语义要求将Bean定义资源解析并转换为容器内部数据结构
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	//得到BeanDefinitionDocumentReader来对xml格式的BeanDefinition解析
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	documentReader.setEnvironment(this.getEnvironment());
	//获得容器中注册的Bean数量
	int countBefore = getRegistry().getBeanDefinitionCount();
	//解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader只是个接口，具体的解析实现过程有实现类DefaultBeanDefinitionDocumentReader完成
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	//统计解析的Bean数量
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
>Bean定义资源的载入解析的两个过程：  
①通过调用XML解析器将Bean定义资源文件转换得到Document对象（没按照Spring的Bean规则解析，载入过程）②完成通用XML解析后，按照Spring的Bean规则对Document对象进行解析。  
按照Spring的Bean规则对Document对象解析的过程是在接口BeanDefinitionDocumentReader的实现类DefaultBeanDefinitionDocumentReader中实现的。

---

看源码，找入口非常关键。
IOC/DI/AOP/BOP，要和设计模式联系上。

eg.ClassPathXmlApplicationContext最顶层的入口是BeanFactory。

BeanFactory做为最顶级接口类，定义IOC容器基本功能规范，其有三个子类：.....
其最终实现类：DefaltListableBeanFactory，实现了所有接口。
eg.ListableBeanFactory：这些bean是可列表的；
HierarchialBeanFactory：这些bean有继承关系；
AutowireCapableBeanFactory：bean的自动装配规则。

这四个接口共同定义Bean的集合、bean之间的关系以及bean的行为。

Spring BeanFactory源码学习
http://www.cnblogs.com/wxgblogs/p/6659417.html

```xml
<!--表象，只是我们用户所关心的产品-->
<bean class="">
	<property name="">
		<list>
			
		</list>
		<map>
			<entry>
				<key></key>
				<value></value>
			</entry>
		</map>
		<set>
			<bean></bean>
		</set>
	</property>
</bean>
```

#### spring-beans

- 

工厂如何产生对象，需看IOC容器实现（eg.XmlBeanFactory[最基本的IOC容器实现]、ClasspathXmlApplicationContext[高级IOC容器]）

ApplicationContext接口看特点：
①支持信息源，可国际化（MessageSource接口）；
②访问资源，（实现ResourcePatternResolver接口）；
③支持应用事件（实现ApplicationEventPublisher接口）。

BeanDefinition
IOC容器管理各种Bean对象和其相互关系，Bean对象在spring实现中以BeanDefinition来描述。
org.springframework.beans.factory.config.BeanDefinition


bean的解析过程非常复杂，功能被分的很细，Bean的解析过程主要是对spring配置文件的解析，主要通过以下完成：

org.springframework.beans.factory.xml.XmlBeanDefinitionReader

Spring源码浅析之BeanDefinition
https://blog.csdn.net/ljw761123096/article/details/80353202

汽车倒车入库技巧图解  
https://jingyan.baidu.com/article/3f16e003bc653c2590c1035d.html



