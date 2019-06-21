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
// org.springframework.context.annotation.AnnotatedBeanDefinitionReader

```
