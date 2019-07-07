AOP到底能干嘛？

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


before->open开启事务
throwing->rollback回滚
after->commit提交事务

1、加载配置信息，解析成AopConfig；
2、交给AopProxyFactory，调用一个createAopProxy方法；
JdkDynamicAopProxy调用AdvisedSupport的getInterceptorsAndDynamicInterceptionAdvice方法得到方法拦截器链，并保存到一个容器（List，IOC容器是Map），递归执行拦截器方法proceed方法。

MethodInterceptor容器是List，IOC容器是一个ConcurrentHashMap。

最终就是由一个Advisor来调用切面中的方法。

Cglib调用链：getCallbacks、advised.getInterceptorsAndDynamicInterceptionAdvice

JDK代理和Cglib有何区别？AOP有自动判断，若实现interface默认用JDK，否则使用Cglib，最终目的都是扫描所有切点，形成拦截器链。

JDK性能更低。

springjdbc封装了jdbc操作的一个框架，必须依赖spring。本身基于模板模式开发，JDBCTemplate。
JDBC二次开发，来封装一个NoSql框架。

1、加载驱动类；
2、获取连接（被封装到DataSource里）；
3、创建语句集（预处理语句集合、标准语句集）；
4、执行语句集（执行事务操作）；
5、获取结果集（若是增删改，拿到一个int值，影响行数，若是查询，拿到一个ResultSet）。

mybatis是半自动ORM框架，hibernate是全自动。springjdbc采用template设计模式，只定义了一个RowMapper接口，mapping方法，没有实现，主要用来做ORM，由自己实现。
不具体做ORM，但制定了一套规范。

手写jdbc框架，做两件事：1、单表操作实现nosql；2、把手动ORM变成自动ORM。

spring jdbc有没有sql注入。底层默认使用预处理语句集（PreparedStatement)

>1、制定QueryRules规范，定义很多查询规则常量、查询规则保存到方法；  
2、提供抽象DAO类给用户实现（基于单表，传两泛型值-类、主键）；  
3、把实体配置信息解析成EntityOperation对象，实现自动ORM；  
4、在抽象的DAO调用查询方法，把queryRule做为参数传入；  
>>做为框架，要给用户提供很方便的操作窗口。eg.spring，ClassPathXmlApplication   cpxa;  
cpxa.getBean获取所有bean。    
QueryRule解决所有查询操作。（框架设计经典之处）  

>5、将传入的QueryRule交给QueryRuleSqlBulider构建出SQL语句（被拆分的）；  
6、拼接CRUD SQL语句；  
7、交给JdbcTemplate执行；  
8、调用EntityOperation的ORM过程；  
9、返回结果（基于单表，也无需强转类型）。