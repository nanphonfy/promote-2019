[TOC]


### 架构发展带来的问题
>- ①当服务越来越多时，服务URL配置管理变得很困难，F5硬件负载均衡器单点压力也越来越大。此时需要一个服务注册中心，动态注册、发现服务，使
服务位置透明。并通过在消费方获取服务提供方地址列表，实现软负载均衡和Failover，降低对F5硬件负载均衡器依赖，也减少部分成本；  
>- ②当进一步发展，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在谁之前启动。这时，需自动画出应用间的依赖关系图，帮助架构师理清关系；  
>- ③服务调用量越来越大，暴露服务的容量问题：该服务需多少机器支撑？该加几台机器？  
解决：第一步、统计服务每天调用量、响应时间（容量规划参考指标）；第二步，要可动态调整权重，在线上，将某机器权重加大，该过程中记录响应时间的变化，直到响应时间到达阀值，记录此时访问量，再以此访问量乘以机器数反推总容量。

#### 多版本支持
>设置不同版本的目的：接口升级后的兼容问题。Dubbo中配置不同版本的接口，会在Zookeeper地址中有多个协议url的体现：

```linux  
[zk: 192.168.25.154:2181(CONNECTED) 3] ls /dubbo/cn.nanphonfy.dubbo.IUserService/providers
[dubbo%3A%2F%2F192.168.25.1%3A20880%2Fcn.nanphonfy.dubbo.IUserService%3Fanyhost%3Dtrue%26application%3Ddubbo-server%26dubbo%3D2.5.3%26interface%3Dcn.nanphonfy.dubbo.IUserService%26methods%3Dlogin%26owner%3Dnp%26pid%3D20968%26revision%3D1.0.0%26side%3Dprovider%26timestamp%3D1572303668221%26version%3D1.0.0
, dubbo%3A%2F%2F192.168.25.1%3A20880%2Fcn.nanphonfy.dubbo.IUserService2%3Fanyhost%3Dtrue%26application%3Ddubbo-server%26dubbo%3D2.5.3%26interface%3Dcn.nanphonfy.dubbo.IUserService%26methods%3Dlogin%26owner%3Dnp%26pid%3D20968%26revision%3D1.0.1%26side%3Dprovider%26timestamp%3D1572303669568%26version%3D1.0.1]
```
### 主机绑定
>发布一个Dubbo服务时，会生成dubbo://ip:port的协议地址，该IP如何生成？可在ServiceConfig中找到如下代码。发现：生成绑定主机时，会通过一层层的判断，直到获取合法ip地址。

```java 
1.从配置文件获取host：NetUtils.isInvalidLocalHost(host)
2.host = InetAddress.getLocalHost().getHostAddress();
3.Socket socket = new Socket();
try {
    SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
    socket.connect(addr, 1000);
    host = socket.getLocalAddress().getHostAddress();
    break;
} finally {
    try {
        socket.close();
    } catch (Throwable e) {}
}
```

#### 集群容错
>容错机制：某种系统控制在一定范围内的一种允许或包容犯错情况的发生，eg.电脑运行一程序，有时无响应，会弹出提示框——立即结束or继续等待，然后根据选择执行对应操作。  
分布式架构下，网络、硬件、应用都可能发生故障，由于各服务间可能存在依赖关系，若一条链路中其中一节点故障，会导致雪崩效应。为减少某一节点故障的影响范围，才需构建容错服务，优雅处理这种中断响应结果。

- Dubbo的6种容错机制
>1.failsafe失败安全，可认为把错误吞掉（记录日志）；  
2.failover(默认)重试其他服务器；retries（2）  
3.failfast快速失败，失败后立马报错；  
4.failback失败后自动恢复；  
5.forking forks，设置并行数；  
6.broadcast广播，任意一台报错，则执行的方法报错配置方式：通过cluster方式，配置指定的容错方案。

-- 客户端或服务端的配置
```xml 
<dubbo:reference interface="cn.nanphonfy.dubbo.IPayService" id="npPayService" protocol="hessian" cluster="failfast"/>
```
#### 配置优先级别

```xml 
- client
<dubbo:reference interface="cn.nanphonfy.dubbo.IPayService" id="npPayService" protocol="hessian" cluster="failfast" timeout="50"/>

-server
<dubbo:reference interface="cn.nanphonfy.dubbo.IPayService" id="npPayService" protocol="hessian" cluster="failfast" timeout="500"/>
```
>客户端的配置优先服务端：  
1. 方法级别优先，然后是接口，最后是全局配置； 
2. 若级别一样，客户端优先，eg.retries、loadbalance、cluster（客户端）、timeout（服务端）。

