[TOC]

### Dubbo Extension扩展点
>SPI->Extension  
Extension.getExtensionLoader().getAdaptiveExtension(); //动态适配器的扩展点  
最终会生成动态代理类：Protocol$Adaptive  

>com.alibaba.dubbo.common.extension.ExtensionLoader的injectExtension方法采用了依赖注入思想，eg.加载的扩展点里存在一个属性，该属性也是扩展点时，会对其进行注入。  
eg.AdaptiveCompiler是自适应扩展点，但它没实现，而是通过动态加载实现（ExtensionLoader.getExtensionLoader(Compiler.class)）

```java 
@Adaptive
public class AdaptiveCompiler implements Compiler {
    private static volatile String DEFAULT_COMPILER;
    public static void setDefaultCompiler(String compiler) {
        DEFAULT_COMPILER = compiler;
    }

    public Class<?> compile(String code, ClassLoader classLoader) {
        Compiler compiler;
        ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
        String name = DEFAULT_COMPILER; // copy reference
        if (name != null && name.length() > 0) {
            compiler = loader.getExtension(name);
        } else {
            compiler = loader.getDefaultExtension();
        }
        return compiler.compile(code, classLoader);
    }
}
```

>- @SPI("")  
>- @Adaptive（若该注解在方法级别上，会动态生成一个自适应的适配器；若在类级别上，直接加载自定义的自适应适配器）。  
Extension.getExtensionLoader().getExtension(“”); //加载一个指定名称的扩展点

案例：上一文件。

>加载自适应扩展点需要用到的包装类：ProtocolFilterWrapper(ProtocolListenerWrapper(Protocol$Adaptive))

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                if (method.getName().startsWith("set") && method.getParameterTypes().length == 1 && Modifier
                        .isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        String property = method.getName().length() > 3 ?
                                method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) :
                                "";
                        // private final ExtensionFactory objectFactory;
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error(
                                "fail to inject via method " + method.getName() + " of interface " + type.getName()
                                        + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```
>Object object = objectFactory.getExtension(pt, property);

```
// com.alibaba.dubbo.common.extension.ExtensionFactory
@SPI 
public interface ExtensionFactory {
    /**
     * Get extension.
     *
     * @param type object type.
     * @param name object name.
     * @return object instance.
     */
    <T> T getExtension(Class<T> type, String name);
}

其实现类有：
public class AdaptiveExtensionFactory implements ExtensionFactory
public class SpiExtensionFactory implements ExtensionFactory
public class SpringExtensionFactory implements ExtensionFactory
```

>ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
objectFactory => AdaptiveExtensionFactory

```java 
// com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
@Adaptive 
public class AdaptiveExtensionFactory implements ExtensionFactory {
    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```

- ExtensionLoader.getExtensionLoader(ExtensionFactory.class);加载所有扩展点（factories）。
>我们搜索配置文件，所有的出处如下：  
dubbo-common/META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory  
dubbo-config/dubbo-config-spring/META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory

```xml 
com.alibaba.dubbo.common.extension.ExtensionFactory->
adaptive=com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
spi=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory

com.alibaba.dubbo.common.extension.ExtensionFactory->
spring=com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory
```
>方法： 动态创建一个自适应的适配器  
类：直接加载当前的适配器

- dubbo网站链接
>http://dubbo.apache.org/zh-cn/docs/dev/impls/protocol.html

### 服务发布及注册流程
>spring写的标签，和spring官方无关系，意味着它是基于spring的扩展。  
spring提供的扩展类如下：NamespaceHandler、BeanDefinitionParse，dubbo的标签也是基于spring做拓展的。

```java 
\dubbo-config\dubbo-config-spring
\META-INF\spring.handlers
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler

// com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
	static {
		Version.checkDuplicate(DubboNamespaceHandler.class);
	}

	public void init() {
	    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}
```
>dubbo源码是基于领域驱动的设计思想，可以看下源码结构。  

- DubboBeanDefinitionParser（解析spring配置文件）->ServiceBean->ServiceConfig(export->doExport->doExportUrls->doExportUrlsFor1Protocol)->Dubbo$Protocol(export)->RegistryProtocol(export)

```java 
// com.alibaba.dubbo.config.spring.ServiceBean
// public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware 
public void afterPropertiesSet() throws Exception {
    ......
    // 非常关键的，进入服务发布的过程
    if (! isDelay()) {
        export();
    }
}
```
>启动一个服务时做了啥（调用注册中心发布服务到zookeeper、启动一个netty 服务）

```java 
// com.alibaba.dubbo.config.ServiceConfig
private void doExportUrls() {
    // 是不是获得注册中心的
    List<URL> registryURLs = loadRegistries(true);
    // 是不是支持多协议发布
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}

// registryURLs = registry://192.168.25.154:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-server&dubbo=2.5.3&owner=np&pid=16932&registry=zookeeper&timestamp=1572877311908


private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    ......
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
    ......
    // 【核心代码】
    if (registryURLs != null && registryURLs.size() > 0 && url.getParameter("register", true)) {
        for (URL registryURL : registryURLs) {
            url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
            URL monitorUrl = loadMonitor(registryURL);
            if (monitorUrl != null) {
                url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
            }
            if (logger.isInfoEnabled()) {
                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url
                        + " to registry " + registryURL);
            }
            // 通过proxyFactory来获取Invoker对象
            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass,
                    registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
            // 注册服务
            // private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
            Exporter<?> exporter = protocol.export(invoker);
            // 将exporter添加到list中
            exporters.add(exporter);
        }
    }else {
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);

        Exporter<?> exporter = protocol.export(invoker);
        exporters.add(exporter);
    }
}

