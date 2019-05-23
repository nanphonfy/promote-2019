spring IOC体系结构

BeanFactory
spring bean的创建是典型的工厂模式，这一系列bean工厂(IOC容器)，相互关系图：

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

- org.springframework.beans.factory.BeanFactory
```java  
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
```



