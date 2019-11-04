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
spring提供的扩展类如下：NamespaceHandler、BeanDefinitionParse  

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
DubboBeanDefinitionParser（解析spring配置文件）->ServiceBean

```java 
// com.alibaba.dubbo.config.spring.ServiceBean
public void afterPropertiesSet() throws Exception {
    ......
    // 非常关键的，进入服务发布的过程
    if (! isDelay()) {
        export();
    }
}
```
>启动一个服务时做了啥（调用注册中心发布服务到zookeeper、启动一个netty 服务）