// protocolConfig->对应：<dubbo:protocol name="dubbo" port="20880" />

// 【URL url】dubbo://192.168.25.1:20880/cn.nanphonfy.dubbo.IUserService?anyhost=true&application=dubbo-server&dubbo=2.5.3&interface=cn.nanphonfy.dubbo.IUserService&methods=login&owner=np&pid=3232&revision=1.0.0&side=provider&timestamp=1572877693723&version=1.0.0
```

>ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()返回动态适配器：Protocol$Adaptive。

>registryURL的值为：registry://192.168.25.154:2181...，故此处使用适配器，若不使用则要写很多ifelse。  
eg.registry://192.168…、dubbo:// 、rmi://、hessian://  
if(dubbo)  
else if(rmi)  
elseif (hessian)  
else if(myprotocol…  

>若自定义了一个protocol，那我们还得改代码。所以适配器的作用是：能够根据当前请求运行时做适配，动态自适应。

- Protocol$Adaptive
```java 
import com.alibaba.dubbo.common.extension.ExtensionLoader;
//package com.alibaba.dubbo.rpc;

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
        if (arg1 == null)
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException(
                    "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString()
                            + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
                .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0)
            throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException(
                    "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString()
                            + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
                .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```
>Protocol extension = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);的extension：
指定名称的Protocol->该场景下，具体是啥Protocol（RegistryProtocol）

```java 
invoker的url为->registry://192.168.25.154:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-server&dubbo=2.5.3&export=dubbo%3A%2F%2F192.168.25.1%3A20880%2Fcn.nanphonfy.dubbo.IUserService%3Fanyhost%3Dtrue%26application%3Ddubbo-server%26dubbo%3D2.5.3%26interface%3Dcn.nanphonfy.dubbo.IUserService%26methods%3Dlogin%26owner%3Dnp%26pid%3D14144%26revision%3D1.0.0%26side%3Dprovider%26timestamp%3D1572909163386%26version%3D1.0.0&owner=np&pid=14144&registry=zookeeper&timestamp=1572909163178
```
>Protocol extension = 
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(“registry”);，指定扩展点，跟适配器扩展点不一样。

>registry=com.alibaba.dubbo.registry.integration.RegistryProtocol  
extension=RegistryProtocol  


```java 
// com.alibaba.dubbo.registry.integration.RegistryProtocol
//本地发布服务,启动服务
final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

private <T> ExporterChangeableWrapper<T>  doLocalExport(final Invoker<T> originInvoker){
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>)protocol.export(invokerDelegete), originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return (ExporterChangeableWrapper<T>) exporter;
}


