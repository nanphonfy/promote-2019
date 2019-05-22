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