### 1.基于Annotation的依赖注入
>注解(Annotation)是JDK1.5引入的新特性，简化Bean的配置，某些场合可取代XML。注解可大大简化配置，提高开发速度，不能完全取代XML(更加灵活，发展相对成熟,为大多数开发者熟悉)；注解方式非常简洁，处于发展阶段，XML和注解可以相互配合。  

> IOC容器对于类级别的注解和类内部的注解分两种处理策略：
>- 类级别的注解：eg.@Component、@Repository、@Controller、@Service及JavaEE6的@ManagedBean和@Named注解，Spring容器根据注解的过滤规则扫描读取注解Bean定义类，并将其注册到IOC容器；
>- 类内部的注解：eg.@Autowire、@Value、@Resource及EJB和WebService相关注解等（添加在类内部的字段或方法的类内部注解），IOC容器通过Bean后置注解处理器解析Bean内部的注解。

### 2.AnnotationConfigApplicationContext对注解Bean初始化
>Spring中管理注解Bean定义的容器有两个：AnnotationConfigApplicationContext和AnnotationConfigWebApplicationContex。  
这两个类是专门处理Spring注解方式配置的容器，直接依赖于注解作为容器配置信息来源的IOC容器。AnnotationConfigWebApplicationContext是AnnotationConfigApplicationContext的web版本，两者用法及对注解的处理方式没有啥差别。  
- AnnotationConfigApplicationContext的源码

```java 
//org.springframework.context.annotation.AnnotationConfigApplicationContext
public class AnnotationConfigApplicationContext extends GenericApplicationContext {
    // 保存一个读取注解的Bean定义读取器，将其设置到容器中
    private final AnnotatedBeanDefinitionReader reader;
    // 保存一个扫描指定类路径中注解Bean定义读取器，将其设置到容器中
    private final ClassPathBeanDefinitionScanner scanner;
    
    
    // 默认构造器，初始化空容器（不包含任何bean信息，调用其register注册配置类，调用refresh刷新容器，触发容器对注解ben载入、解析、注册）
    public AnnotationConfigApplicationContext() {
    	this.reader = new AnnotatedBeanDefinitionReader(this);
    	this.scanner = new ClassPathBeanDefinitionScanner(this);
    }
    
    // 最常用构造器，将涉及配置类传递给该构造函数，实现将相应配置类中的bean自动注册到容器中
    public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    	this();
    	register(annotatedClasses);
    	refresh();
    }
    
    // 该构造器自动扫描给定包及其子包所有类，自动识别所有bean将其注册到容器中
    public AnnotationConfigApplicationContext(String... basePackages) {
    	this();
    	scan(basePackages);
    	refresh();
    }

    @Override
    public void setEnvironment(ConfigurableEnvironment environment) {
    	super.setEnvironment(environment);
    	this.reader.setEnvironment(environment);
    	this.scanner.setEnvironment(environment);
    }
    
    // 为容器的注解bean读取器和注解bean扫描器设置bean名称产生器
    public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
    	this.reader.setBeanNameGenerator(beanNameGenerator);
    	this.scanner.setBeanNameGenerator(beanNameGenerator);
    	getBeanFactory().registerSingleton(
    			AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
    }
    
    // 为容器注解bean读取器和注解bean扫描器设置作用范围元信息解析器
    public void setScopeMetadataResolver(ScopeMetadataResolver scopeMetadataResolver) {
    	this.reader.setScopeMetadataResolver(scopeMetadataResolver);
    	this.scanner.setScopeMetadataResolver(scopeMetadataResolver);
    }
    
    // 为容器注册一个要被处理的注解bean，新注册的bean必须手动调用容器refresh刷新容器，触发容器对新注册的bean处理
    public void register(Class<?>... annotatedClasses) {
    	Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
    	this.reader.register(annotatedClasses);
    }
    
    //扫描指定包路径及子包下的注解类，手动调用refresh刷新容器
    public void scan(String... basePackages) {
    	Assert.notEmpty(basePackages, "At least one base package must be specified");
    	this.scanner.scan(basePackages);
    }
    
    @Override
    protected void prepareRefresh() {
    	this.scanner.clearCache();
    	super.prepareRefresh();
    }
}
```
>Spring对注解的处理分为两种方式：
>- 直接将注解Bean注册到容器：  
可以在初始化容器时注册；也可在容器创建后手动调用注册方法向容器注册，然后手动刷新容器，使容器对注册的注解Bean进行处理；
>- 通过扫描指定包及其子包下的所有类： 
在初始化注解容器时指定要自动扫描的路径，若容器创建后向给定路径动态添加了注解Bean，则需手动调用容器扫描的方法，然后手动刷新容器，使得容器对所注册的Bean进行处理。  
接下来，将会对两种处理方式详细分析其实现过程。
### 3. AnnotationConfigApplicationContext注册注解Bean
>创建注解处理容器时，若传入的初始参数是具体的注解Bean定义类时，注解容器读取并注册。  