protocol.export(invokerDelegete),originInvoker)  
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()
Protocol$Adatpive. export()
```

---

### Dubbo的Extension源码分析  
>前面基于ExtensionLoader.getExtensionLoader().getAdaptiveExtension()入口进行源码分析。  

- getAdaptiveExtension流程  
`getExtensionLoader->getAdaptiveExtension->（injectExtension依赖注入）getAdaptiveExtensionClass->是否存在自定义AdaptiveClass->是（结束）|否->createAdaptiveExtensionClass->Protocol$Adaptive`

#### injectExtension
```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private T injectExtension(T instance) {
    try {
        // private final ExtensionFactory objectFactory;
        // getExtensionLoader时赋值
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                // 判断是否以set开头，通过set进行动态注入
                if (method.getName().startsWith("set") && method.getParameterTypes().length == 1 && Modifier
                        .isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        String property = method.getName().length() > 3 ?
                                method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) :
                                "";
                        // 根据类型、名称获得对应扩展点
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName() + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```
>扩展点自动注入是根据setter方法对应的参数类
型和property名称从ExtensionFactory中查询，若有返回扩展点实例，就注入操作。  
讲解@Adaptive时提到过AdaptiveCompiler类，该类有一个setDefaultCompiler方法，本身没实现compile，而是基于DEFAULT_COMPILER，然后加载指定扩展点进行动态调用。那么该DEFAULT_COMPILER ，就是在injectExtension方法中进行注入的。

- AdaptiveCompiler
```java 
在上面......
```

#### objectFactory
>injectExtension方法中，首先判断objectFactory是否为空。在获得ExtensionLoader时，就对 objectFactory进行了初始化。  

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
>通过ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()获得一个自适应扩展点，ExtensionFactory接口是一个扩展点，且有一个自己实现的自适应扩展点AdaptiveExtensionFactory。 
```java 
@SPI
public interface ExtensionFactory
AdaptiveExtensionFactory
SpiExtensionFactory
SpringExtensionFactory
```
>注意：@Adaptive加载到类上表示这是一个自定义适配器类，表示再调用getAdaptiveExtension时，
不需走上面这么复杂的过程。会直接加载到AdaptiveExtensionFactory。（此处代码在loadFile），然后在getAdaptiveExtensionClass（）方法处有判断。

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
......
if (clazz.isAnnotationPresent(Adaptive.class)) {
    if(cachedAdaptiveClass == null) {
        cachedAdaptiveClass = clazz;
    } else if (! cachedAdaptiveClass.equals(clazz)) {
        throw new IllegalStateException("More than 1 adaptive class found: "
                + cachedAdaptiveClass.getClass().getName()
                + ", " + clazz.getClass().getName());
    }
}
......
```
>除自定义自适应适配器类外，还有两个实现类，一个是SPI，一个是Spring，AdaptiveExtensionFactory，它轮询这2个，从一个中获取到就返回。

```java 
// com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
public <T> T getExtension(Class<T> type, String name) {
    for (ExtensionFactory factory : factories) {
        T extension = factory.getExtension(type, name);
        if (extension != null) {
            return extension;
        }
    }
    return null;
}
```
### 服务端发布流程
#### Spring对外留出了扩展
>dubbo是基于spring配置来实现服务发布的，是基于spring的扩展写自己的标签。  
总结：可通过spring扩展机制扩展自己的标签。dubbo配置文件中的<dubbo:service>，就是自定义扩展标签。  
实现自定义扩展，有三个步骤（spring定义了两个接口，用来实现扩展）  
>- 1.NamespaceHandler:注册一堆BeanDefinitionParser，利用他们进行解析；  
>- 2.BeanDefinitionParser:用于解析每个element；  
>- 3.Spring默认加载jar包下的META-INF/spring.handlers文件寻找对应的NamespaceHandler。  

>以下是dubbo-config模块下的dubbo-config-spring  
>>dubbo-config/dubbo-config-spring  
META-INF/spring.handlers

#### dubbo接入实现
>dubbo中spring扩展就是使用spring自定义类型，故同样也有NamespaceHandler、BeanDefinitionParser。而NamespaceHandler是DubboNamespaceHandler

```java 
// com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
	static {
		Version.checkDuplicate(DubboNamespaceHandler.class);
	}

	public void init() {
	    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}
```
>BeanDefinitionParser全部都使用了DubboBeanDefinitionParser，它主要做的事：把不同的配置分别转化成spring容器中的bean对象。  
>application 对应 ApplicationConfig  
registry 对应 RegistryConfig  
monitor 对应 MonitorConfig  
provider 对应 ProviderConfig  
consumer 对应 ConsumerConfig  
......

>为了在spring启动时，相应启动provider发布服务、注册服务的过程，同时为让客户端在启动时自动订阅发现服务，加入两个bean：ServiceBean、ReferenceBean。  
>>分别继承了ServiceConfig和ReferenceConfig，
同时还分别实现了InitializingBean 、DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware。 
```java 
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware

public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean 
```
>InitializingBean接口为bean提供了初始化方法的方式，只包括afterPropertiesSet，凡是继承该接口的类，在初始化bean时会执行该方法；  
DisposableBean bean被销毁时，spring容器会自动执行destory，eg.释放资源；  
ApplicationContextAware实现了该接口的bean，当spring容器初始化时，会自动将ApplicationContext注入进来；  
ApplicationListener、ApplicationEvent事件监听，spring容器启动后会发一个事件通知；  
BeanNameAware获得自身初始化时，本身的bean的 id属性。  

- 基本实现思路：
>1.利用spring的解析收集xml中的配置信息，把这些配置信息存储到serviceConfig中；  
2.调用ServiceConfig的export进行服务的发布和注册。

### 服务的发布过程
>serviceBean是服务发布的切入点，通过afterPropertiesSet方法，调用export()方法进行发布。export为父类ServiceConfig中的方法，故跳转到SeviceConfig类中的export方法delay的使用。  

```java 
// com.alibaba.dubbo.config.ServiceConfig
public synchronized void export() {
......
    if (delay != null && delay > 0) {
        Thread thread = new Thread(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(delay);
                } catch (Throwable e) {
                }
                doExport();
            }
        });
        thread.setDaemon(true);
        thread.setName("DelayExportServiceThread");
        thread.start();
    } else {
        doExport();
    }
}
```
>delay作用：延迟暴露->Thread.sleep(delay)
>>1.export是synchronized修饰的方法。暴露过程是原子操作，正常情况不会出现锁竞争问题，毕竟初始化过程大多数情况下都是单一线程操作，这里联想到了spring的初始化流程，也进行了加锁操作（给我们平时设计一个不错的启示：初始化流程的性能调优 优先级应放比较低，但安全的优先级应该放比较高）；  
2. doExport()同样是一堆初始化代码。

#### export过程
```java 
// com.alibaba.dubbo.config.ServiceConfig
protected synchronized void doExport() {
    ......
    doExportUrls();
}

private void doExportUrls() {
    // 是不是获得注册中心的
    List<URL> registryURLs = loadRegistries(true);
    // 是不是支持多协议发布
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```
>doExport()最终会调用到doExportUrls()，该protocols长这样`<dubbo:protocol name="dubbo" 
port="20888" id="dubbo" />`   
protocols也是根据配置装配出来的。

- doExportUrlsFor1Protocol是怎样将服务暴露出去的？具体逻辑如下：
```java 
// com.alibaba.dubbo.config.ServiceConfig
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    ......
    //配置为none不暴露
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
        //配置不是remote的情况下做本地暴露 (配置为remote，则表示只暴露远程服务)
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        //如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露远程服务)
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
            if (registryURLs != null && registryURLs.size() > 0 && url.getParameter("register", true)) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url
                                + " to registry " + registryURL);
                    }
                    // 通过proxyFactory来获取Invoker对象
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass,
                            registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    // 注册服务
                    Exporter<?> exporter = protocol.export(invoker);
                    // 将exporter添加到list中
                    exporters.add(exporter);
                }
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);

                Exporter<?> exporter = protocol.export(invoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```
>dubbo工作原理：doExportUrlsFor1Protocol方
法，先创建两个URL，分别如下`dubbo://192.168.xx.25:20888/XXX;`,`registry://192.168.XXX;`(注册中心看到的services的providers信息)
>>比较核心的抽象：Invoker，Invoker是一个代理类，从ProxyFactory中生成。  
- 小结
>1. Invoker - 执行具体远程调用；  
2. Protocol – 服务地址的发布和订阅；  
3. Exporter – 暴露服务或取消暴露；  
>>protocol.export(invoker)，protocol 这个地方，并不是直接调用DubboProtocol协议的 export。具体实例化代码如下：

```java 
private static final Protocol protocol = ExtensionLoader.
getExtensionLoader(Protocol.class).
getAdaptiveExtension(); //Protocol$Adaptive
```
>实际，该Protocol得到一个Protocol$Adaptive（自适应适配器）。此时通过protocol.export(invoker),实际上调用的是Protocol$Adaptive动态类的 export方法。具体代码参考上面。

#### 调用ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
>该段代码通过工厂模式获得一个ExtensionLoader 实例。
##### getExtension
>该方法主要作用：用来获取ExtensionLoader实例代表的扩展指定实现。已扩展实现的名字作为参数，结合前面学习getAdaptiveExtension
的代码.

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    //判断是否已经缓存过该扩展点
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //createExtension ，创建扩展点
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```
>createExtension主要做4个事情:  
1.根据name获取对应class;  
2.根据class创建一个实例；  
3.对获取实例进行依赖注入；  
4.对实例进行包装，分别调用带Protocol参数的构造函数创建实例，然后进行依赖注入。  
>a)dubbo-rpc-api/resources/com.alibaba.dubbo.rcp.Protocol文件中有存在filter/listener；  
b)遍历cachedWrapperClass对DubboProtocol进行包装，会通过ProtocolFilterWrapper、ProtocolListenerWrapper包装。  

```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //对获取的实例进行依赖注入
        injectExtension(instance);
        //cachedWrapperClasses在loadFile中赋值
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                // 对实例进行包装，分别调用带Protocol参数的构造函数创建实例，然后进行依赖注入。
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```
##### getExtensionClasses
>该方法就是加载扩展点实现类。然后调用loadExtensionClasses，去对应文件下加载指定的扩展点。
```java 
// com.alibaba.dubbo.common.extension.ExtensionLoader
private Map<String, Class<?>> getExtensionClasses() {
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
```
##### 总结
➢ EExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName); 当extName 为registry时，不需再次阅读这块代码了，直接可在扩展点中找到相应的实现扩展点:
`[/dubbo-registryapi/src/main/resources/METAINF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol]` 配置如下:`registry=com.alibaba.dubbo.registry.integration.RegistryProtocol`，故可定位到RegistryProtocol类的export方法。

#### export

```java 
// com/alibaba/dubbo/registry/integration/RegistryProtocol
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker 本地发布服务（启动 netty）
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl);
    // 订阅override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //保证每次export都返回一个新的exporter实例
    return new Exporter<T>() {
        public Invoker<T> getInvoker() {
            return exporter.getInvoker();
        }

        public void unexport() {
            try {
                exporter.unexport();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                registry.unregister(registedProviderUrl);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                overrideListeners.remove(overrideSubscribeUrl);
                registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    };
}
```
##### doLocalExport
- 本地先启动监听服务
```java 
// com.alibaba.dubbo.registry.integration.RegistryProtocol
private <T> ExporterChangeableWrapper<T> doLocalExport(
        final Invoker<T> originInvoker) {
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker,
                        getProviderUrl(originInvoker));
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete),
                        originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return (ExporterChangeableWrapper<T>) exporter;
}


public class RegistryProtocol implements Protocol {
    ......
    private Protocol protocol;
    public void setProtocol(Protocol protocol) {
        this.protocol = protocol;
    }
    ......
}
```
>protocol怎么赋值？是一个依赖注入的扩展点。在加载扩展点时，有一个injectExtension方法，针对已加载的扩展点中的扩展点属性进行依赖注入。

- PROTOCOL.EXPORT
>protocol是一个自适应扩展点，Protocol$Adaptive，然后调用这个自适应扩展点中的export方法，这时传入的协议地址应该是dubbo://127.0.0.1/xxxx… 因此在Protocol$Adaptive.export方法中，ExtensionLoader.getExtension(Protocol.class).getExtension应该就是基于DubboProtocol协议去发布服务了？这里并不是获得一个单纯的DubboProtocol扩展点，而是会通过Wrapper对Protocol进行装 饰，装饰器分别为: ProtocolFilterWrapper/ProtocolListenerWrapper; 至于MockProtocol为何不在装饰器里？ExtensionLoader.loadFile有一个判断，装饰器必须要具备一个带有Protocol的构造方法

```java 
// com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
public ProtocolFilterWrapper(Protocol protocol){
    if (protocol == null) {
        throw new IllegalArgumentException("protocol == null");
    }
    this.protocol = protocol;
}
```
>Protocol$Adaptive里的export方法，会调用 ProtocolFilterWrapper、ProtocolListenerWrapper类的方法，这两个装饰器是干嘛的？ProtocolFilterWrapper类非常重要，dubbo机制里日志记录、超时等功能都在这一部分实现，该类有3个特点，
①它有一个参数为Protocol protocol的构造函数；  
②它实现了Protocol接口；  
③它使用责任链模式，对export和refer函数进行了封装。

```java 
// com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        return protocol.export(invoker);
    }
    return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
}