#### 服务降级
>目的：保证核心服务可用。  
降级有几个层面分类：自动降级和人工降级；   按功能分：读服务降级和写服务降级；  
>- 1.对一些非核心服务做人工降级，大促前通过降级开关关闭哪些推荐内容、评价等对主流程无影响的功能；  
>- 2.故障降级，eg.调用远程服务挂了、网络故障、或RPC服务异常。可直接降级，方案eg.设置默认值、采用兜底数据（推荐行为的广告挂了，可提前准备静态页面返回）等；  
>- 3.限流降级，在秒杀中流量较集中且流量特大的情况，突发访问量特大可能会导致系统支撑不了。此时可采用限流限制访问量。当达到阀值时，后续请求被降级，eg.进入排队页面、跳转到错误页。
>>dubbo降级方式：Mock  
>- 1.client端创建TestMock类，实现对应接口，名称必须以Mock结尾；  
>- 2.client端的xml配置文件中，添加如下配置，add一个mock属性指向创建的TestMock；  
>- 3.模拟错误（设置timeout），模拟超时异常，运行测试代码即可访问TestMock。当服务端故障解除后，调用过程恢复正常。  

- 超时会走mock返回
```java 
<dubbo:reference interface="cn.nanphonfy.dubbo.IPayService" id="npPayService" protocol="hessian" cluster="failfast" timeout="1" mock="cn.nanphonfy.dubbo.TestMock"/>

public class TestMock implements IUserService{
    @Override
    public String login(String name, String pwd) {
        return "系统繁忙，尊敬的："+name;
    }
}

// 客户端超时时长为1毫秒，超时走我们客户端的mock
```

### 配置优先级别
>以timeout为例，显示配置查找顺序，retries、loadbalance等也类似。  
1.方法级优先，接口级次之，全局配置再次之；  
2.若级别一样，则消费方优先，提供方次之。  
>>建议服务提供方设置超时，一个方法需执行多长时间，服务提供方更清楚，若一个消费方同时引用多个服务，则不需关心每个服务的超时设置。

### Dubbo SPI VS JAVA SPI
>Dubbo中SPI是很核心的机制，贯穿几乎所有流程。搞懂这块内容，是了解源码的关键因素。  
Dubbo基于Java原生SPI机制思想。
- JAVA的SPI机制
>SPI（service provider interface），JDK内置的一种服务提供发现机制，有很多框架都用它做服务的扩展发现，eg.JDBC、日志框架。  
它是一种动态替换发现的机制。eg.若定义一个规范，需第三方厂商实现，对于应用方而言，只需集成对应厂商的插件，即可完成对应规范的实现机制，形成一种插拔式的扩展手段。

#### 实现一个SPI机制


##### 代码演示


##### SPI规范总结
>实现SPI，需按照SPI本身定义的规范进行配置，SPI规范如下：
>>1.需在classpath下创建一个目录，该目录命名必须是：META-INF/services；  
2.在该目录下创建一个properties文件，该文件需满足几个条件：
>>a)文件名必须是扩展接口的全路径名称；  
b)文件内部描述的是该扩展接口的所有实现类；  
c)文件的编码格式是UTF-8。  

>3.通过java.util.ServiceLoader的加载机制来发现

##### SPI实际应用
>SPI应用在很多地方，JDK官方提供了java.sql.Driver的驱动扩展点。  
以连接Mysql为例，我们需添加mysql-connector-java依赖。然后，可在该jar包中找到SPI的配置信息。故java.sql.Driver由各数据库厂商自行实现。
>>spring框架也有大量实现。

##### SPI缺点
>1.JDK标准的SPI会一次性加载实例化扩展点的所有实现。在META-INF/service下的文件里加了N个实现类，那么JDK启动时都会一次性全部加载。若有的扩展点实现初始化很耗时或有些实现类并没用到，很浪费资源；  
2.若扩展点加载失败，会导致调用方报错，该错误很难定位到是该原因。

#### Dubbo优化后的SPI实现
##### 基于Dubbo提供的SPI规范实现自己的扩展 
>了解Dubbo的SPI机制前，先通过一段代码初步了解Dubbo的实现方式。