- AnnotationConfigApplicationContext通过调用注解Bean定义读取器AnnotatedBeanDefinitionReader的register方法向容器注册指定的注解Bean，注解Bean定义读取器向容器注册注解Bean的源码如下：
```java 
public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
	// 根据指定的注解bean定义类，创建容器对注解bean封装的数据结构
	AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
	AnnotationMetadata metadata = abd.getMetadata();
	if (metadata.isAnnotated(Profile.class.getName())) {
		AnnotationAttributes profile = MetadataUtils.attributesFor(metadata, Profile.class);
		if (!this.environment.acceptsProfiles(profile.getStringArray("value"))) {
			return;
		}
	}
	// 解析注解bean定义的作用域，若@scope（"prototype"），则bean为原型类型，若@scope（"singleton"），则bean为单态类型
	ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
	// 为注解bean定义设置作用
	abd.setScope(scopeMetadata.getScopeName());
	// 为注解bean定义生成bean名称
	String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
	// 处理注解bean定义的通用注解
	AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
	// 若向容器注册注解bean定义时，使用了额外的限定符注解，则解析限定符注解。配置：autowiring自动依赖注入装配的限定条件，即@qualifier注解，spring自动依赖注入装配默认按类型装配，@qualifier按名称
	if (qualifiers != null) {
		for (Class<? extends Annotation> qualifier : qualifiers) {
			// 若配置了@primary注解，设置该bean为autowiring自动依赖注入装配时的首选
			if (Primary.class.equals(qualifier)) {
				abd.setPrimary(true);
			}
			// 若配置了@lazy注解，则设置该bean为非延迟初始化，若没配置，则该bean为预实例化
			else if (Lazy.class.equals(qualifier)) {
				abd.setLazyInit(true);
			}
			else {
				// 若使用了除@primary和@lazy外的其他注解，则为该bean添加一个autowiring自动依赖注入装配限定符，该bean在进autowiring自动依赖注入装配时，根据名称装配限定符指定的bean
				abd.addQualifier(new AutowireCandidateQualifier(qualifier));
			}
		}
	}
	// 创建一个指定bean名称的bean定义对象，封装注解bean定义类数据
	BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
	// 根据注解bean定义类中配置的作用域，创建相应的代理对象
	definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
	// 向IOC容器注册注解bean类定义对象
	BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```
>注册注解Bean定义类的基本步骤：  
①需用注解元数据解析器解析注解Bean中关于作用域的配置；  
②用AnnotationConfigUtils的processCommonDefinitionAnnotations方法处理注解Bean定义类中通用的注解；  
③用AnnotationConfigUtils的applyScopedProxyMode方法创建对于作用域的代理对象；  
④通过BeanDefinitionReaderUtils向容器注册Bean。

- AnnotationScopeMetadataResolver解析作用域元数据  

```java 
// 解析注解bean定义类中的作用域元信息
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
	ScopeMetadata metadata = new ScopeMetadata();
	if (definition instanceof AnnotatedBeanDefinition) {
		AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
		// 从注解bean定义类的属性中查找属性scope的值，即@scope注解的值annDef.getMetadata().getAnnotationAttributes将bean所有注解和注解的值存放在一个map集合中
		AnnotationAttributes attributes = attributesFor(annDef.getMetadata(), this.scopeAnnotationType);
		// 将获取到的@scope注解的值设置到要返回的对象中// 将获取到的@scope注解的值设置到要返回的对象中
		if (attributes != null) {
			metadata.setScopeName(attributes.getString("value"));
			// 获取@scope注解中的proxymode属性值，创建代理对象时会用
			ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
			// 若@scope的proxyMode属性为DEFAULT或NO
			if (proxyMode == null || proxyMode == ScopedProxyMode.DEFAULT) {
				// 设置proxyMode为NO
				proxyMode = this.defaultProxyMode;
			}
			// 为返回的元数据设置proxyMode
			metadata.setScopedProxyMode(proxyMode);
		}
	}
	// 返回解析的作用域元信息对象
	return metadata;
}
```
>annDef.getMetadata().getAnnotationAttributes方法就是获取对象中指定类型的注解值。

