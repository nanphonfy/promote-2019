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

案例：