##### Dubbo的SPI机制规范
>大部分思想都和SPI一样，只是下面两个地方有差异。
>- 1.需在resource目录下配置META-INF/dubbo或META-INF/dubbo/internal或META-INF/services，并基于SPI接口创建一个文件；  
>- 2.文件名称和接口名称保持一致，文件内容SPI有差异，内容是key，对应value。  

### Dubbo SPI机制源码
>为什么传入一个myProtocol就能获得自定义的DefineProtocol对象？getAdaptiveExtension是一个什么东西？
```java 
①Protocol protocol = ExtensionLoader. getExtensionLoader(Protocol.class).getExtension("myProtocol");
②Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
#### 源码阅读入口
>分析完②的代码，对①的理解会很容易。  
将②分成两段，一段getExtensionLoader、 另一段getAdaptiveExtension。  
>>第一段通过Class参数获得ExtensionLoader对象，类似工厂模式；  
第二段getAdaptiveExtension，获得一个自适应的扩展点。

#### Extension源码结构
>了解源码结构，建立一个全局认识。结构图：

![image](https://github.com/nanphonfy/note-images/blob/master/promote-2019/distributed/11/pkg-extension.png?raw=true)

初步了解这些代码在扩展点中的痕迹。

#### Protocol源码
>Protocol源码有两个注解：①类级别上的@SPI(“dubbo”)；②@Adaptive。
>>@SPI：当前该接口是一个扩展点，可实现自己的扩展，默认扩展点是DubboProtocol；  
@Adaptive：一个自适应扩展点，在方法级别上，会动态生成一个适配器类。
```java 
// com.alibaba.dubbo.rpc.Protocol
@SPI("dubbo")
public interface Protocol {
    /**
     * 获取缺省端口，当用户没有配置端口时使用。
     * 
     * @return 缺省端口
     */
    int getDefaultPort();