- AnnotationConfigUtils处理注解Bean定义类中的通用注解  
>AnnotationConfigUtils类的processCommonDefinitionAnnotations在向容器注册Bean前，先对注解Bean定义类中的通用Spring注解进行处理


```java 
// org.springframework.context.annotation.AnnotationConfigUtils
// 处理bean定义中通用注解
public static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd) {
	AnnotationMetadata metadata = abd.getMetadata();
	// 若bean定义中有@primary注解，则为该bean设置为autowiring自动依赖注入装配的首选对象
	if (metadata.isAnnotated(Primary.class.getName())) {
		abd.setPrimary(true);
	}
	// 若有@lazy注解，则将该bean预实例化属性设置为@lazy注解值
	if (metadata.isAnnotated(Lazy.class.getName())) {
		abd.setLazyInit(attributesFor(metadata, Lazy.class).getBoolean("value"));
	}
	// 若bean定义中有@dependson注解，则为该bean设置所依赖的bean名称，容器将确保在实例化该bean前首选实例化所依赖的bean
	if (metadata.isAnnotated(DependsOn.class.getName())) {
		abd.setDependsOn(attributesFor(metadata, DependsOn.class).getStringArray("value"));
	}
	if (abd instanceof AbstractBeanDefinition) {
		if (metadata.isAnnotated(Role.class.getName())) {
			Integer role = attributesFor(metadata, Role.class).getNumber("value");
			((AbstractBeanDefinition)abd).setRole(role);
		}
	}
}
```
- AnnotationConfigUtils根据注解Bean定义类中配置的作用域为其应用相应的代理策略
>AnnotationConfigUtils类的applyScopedProxyMode方法根据注解Bean定义类中配置的作用域@Scope注解的值，为Bean定义应用相应的代理模式，主要在Spring面向切面编程(AOP)中使用。

```java 
// org.springframework.context.annotation.AnnotationConfigUtils
// 根据作用域为bean应用引用的代理模式
static BeanDefinitionHolder applyScopedProxyMode(ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
	// 获取注解bean定义类中@scope注解的proxyMode属性值
	ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
	// 若配置的@scope注解的proxyMode属性值为NO，则不应用代理模式
	if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
		return definition;
	}
	// 获取配置的@scope注解的proxyMode属性值，若为TARGET_CLASS则返回true，若为INTERFACES则返回false
	boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
	// 为注册的bean创建相应模式的代理对象
	return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
```
- BeanDefinitionReaderUtils向容器注册Bean
BeanDefinitionReaderUtils向容器注册载入的Bean(主要校验Bean定义，将Bean添加到容器中一个管理Bean定义的HashMap中),前面已分析。

### 4. AnnotationConfigApplicationContext扫描指定包及其子包下的注解Bean
>当创建注解处理容器时，若传入的初始参数是注解Bean定义类所在的包时，注解容器将扫描给定的包及其子包，将扫描到的注解Bean定义载入并注册。
- ClassPathBeanDefinitionScanner扫描给定的包及其子包
>AnnotationConfigApplicationContext通过调用类路径Bean定义扫描器ClassPathBeanDefinitionScanner扫描给定包及其子包下的所有类。