/**buildInvokerChain 函数：它读取所有的 filter 类，利用这些类封装invoker**/
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    //自动激活扩展点，根据条件获取当前扩展可自动激活的实现
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (filters.size() > 0) {
        for (int i = filters.size() - 1; i >= 0; i --) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                public URL getUrl() {
                    return invoker.getUrl();
                }

                public boolean isAvailable() {
                    return invoker.isAvailable();
                }

                public Result invoke(Invocation invocation) throws RpcException {
                    return filter.invoke(next, invocation);
                }

                public void destroy() {
                    invoker.destroy();
                }

                @Override
                public String toString() {
                    return invoker.toString();
                }
            };
        }
    }
    return last;
}
```
>`/dubbo-rpc-api/src/main/resources/METAINF/dubbo/internal/com.alibaba.dubbo.rpc.Filter`就是对Invoker，通过如下的Filter组装成一个责任链。
```xml 
echo=com.alibaba.dubbo.rpc.filter.EchoFilter
generic=com.alibaba.dubbo.rpc.filter.GenericFilter
genericimpl=com.alibaba.dubbo.rpc.filter.GenericImplFilter
token=com.alibaba.dubbo.rpc.filter.TokenFilter
accesslog=com.alibaba.dubbo.rpc.filter.AccessLogFilter
activelimit=com.alibaba.dubbo.rpc.filter.ActiveLimitFilter
classloader=com.alibaba.dubbo.rpc.filter.ClassLoaderFilter
context=com.alibaba.dubbo.rpc.filter.ContextFilter
consumercontext=com.alibaba.dubbo.rpc.filter.ConsumerContextFilt
er
exception=com.alibaba.dubbo.rpc.filter.ExceptionFilter
executelimit=com.alibaba.dubbo.rpc.filter.ExecuteLimitFilter
deprecated=com.alibaba.dubbo.rpc.filter.DeprecatedFilter
compatible=com.alibaba.dubbo.rpc.filter.CompatibleFilter
timeout=com.alibaba.dubbo.rpc.filter.TimeoutFilter
```
>这其中涉及到很多功能：权限验证、异常、超时等，当然可预计计算调用时间等应该也是在这其中的某个类实现的；这里可看到export和refer过程都会被filter过滤。

#### ProtocolListenerWrapper
>export和refer分别对应不同的Wrapper；export
是对应的ListenerExporterWrapper。这块暂时不分析。

```java 
// com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        return protocol.export(invoker);
    }
    return new ListenerExporterWrapper<T>(protocol.export(invoker), 
            Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                    .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
}

public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
        return protocol.refer(type, url);
    }
    return new ListenerInvokerWrapper<T>(protocol.refer(type, url), 
            Collections.unmodifiableList(
                    ExtensionLoader.getExtensionLoader(InvokerListener.class)
                    .getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)));
}
```




