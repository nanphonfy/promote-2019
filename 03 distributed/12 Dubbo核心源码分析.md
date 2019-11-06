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