```java 
// org.springframework.context.annotation.ClassPathBeanDefinitionScanner
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
    ......
	// 创建一个类路径bean定义扫描器
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
		this(registry, true);
	}

	// 为容器创建一个类路径bean定义扫描器，并制定是否使用默认的扫描过滤规则。即spring默认扫描配置：@component、@repository、@service、@controller注解的bean，同时也支持javaEE6的@managedbean和JSR-330的@named注解
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
		this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
	}
	
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters, Environment environment) {
		super(useDefaultFilters, environment);

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		// 为容器设置加载bean定义的注册器
		this.registry = registry;

		// Determine ResourceLoader to use.
		if (this.registry instanceof ResourceLoader) {
			// 为容器设置资源加载器
			setResourceLoader((ResourceLoader) this.registry);
		}
	}

	public final BeanDefinitionRegistry getRegistry() {
		return this.registry;
	}

	public void setBeanDefinitionDefaults(BeanDefinitionDefaults beanDefinitionDefaults) {
		this.beanDefinitionDefaults =
				(beanDefinitionDefaults != null ? beanDefinitionDefaults : new BeanDefinitionDefaults());
	}

	public void setAutowireCandidatePatterns(String[] autowireCandidatePatterns) {
		this.autowireCandidatePatterns = autowireCandidatePatterns;
	}

	public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
		this.beanNameGenerator = (beanNameGenerator != null ? beanNameGenerator : new AnnotationBeanNameGenerator());
	}

	public void setScopeMetadataResolver(ScopeMetadataResolver scopeMetadataResolver) {
		this.scopeMetadataResolver = (scopeMetadataResolver != null ? scopeMetadataResolver : new AnnotationScopeMetadataResolver());
	}

	public void setScopedProxyMode(ScopedProxyMode scopedProxyMode) {
		this.scopeMetadataResolver = new AnnotationScopeMetadataResolver(scopedProxyMode);
	}

	public void setIncludeAnnotationConfig(boolean includeAnnotationConfig) {
		this.includeAnnotationConfig = includeAnnotationConfig;
	}

	// 调用类路径bean定义扫描器入口方法
	public int scan(String... basePackages) {
		// 获取容器中已注册的bean个数
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		// 启动扫描器扫描给定包
		doScan(basePackages);

		// Register annotation config processors, if necessary.
		// 注册注解配置处理器
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		// 返回注册的bean个数
		return this.registry.getBeanDefinitionCount() - beanCountAtScanStart;
	}

	// 类路径bean定义扫描器扫描给定包及其子包
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		// 创建一个集合，存放扫描到bean定义的封装类
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		// 遍历扫描所有给定包
		for (String basePackage : basePackages) {
			// 调用父类ClassPathScanningCandidateComponentProvider的方法，扫描给定类路径，获取符合条件的bean定义
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			// 遍历扫描到的bean
			for (BeanDefinition candidate : candidates) {
				// 获取bean定义类中@scope注解的值，即获取bean的作用域
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				// 为bean设置注解配置的作用域
				candidate.setScope(scopeMetadata.getScopeName());
				// 为bean生成名称
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				// 若扫描到的bean不是spring注解的bean，则为bean设置默认值，设置bean的自动依赖注入装配属性等
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				// 若扫描到的bean是spring注解的bean，则处理其通用的spring注解
				if (candidate instanceof AnnotatedBeanDefinition) {
					// 处理注解bean中通用的注解，在分析注解bean定义类读取器时已经分析过
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				// 根据bean名称检查指定bean是否需要在容器中注册或在容器中冲突
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					// 根据注解中配置的作用域，为bean应用相应的代理模式
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					// 向容器注册扫描到的bean
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		// 返回注册的bean个数
		return beanDefinitions;
	}
}
```
>类路径Bean定义扫描器ClassPathBeanDefinitionScanner主要通过findCandidateComponents方法调用其父类ClassPathScanningCandidateComponentProvider类来扫描获取给定包及其子包下的类。

- ClassPathScanningCandidateComponentProvider扫描给定包及其子包的类