    /**
     * 暴露远程服务：<br>
     * 1. 协议在接收请求时，应记录请求来源方地址信息：RpcContext.getContext().setRemoteAddress();<br>
     * 2. export()必须是幂等的，也就是暴露同一个URL的Invoker两次，和暴露一次没有区别。<br>
     * 3. export()传入的Invoker由框架实现并传入，协议不需要关心。<br>
     * 
     * @param <T> 服务的类型
     * @param invoker 服务的执行体
     * @return exporter 暴露服务的引用，用于取消暴露
     * @throws RpcException 当暴露服务出错时抛出，比如端口已占用
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * 引用远程服务：<br>
     * 1. 当用户调用refer()所返回的Invoker对象的invoke()方法时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法。<br>
     * 2. refer()返回的Invoker由协议实现，协议通常需要在此Invoker中发送远程请求。<br>
     * 3. 当url中有设置check=false时，连接失败不能抛出异常，并内部自动恢复。<br>
     * 
     * @param <T> 服务的类型
     * @param type 服务的类型
     * @param url 远程服务的URL地址
     * @return invoker 服务的本地代理
     * @throws RpcException 当连接服务提供方失败时抛出
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    /**
     * 释放协议：<br>
     * 1. 取消该协议所有已经暴露和引用的服务。<br>
     * 2. 释放协议所占用的所有资源，比如连接和端口。<br>
     * 3. 协议在释放后，依然能暴露和引用新的服务。<br>
     */
    void destroy();
}
```

#### getExtensionLoader
>该方法需一个Class类型的参数，该参数（必须是接口，必须被@SPI注解注释，否则拒绝处理）表示希望加载的扩展点类型。  
检查通过后首先会检查ExtensionLoader缓存中是否已存在该扩展对应的ExtensionLoader，若有则直接返回，否则创建一个新ExtensionLoader负责加载该扩展实现，同时将其缓存起来。可以看到对于每一个扩展，dubbo中只会有一个对应的ExtensionLoader实例。
```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    if(!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    if(!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    // private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}

/**
 * ExtensionLoader提供了私有构造函数
 * 且对两个成员变量type/objectFactory赋值
 * 而objectFactory赋值的意义是什么？先留个悬念
 * @param type
 */
private ExtensionLoader(Class<?> type) {
    this.type = type;
    // private final ExtensionFactory objectFactory;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
#### getAdaptiveExtension
>通过getExtensionLoader获得了对应的ExtensionLoader实例后，再调用getAdaptiveExtension()方法获得一个自适应扩展点。
>>ps：自适应扩展点实际上就是一个适配器。

>该方法主要做几个事情：
①从cacheAdaptiveInstance内存缓存获得一个对象实例；  
②若实例为空，第一次加载，则通过双重检查锁去创建一个适配器扩展点。

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
public T getAdaptiveExtension() {
    // private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if(createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }
        else {
            throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```

#### createAdaptiveExtension
>这段代码有两结构：①injectExtension；②getAdaptiveExtensionClass()  
需先了解getAdaptiveExtensionClass方法做了什么？从后面的.newInstance来看，应该是获得一个类且进行实例)。  
```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private T createAdaptiveExtension() {
    try {
        // 可以实现扩展点的注入
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);
    }
}
```
#### getAdaptiveExtensionClass
>是获得适配器扩展点的类。  
做了两件事：  
①getExtensionClasses()加载所有路径下的扩展点；  
②createAdaptiveExtensionClass()动态创建一个扩展点cachedAdaptiveClass。有个判断，用来判断当前Protocol扩展点是否存在一个自定义适配器。若有，则直接返回自定义适配器，否则，动态创建，该值在getExtensionClasses中赋值。

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    // private volatile Class<?> cachedAdaptiveClass = null;
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```
#### createAdaptiveExtensionClass
>动态生成适配器代码，及动态编译。
>①createAdaptiveExtensionClassCode,动态创建一个字节码文件，返回字符串；  
②compiler.compile进行编译（默认使用javassist）；  
③通过ClassLoader加载到jvm。

```java 
// 创建一个适配器扩展点（创建一个动态字节码文件）
private Class<?> createAdaptiveExtensionClass() {
    // 生成字节码代码
    String code = createAdaptiveExtensionClassCode();
    // 获得类加载器
    ClassLoader classLoader = findClassLoader();
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 动态编译字节码
    return compiler.compile(code, classLoader);
}
```

####  server
- dubbo-server.xml
```
<!--提供方信息-->
<dubbo:application name="dubbo-server" owner="np"/>

<!--注册中心，不依赖-->
<!--<dubbo:registry address="N/A"/>-->
<dubbo:registry address="zookeeper://192.168.25.154:2181"/>
<!--多注册中心，eg.中英文网站，在具体的service要指定register的地址，在registry要指定id，eg.id=zk1,zk2-->

<!--多协议支持-->
<dubbo:protocol name="dubbo" port="20880"/>
<dubbo:protocol name="hessian" port="8880"/>

<dubbo:service interface="cn.nanphonfy.dubbo.IUserService" ref="npUserService" protocol="dubbo,hessian"/>
<dubbo:service interface="cn.nanphonfy.dubbo.IPayService" ref="npPayService" protocol="hessian"/>

<bean id="npUserService" class="cn.nanphonfy.dubbo.UserServiceImpl"/>
<bean id="npPayService" class="cn.nanphonfy.dubbo.PayServiceImpl"/>
```

- Bootstrap启动函数
```java 
public class Bootstrap {
    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("META-INF.spring/dubbo-server.xml");
        context.start();
        // 阻塞当前线程
        System.in.read();
    }
}
```
#### 负载均衡和集群
- dubbo-cluster1.xml
```java 
<!--提供方信息-->
<dubbo:application name="dubbo-server" owner="np"/>

<dubbo:registry address="zookeeper://192.168.25.154:2181"/>

<!--多协议支持-->
<dubbo:protocol name="dubbo" port="20880"/>

<dubbo:service interface="cn.nanphonfy.dubbo.IUserService" ref="npUserService" protocol="dubbo"/>

<bean id="npUserService" class="cn.nanphonfy.dubbo.UserServiceImpl"/>
```
- dubbo-cluster2.xml

```java 
<!--提供方信息-->
<dubbo:application name="dubbo-server" owner="np"/>

<dubbo:registry address="zookeeper://192.168.25.154:2181"/>

<!--多协议支持-->
<dubbo:protocol name="dubbo" port="20881"/>

<dubbo:service interface="cn.nanphonfy.dubbo.IUserService" ref="npUserService" protocol="dubbo"/>

<bean id="npUserService" class="cn.nanphonfy.dubbo.UserServiceImpl2"/>
```

- BootstrapCluster1、2
```java 
public static void main(String[] args) throws IOException {
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("META-INF.spring/dubbo-cluster1.xml");
    context.start();
    // 阻塞当前线程
    System.in.read();
}

public class BootstrapCluster2 {
    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("META-INF.spring/dubbo-cluster2.xml");
        context.start();
        // 阻塞当前线程
        System.in.read();
    }
}
```
### client

- dubbo-client.xml
```
<!--提供方信息-->
<dubbo:application name="dubbo-client" owner="np"/>

<!--注册中心，不依赖-->
<!--<dubbo:registry address="N/A"/>-->
<!--本地缓存d:/dubbotest-->
<!--<dubbo:registry address="zookeeper://192.168.25.154:2181" file="d:\\dubbotest"/>-->
<dubbo:registry address="zookeeper://192.168.25.154:2181"/>

<dubbo:protocol name="dubbo" port="20880"/>

<dubbo:reference interface="cn.nanphonfy.dubbo.IUserService" id="npUserService" protocol="dubbo"/>
<!--不走zookeeper-->
<!--<dubbo:reference interface="cn.nanphonfy.dubbo.IUserService" id="npUserService"-->
<!--url="dubbo://localhost:20880/cn.nanphonfy.dubbo.IUserService"/>-->
<dubbo:reference interface="cn.nanphonfy.dubbo.IPayService" id="npPayService" protocol="hessian"/>

<!--解决循环依赖，eg.c1调用s1，s1服务未启动，则c1调不通,可加check解决，eg.check=false-->
<!--<dubbo:reference interface="cn.nanphonfy.dubbo.IPayService" id="npPayService" protocol="hessian" check="false"/>-->
```

- client-app
```java 
public class App {
    public static void main(String[] args) throws IOException {
        /*IUserService userService = null;
        System.out.println(userService.login("nan","123"));*/
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("dubbo-client.xml");
        /*IPayService payService = (IPayService) context.getBean("npPayService");
        payService.payFor(399.0);*/
        for (int i = 0; i < 10; i++) {
            IUserService userService = (IUserService) context.getBean("npUserService");
            System.out.println(userService.login("np","123"));
        }

        // 观察zookeeper生成的consumers
        System.in.read();
    }
}
```

```
[zk: 192.168.25.154:2181(CONNECTED) 3] ls /
[np-test, dubbo, zk-persis-np, zookeeper]
[zk: 192.168.25.154:2181(CONNECTED) 4] ls /dubbo
[cn.nanphonfy.dubbo.IUserService]
[zk: 192.168.25.154:2181(CONNECTED) 6] ls /dubbo/cn.nanphonfy.dubbo.IUserService
[configurators, providers]
[zk: 192.168.25.154:2181(CONNECTED) 7] ls /dubbo/cn.nanphonfy.dubbo.IUserService/providers
[dubbo%3A%2F%2F192.168.25.1%3A20880%2Fcn.nanphonfy.dubbo.IUserService%3Fanyhost%3Dtrue%26application%3Ddubbo-server%26dubbo%3D2.5.3%26interface%3Dcn.nanphonfy.dubbo.IUserService%26methods%3Dlogin%26owner%3Dnp%26pid%3D20300%26side%3Dprovider%26timestamp%3D1571068902907]
```


---

```java 
https://blog.csdn.net/xiaoxufox/article/details/75117992#getactivateextension
/** 加载扩展类 */
private Map<String, Class<?>> getExtensionClasses() {
	// private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String,Class<?>>>();
	Map<String, Class<?>> classes = cachedClasses.get();
	if (classes == null) {
		synchronized (cachedClasses) {
			classes = cachedClasses.get();
			if (classes == null) {
				classes = loadExtensionClasses();
				cachedClasses.set(classes);
			}
		}
	}
	return classes;
}

/** 从不同目录加载扩展实现 */
private Map<String, Class<?>> loadExtensionClasses() {
	// type->Protocol.class,得到SPI的注解
	final SPI defaultAnnotation = type.getAnnotation(SPI.class);
	// 若不为空
	if(defaultAnnotation != null) {
		// 获取注解标记值
		String value = defaultAnnotation.value();
		if(value != null && (value = value.trim()).length() > 0) {
			String[] names = NAME_SEPARATOR.split(value);
			if(names.length > 1) {
				throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
						+ ": " + Arrays.toString(names));
			}
			if(names.length == 1) cachedDefaultName = names[0];
		}
	}
	
	// 从3个目录加载
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
	// META-INF/dubbo/internal/
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
	// META-INF/dubbo/
    loadFile(extensionClasses, DUBBO_DIRECTORY);
	// META-INF/services/
    loadFile(extensionClasses, SERVICES_DIRECTORY);
	return extensionClasses;
}

/** 从目录加载扩展实现 */
private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
	String fileName = dir + type.getName();
	try {
		Enumeration<java.net.URL> urls;
		ClassLoader classLoader = findClassLoader();
		// https://www.cnblogs.com/drwong/p/5389631.html 关于getSystemResource, getResource的总结
		if (classLoader != null) {
			urls = classLoader.getResources(fileName);
		} else {
			urls = ClassLoader.getSystemResources(fileName);
		}
		if (urls != null) {
			while (urls.hasMoreElements()) {
				java.net.URL url = urls.nextElement();
				try {
					BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
					try {
						String line = null;
						while ((line = reader.readLine()) != null) {
							final int ci = line.indexOf('#');
							if (ci >= 0) line = line.substring(0, ci);
							line = line.trim();
							if (line.length() > 0) {
								try {
									String name = null;
									int i = line.indexOf('=');
									// 文件采用name = value方法
									if (i > 0) {
										// 文件采用properties方式的配置，name=实现类的全路径
										name = line.substring(0, i).trim();
										line = line.substring(i + 1).trim();
									}
									if (line.length() > 0) {
										Class<?> clazz = Class.forName(line, true, classLoader);
										// 必须是该扩展点的实现
										if (! type.isAssignableFrom(clazz)) {
											throw new IllegalStateException("Error when load extension class(interface: " +
													type + ", class line: " + clazz.getName() + "), class " 
													+ clazz.getName() + "is not subtype of interface.");
										}
										// 判断是否有自定义适配器类，若有，后面获取适配器时，直接用该创建返回，不用dubbo动态创建
										if (clazz.isAnnotationPresent(Adaptive.class)) {
											// private volatile Class<?> cachedAdaptiveClass = null;
											if(cachedAdaptiveClass == null) {
												cachedAdaptiveClass = clazz;
											} else if (! cachedAdaptiveClass.equals(clazz)) {
												throw new IllegalStateException("More than 1 adaptive class found: "
														+ cachedAdaptiveClass.getClass().getName()
														+ ", " + clazz.getClass().getName());
											}
										} else {
											try {
												// 处理实现类的包裹类，必须是构造器注入扩展点，后期获取具体扩展点实现类时，会使用包裹类封装下
												clazz.getConstructor(type);
												// private Set<Class<?>> cachedWrapperClasses;
												Set<Class<?>> wrappers = cachedWrapperClasses;
												if (wrappers == null) {
													cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
													wrappers = cachedWrapperClasses;
												}
												wrappers.add(clazz);
											} catch (NoSuchMethodException e) {
												clazz.getConstructor();
												if (name == null || name.length() == 0) {
													name = findAnnotationName(clazz);
													if (name == null || name.length() == 0) {
														if (clazz.getSimpleName().length() > type.getSimpleName().length()
																&& clazz.getSimpleName().endsWith(type.getSimpleName())) {
															name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
														} else {
															throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
														}
													}
												}
												String[] names = NAME_SEPARATOR.split(name);
												if (names != null && names.length > 0) {
													// 判断Activate注解，会缓存Activate注解实现
													Activate activate = clazz.getAnnotation(Activate.class);
													if (activate != null) {
														// private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();
														cachedActivates.put(names[0], activate);
													}
													for (String n : names) {
														// private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();
														if (! cachedNames.containsKey(clazz)) {
															cachedNames.put(clazz, n);
														}
														Class<?> c = extensionClasses.get(n);
														if (c == null) {
															// 缓存扩展实现
															extensionClasses.put(n, clazz);
														} else if (c != clazz) {
															throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
														}
													}
												}
											}
										}
									}
								} catch (Throwable t) {
									IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + url + ", cause: " + t.getMessage(), t);
									exceptions.put(line, e);
								}
							}
						} // end of while read lines
					} finally {
						reader.close();
					}
				} catch (Throwable t) {
					logger.error("Exception when load extension class(interface: " +
										type + ", class file: " + url + ") in " + url, t);
				}
			} // end of while urls
		}
	} catch (Throwable t) {
		logger.error("Exception when load extension class(interface: " +
				type + ", description file: " + fileName + ").", t);
	}
}

/** Dubbo生成适配类 */
private Class<?> createAdaptiveExtensionClass() {
	// 动态生成适配器代码
	String code = createAdaptiveExtensionClassCode();
	ClassLoader classLoader = findClassLoader();
	com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
	// 动态编译生成适配器类
	return compiler.compile(code, classLoader);
}
```
