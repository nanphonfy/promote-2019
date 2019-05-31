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