```java 
// org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider
public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
	// 保存过滤规则要包含的注解，即spring默认的@Component、@Repository、@Controller、@Service注解的bean，即javaEE6的@managedbean和@named注解
	private final List<TypeFilter> includeFilters = new LinkedList<TypeFilter>();

	// 保存过滤规则要排除的注解
	private final List<TypeFilter> excludeFilters = new LinkedList<TypeFilter>();

	// 构造方法，该方法在子类ClassPathBeanDefinitionScanner的构造方法中被调用
	public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
		this(useDefaultFilters, new StandardEnvironment());
	}

	public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters, Environment environment) {
		// 若使用spring默认的过滤规则，则向容器注册过滤规则
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		this.environment = environment;
	}

	// 向容器注册过滤规则
	protected void registerDefaultFilters() {
		// 向要包含的过滤规则中添加@component注解类（@Repository、@Controller、@Service都是component）
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		// 获取当前类的类加载器
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			// 向要包含的过滤规则添加javaEE6的@managedbean注解
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) cl.loadClass("javax.annotation.ManagedBean")), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			// 向要包含的过滤规则添加@named注解
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) cl.loadClass("javax.inject.Named")), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}

	// 扫描给定类路径的包
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		// 创建存储扫描到类的集合
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + "/" + this.resourcePattern;
			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						// 为指定资源获取元数据读取器，元信息读取器通过汇编(ASM)读取资源元信息
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
						// 若扫描到的类符合容器配置 的过滤规则
						if (isCandidateComponent(metadataReader)) {
							// 通过汇编(ASM)读取资源字节码中的bean定义元信息
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

	// 判断元信息读取器读取的类是否符合容器定义的注解过滤规则
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		// 若读取的类的注解在排除注解过滤规则中，则返回false
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				return false;
			}
		}
		// 若读取的类的注解在包含的注解过滤规则中，则返回true
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();
				if (!metadata.isAnnotated(Profile.class.getName())) {
					return true;
				}
				AnnotationAttributes profile = MetadataUtils.attributesFor(metadata, Profile.class);
				return this.environment.acceptsProfiles(profile.getStringArray("value"));
			}
		}
		// 若读取的类的注解既不在排除注解过滤规则，也不在包含的注解过滤规则中，则返回false
		return false;
	}
    ......
}
```
### 5.AnnotationConfigWebApplicationContext载入注解Bean定义
>AnnotationConfigWebApplicationContext是AnnotationConfigApplicationContext的Web版，它们对于注解Bean的注册和扫描基本相同，但AnnotationConfigWebApplicationContext对注解Bean定义的载入稍有不同，AnnotationConfigWebApplicationContext注入注解Bean定义源码:

```java 
// org.springframework.web.context.support.AnnotationConfigWebApplicationContext
// 载入注解bean定义资源
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
	// 为容器设置注解bean定义读取器
	AnnotatedBeanDefinitionReader reader = new AnnotatedBeanDefinitionReader(beanFactory);
	reader.setEnvironment(this.getEnvironment());

	// 为容器设置类路径bean定义扫描器
	ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(beanFactory);
	scanner.setEnvironment(this.getEnvironment());

	// 获取容器bean名称生成器
	BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
	// 获取容器的作用域元信息解析器
	ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
	// 为注解bean定义的读取器和类路径扫描器设置bean名称生成器
	if (beanNameGenerator != null) {
		reader.setBeanNameGenerator(beanNameGenerator);
		scanner.setBeanNameGenerator(beanNameGenerator);
		beanFactory.registerSingleton(
				AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
	}
	// 为注解bean定义读取器和类路径扫描器设置作用域元信息解析器
	if (scopeMetadataResolver != null) {
		reader.setScopeMetadataResolver(scopeMetadataResolver);
		scanner.setScopeMetadataResolver(scopeMetadataResolver);
	}

	if (!this.annotatedClasses.isEmpty()) {
		if (logger.isInfoEnabled()) {
			logger.info("Registering annotated classes: [" +
					StringUtils.collectionToCommaDelimitedString(this.annotatedClasses) + "]");
		}
		reader.register(this.annotatedClasses.toArray(new Class<?>[this.annotatedClasses.size()]));
	}

	if (!this.basePackages.isEmpty()) {
		if (logger.isInfoEnabled()) {
			logger.info("Scanning base packages: [" +
					StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
		}
		scanner.scan(this.basePackages.toArray(new String[this.basePackages.size()]));
	}

	// 获取容器定义的bean定义资源路径
	String[] configLocations = getConfigLocations();
	// 若定位bean定义资源路径不为空
	if (configLocations != null) {
		for (String configLocation : configLocations) {
			try {
				// 使用当前容器的类加载器定位路径的字节码类文件
				Class<?> clazz = getClassLoader().loadClass(configLocation);
				if (logger.isInfoEnabled()) {
					logger.info("Successfully resolved class for [" + configLocation + "]");
				}
				reader.register(clazz);
			}
			catch (ClassNotFoundException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Could not load class for config location [" + configLocation +
							"] - trying package scan. " + ex);
				}
				// 若容器类加载器加载定义路径的bean定义资源失败，则启用容器类路径扫描器扫描给定路径包及其子包中的类
				int count = scanner.scan(configLocation);
				if (logger.isInfoEnabled()) {
					if (count == 0) {
						logger.info("No annotated classes found for specified class/package [" + configLocation + "]");
					}
					else {
						logger.info("Found " + count + " annotated classes in package [" + configLocation + "]");
					}
				}
			}
		}
	}
}
```




