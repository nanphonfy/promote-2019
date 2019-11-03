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


