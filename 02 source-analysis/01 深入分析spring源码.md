### 1 简介

```JAVA 
class A{}
//实例化后，用变量保存（匿名对象）
A a = new A();

//spring初始化，实例化（控制权）
@Autoried
A a;

//一定要初始化，否则报null
a.execute();
```

>IOC容器（存java bean）  
容器用来装东西，eg.装水容器（水桶、杯子）  
web容器，用来装servlet
spring控制反转后，最终目的：实现依赖注入。


```java 
//依赖链中所有对象，都在IOC容器里初始化
//实例化先后顺序：b a c
class C{
    private A a;
    public void func(){
      a.xxx();
    }
}
class A{
    private B b;
}
```

- spring注入方式
>setter  
构造方法  
强制赋值

>面向Bean编程：bean oriented programming BOP  
依赖注入：dependency injection DI  
IOC

>如果两个模块间不能满足一定规则（切面就是规则），那就是说这两个模块无法合并组装在一起的。

- AOP（解耦）核心思想：
>①先要把一个整体给拆了，分别开发；  
②等到发布时，再组装到一起运行。

>一个切面就有一个规则。AOP的重点是解耦（面向规则）。

- 案例
>①苹果A、B，各切成两半，A的左边无法和B的右边无缝合并；  
②螺丝、螺帽，二者丝纹紧紧咬合才能无缝合并；  
③小汽车的轮子大小30，拿大小40的轮子，无法装进去；  
④事务（开启一个事务、执行提交事务、事务回滚、关闭事务），这种有规律的东西，可认为它是一个固定的规则，这时可单独具有一定规律的规则单独分离出来，做为一个独立模块。
>>面向切面就是面向规则编程。

- spring用到AOP的地方：
>authentication权限认证  
logging日志  
transctions manager事务  
lazy loading懒加载  
context process上下文处理  
error handler错误跟踪（异常捕获机制）  
cache缓存  

di  
IOC  
AOP（核心宗旨：解耦）  
spring的核心宗旨：简化开发  

---
#### 1.1 9种常用设计模式
>- 代理模式：  
①事情必须做，自己没时间做或不想做；  
②持有被代理人的引用。  
>>静态、动态代理。  
JDK动态代理：  
CGLib动态代理：cglib.jar（Code generation library代码生成库）  
asm.jar（全称assembly，装配）

>- 工厂模式：  
1、隐藏复杂的逻辑过程，只关心结果。  
>>简单工厂、工厂方法、抽象工厂

>- 单例模式：

>- 委派模式：  
①类似中介（委托机制）；  
②只有被委托人引用。  
>>要和代理模式区分开。

>- 策略模式：  
过程不同，结果一样。

>- 原型模式：  
过程相同，结果不一样。或模板模式（提高开发效率）

`源码学习顺序：spring-core、spring-beans、spring-aop、spring-context、spring-tx、spring-orm、spring-web及其他部分。`

---
### 2 设计模式  
#### 2.1 代理模式  
>代理模式：租房中介、火车票黄牛、媒人、经纪人  
特点：  
①执行者、被代理人；  
②被代理人事情必须做，自己没时间做或不想做；  
③需获取到被代理人的个人资料。
>>穷举法  
代理模式关心过程，而不是结果。

`"动态代理至少写了50遍。彻底了解，必须反复重复，每次重复会发现一些新问题"。`

>- 总结：代理人模式最底层->做了一件什么事情呢？字节码重组。
>>在原始代码加一些东西，编译生成字节码，加载到JVM动态运行。  
spring用的比较多的是cglib代理模式。  
jdk动态代理是通过接口，而cglib是通过继承。eg.Son类继承Father，该类是cglib给我们自动生成的。  
cglib同样做了字节码重组。这样做的好处：少写几个类、几个接口。
对于使用API的用户来说是无感知的。  

>每种技术都是有应用场景的，eg.面向接口编程一般对外提供给别人调用，形成一种规范。  
AOP搞接口，其实是搞复杂了。  
//AOP 解耦(团队开发)  
//变相：三层架构（解耦）

>代理模式可以做一件什么事情？可在每个方法调用前加一些代码，在方法调用后加一些代码。  
AOP:事务代理（声明式事务，哪个方法需要加事务，哪个方法不需要加事务）、日志监听  
```JAVA 
service方法  
开启一个事务(open)  

事务的执行(由我们自己的代码完成的)

监听是否有异常，可能需要根据异常类型来决定这个事务是否要回滚，还是继续提交（commit/rollback）。  
事务要关闭（close）  
```
>spring为什么是优秀框架，它把停留在理论层次的方法论落地了，而且告诉大家怎么用。

#### 1.1 JDK动态代理
- 类接口
```java 
public interface Person {
    String getAddress();
    String getPrice();
    /**
     * 租房
     */
    void findRoom();
}
```
- 想找租房中介的小明-被代理者
```java 
/**
 * @author nanphonfy(南风zsr)
 * @date 2019/6/30
 */
public class XiaoMing implements Person {
    private String address = "南山区蛇口";
    private String price = "2800";

    @Override public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    @Override public String getPrice() {
        return price;
    }

    public void setPrice(String price) {
        this.price = price;
    }

    @Override public void findRoom() {
        System.out.println(String.format("我想在%s租价位大概%s左右的房子", this.address, this.price));
        System.out.println("考虑单身公寓或合租主卧");
        System.out.println("带独卫和私人阳台");
        System.out.println("有电梯");
    }
}
```
- 房屋中介-代理者
```java 
/**
 * 租房中介
 *
 * @author nanphonfy(南风zsr)
 * @date 2019/6/30
 */
public class RentalAgent implements InvocationHandler {
    /**
     * 被代理对象的引用作为一个成员变量保存下来
     **/
    private Person target;

    public Object getInstance(Person target) {
        this.target = target;
        Class clazz = target.getClass();
        System.out.println("被代理对象的class是:" + clazz);
        return Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), this);
    }

    @Override public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("我是房屋中介，帮忙挑选心仪房源，赚中介费，实现财富自由赢取白富美");
        System.out.println("开始进行筛选...");
        System.out.println("----------------");

        method.invoke(this.target, args);

        System.out.println("----------------");
        System.out.println("若合适，带你看房");
        return null;
    }
}
```

- 执行类
```java 
/**
 * @author nanphonfy(南风zsr)
 * @date 2019/6/30
 */
public class JdkFindRoomTest {
    public static void main(String[] args) {
        try {
            Person obj = (Person) new RentalAgent().getInstance(new XiaoMing());
            System.out.println("obj class:" + obj.getClass());
            obj.findRoom();
            obj.getAddress();

            /*原理：
            1.拿到被代理对象的引用，获取它的接口
            2.JDK代理重新生成一个类，同时实现代理对象所实现的接口
            3.重新动态生成一个class字节码
            4.编译*/

            //获取字节码内容
            byte[] data = ProxyGenerator.generateProxyClass("$Proxy0", new Class[] { Person.class });
            FileOutputStream os = new FileOutputStream("$Proxy0.class");
            os.write(data);
            os.close();

            //是什么?
            //为什么？
            //怎么做？
            // 自定义的类JDK代理
            /*Person obj = (Person) new NRentalAgent().getInstance(new XiaoMing());
            System.out.println("obj class:" + obj.getClass());
            obj.findRoom();*/
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
#### 1.2 CgLib动态代理
- 小by2想找中介（未实现接口）-被代理者
```java 
/**
 * @author nanphonfy(南风zsr)
 * @date 2019/6/30
 */
public class XiaoBy2 {
    public void findRoom(){
        System.out.println("我要找环境宜人，安静舒适干净的房子");
    }
}
```
- cglib代理小by2的要求-代理者
```java 
/**
 * @author nanphonfy(南风zsr)
 * @date 2019/6/30
 */
public class CgRentalAgent implements MethodInterceptor {
    /**
     * 疑问？
     * 好像没有持有被代理对象的引用
     */
    public Object getInstance(Class clazz) throws Exception{
        // 生成class类的路径
        // https://blog.csdn.net/u010811939/article/details/80763336
        // JDK动态代理生成class文件和cglib动态代理生成class文件
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "E://tmp");

        Enhancer enhancer = new Enhancer();
        //把父类设置为谁？
        //这一步告诉cglib，生成的子类需要继承哪个类
        enhancer.setSuperclass(clazz);
        //设置回调(回调intercept，所以设置this)
        enhancer.setCallback(this);

        //第一步、生成源代码
        //第二步、编译成class文件
        //第三步、加载到JVM中，并返回被代理对象
        return enhancer.create();
    }

    /***
     * 同样做了字节码重组
     * 对于使用API的用户来说，是无感知的
     * @param obj
     * @param method
     * @param objects
     * @param methodProxy
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Object obj, Method method, Object[] objects, MethodProxy methodProxy)
            throws Throwable {
        System.out.println("我是房屋中介，帮忙挑选心仪房源，赚中介费，实现财富自由赢取白富美");
        System.out.println("开始进行筛选...");
        System.out.println("----------------");

        /*该obj的引用是由CGLib new出来的
        cglib new出来后的对象，是被代理对象的子类（继承了我们自己写的那个类）
        OOP, 在new子类前，实际上默认先调用了我们super()方法，
        new了子类的同时，必须先new出父类，相当于间接的持有了我们父类的引用
        子类重写了父类的所有的方法
        改变子类对象的某些属性，可以间接操作父类的属性*/
        methodProxy.invokeSuper(obj,objects);

        System.out.println("----------------");
        System.out.println("若合适，带你看房");
        return null;
    }
}
```
- 执行类

```java 
/**
 * @author nanphonfy(南风zsr)
 * @date 2019/6/30
 */
public class CglibFindRoomTest {
    public static void main(String[] args) {
        try {
            /*JDK的动态代理是通过接口来进行强制转换的
            生成后的代理对象，可强制转换为接口*/

            /*CGLib的动态代理是通过生成一个被代理对象的子类，然后重写父类的方法
            生成后的对象，可强制转换为被代理对象（也就是用自己写的类）
            子类引用赋值给父类*/
            XiaoBy2 xiaoBy2 = (XiaoBy2) new CgRentalAgent().getInstance(XiaoBy2.class);
            xiaoBy2.findRoom();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

-----------------------------------------------------
#### 2.2 工厂模式
>工厂模式不关心过程，只关心结果（结果论）。  
过程需隐藏起来。  

>生产者跟消费者要区分开。    
买早餐：面包、牛奶（产品）  
消费者：关心产品保质期、使用效果、价格。不关心面包生产工艺、生产材料。
>>eg.QS(国家卫生组织权威认证)  
ISO9002(国际标准)

>所有的设计模式都来源于生活。  
AOP在现实生活中使用很普遍，解耦，eg.现代工作分工越来越细，专人做专事。  
切面：eg.建房子，找专业的团队；我需要这个东西，你帮我做。  
只要带有解耦功能，都可以认为是切面。切面的具体实现方式：代理。  
AOP是一种思想，可用代理实现。  
AOP 不等于 代理。工厂不算实现AOP。  

>学习思想，指导行为。  
在实际应用中，不会使用简单工厂模式。  
内功、心法，先把任督二脉打通。  
学习设计模式，是要学一个创造性思维，哪种场景适用哪个设计模式(穷举法)。

>如果没有工厂模式，我们写代码既要关心过程，又要关心结果，分工也比较难。把复杂的业务抽离出来。  

- 工厂模式特点：
>隐藏复杂的逻辑处理过程，只关心执行结果。

- 多个if（内部实现逻辑，可能有几千行），生产的产品多时，非常絮乱，维护困难。
```JAVA  
if("BMW".equalsIgnoreCase(name)){
	//几千行业务逻辑
	return new Bmw();
}else if("Benz".equalsIgnoreCase(name)){
    //几千行业务逻辑
	return new Benz();
}else if("Audi".equalsIgnoreCase(name)){
    //几千行业务逻辑
	return new Audi();
}else{
	System.out.println("这个产品产不出来");
	return null;
}
```
- 把几千行的逻辑都抽离到工厂类，相当于动态配置（固定模式的委派）
```JAVA  
public Car getCar(String name){
	if("BMW".equalsIgnoreCase(name)){
		return new BmwFactory().getCar();
	}else if("Benz".equalsIgnoreCase(name)){
		return new BenzFactory().getCar();
	}else if("Audi".equalsIgnoreCase(name)){
		return new AudiFactory().getCar();
	}else{
		System.out.println("这个产品产不出来");
		return null;
	}
}
```
---

#### 2.1 简单工厂模式 
- 基础类
```java 
/**
 * 产品接口：汽车需满足一定的标准
 * @author nanphonfy(南风zsr)
 * @date 2019/7/1
 */
public interface Computer {
    /** 获取电脑品牌 **/
    String getProduct();
}

public class Apple implements Computer {
    @Override public String getProduct() {
        return "Apple";
    }
}

......
```
- 简单工厂类

```java 
public class SimpleFactory {
    public static String APPLE = "Apple";
    public static String LENOVO = "Lenovo";
    public static String HUAWEI = "HuaWei";
    public Computer getCcmputer(String name){
		/*Spring中的工厂模式
        Bean
		BeanFactory（生成Bean）
		单例的Bean
		被代理过的Bean
		最原始的Bean（原型）
		List类型的Bean
		作用域不同的Bean*/

		/* getBean
		非常紊乱，维护困难
		解耦（松耦合开发）*/
        if(APPLE.equalsIgnoreCase(name)){
            // 可能有几千行逻辑
            return new Apple();
        }else if(LENOVO.equalsIgnoreCase(name)){
            // 可能有几千行逻辑
            return new Lenovo();
        }else if(HUAWEI.equalsIgnoreCase(name)){
            // 可能有几千行逻辑
            return new HuaWei();
        }else{
            System.out.println("工厂不支持生产该产品");
            return null;
        }
    }
}
```
#### 2.2 抽象工厂模式 
- 工厂抽象类
```java 
public abstract class AbstractFactory {
    protected abstract Computer getComputer();

    /**
     * 这段代码就是动态配置的功能
     * 固定模式的委派
     * @param name
     * @return
     */
    public Computer getComputer(String name){
        if(APPLE.equalsIgnoreCase(name)){
            return new AppleFactory().getComputer();
        }else if(LENOVO.equalsIgnoreCase(name)){
            return new LenovoFactory().getComputer();
        }else if(HUAWEI.equalsIgnoreCase(name)){
            return new HuaWeiFactory().getComputer();
        }else{
            System.out.println("工厂不支持生产该产品");
            return null;
        }
    }
}
```
- 继承抽象类的工厂
```java 
/**
 * 生产联想电脑的业务逻辑封装
 * @author nanphonfy(南风zsr)
 * @date 2019/7/1
 */
public class LenovoFactory extends AbstractFactory{
    @Override protected Computer getComputer() {
        return new Lenovo();
    }
}
......
public class DefaultFactory extends AbstractFactory{
    private AbstractFactory defaultFactory = new LenovoFactory();

    @Override protected Computer getComputer() {
        return defaultFactory.getComputer();
    }
}
```
- 执行类
```java 
public abstract class AbstractFactoryTest {
    public static void main(String[] args) {
        DefaultFactory defaultFactory = new DefaultFactory();
        System.out.println(defaultFactory.getComputer().getProduct());
        System.out.println(defaultFactory.getComputer("huawei").getProduct());
    }
}
```
#### 2.3 方法工厂模式 
- 工厂流程规范接口

```java 
/**
 * 工厂接口：定义了所有工厂的执行标准
 * @author nanphonfy(南风zsr)
 * @date 2019/7/1
 */
public interface Factory {
    Computer getComputer();
}
```
- 实现执行标准的工厂
```java 
public class HuaWeiFactory implements Factory{
    @Override public Computer getComputer() {
        return new HuaWei();
    }
}
......
```
- 执行类
```java 
public class FuncFactoryTest {
    public static void main(String[] args) {
        /*工厂方法模式，用户只需关心产品的生产商
		各个产品的生产商，都拥有各自的工厂。生产工艺、高科技程度都是不一样*/
        Factory factory = new AppleFactory();
        System.out.println(factory.getComputer().getProduct());

        factory = new LenovoFactory();
        System.out.println(factory.getComputer().getProduct());
    }
}
```
#### 2.3 单例模式
- 是什么？为什么要有单例模式？怎么实现？2WH what why how
>①保证系统启动到停止，全过程只会产生一个实例；  
②当在应用中遇到功能性冲突时，需要使用单例模式。

人都有天生的惰性

- 穷举法：
>配置文件：若不是单例（两配置文件内容一样，有一个是浪费的，若不一样，不知道以哪个为准）  
直接上级领导：（若有多个领导，听谁的？）  
每个个体（万千世界，两片叶子，不一样）  

- 单例模式的七种写法：  
??????????
#### 3.1 懒汉式单例
```java 
/**
 * 【写法1】懒汉式单例类.第一次调用时进行实例化
 */
public class LazySingleton1 {
    /** 1、将构造方法私有化 **/
    private LazySingleton1() {}
    /** 2、声明静态变量保存单例引用 **/
    private static LazySingleton1 single = null;
    /** 3、提供静态方法获得单例引用 (不安全的)**/
    public static LazySingleton1 getInstance() {
        if (single == null) {
            single = new LazySingleton1();
        }
        return single;
    }
}

/**
 * 【写法2】懒汉式单例类.保证线程安全
 */
public class LazySingleton2 {
    /** 1、将构造方法私有化 **/
    private LazySingleton2() {}
    /** 2、声明静态变量保存单例引用 **/
    private static LazySingleton2 single = null;
    /** 3、提供静态方法获得单例引用 **/
    /**为保证多线程下正确访问，给方法加同步锁synchronized，getInstance()始终单线程来访问
    (慎用  synchronized 关键字，阻塞，性能非常低下的，没充分利用计算机资源，资源浪费)**/
    public static synchronized LazySingleton2 getInstance() {
        if (single == null) {
            single = new LazySingleton2();
        }
        return single;
    }
}
/**
 * 【写法3】懒汉式单例.双重锁检查
 */
public class LazySingleton3 {
    /** 1、将构造方法私有化 **/
    private LazySingleton3() {}
    /** 2、声明静态变量保存单例引用 **/
    private static LazySingleton3 single = null;
    /** 3、提供静态方法获得单例引用 **/
    /**保证多线程下的另一种实现方式:双重锁检查，性能：第一次损耗**/
    public static LazySingleton3 getInstance() {
        if (single == null) {
            synchronized (LazySingleton3.class) {
                if (single == null) {
                    single = new LazySingleton3();
                }
            }
        }
        return single;
    }
}

/**
 * 【写法4】懒汉式单例.静态内部类
 */
public class LazySingleton4 {
    /*java反射机制下，所有代码都是裸奔的，可拿到private修饰的内容即使加上private也不靠谱*/
    /**1、声明静态内部类。private私有，tatic保证全局唯一**/
    private static class LazyHolder {
        /**final为防止内部误操作，eg.代理模式-GgLib**/
        private static final LazySingleton4 INSTANCE = new LazySingleton4();
    }
    /**2、将默认构造方法私有化**/
    private LazySingleton4 (){}
    /** 3、同样提供静态方法获取实例，final确保不能覆盖 **/
    public static final LazySingleton4 getInstance() {
        /*用户调用时才执行静态内部类，方法中实现逻辑调用时才分配内存*/
        return LazyHolder.INSTANCE;
    }
    /*	static int a = 1;
    // 不管该class有没有实例化，static静态块总会在classLoader执行完后加载完毕
    static{
        // 静态块中的内容，只能访问静态属性和静态方法
        // 只要是静态方法或者属性，直接可以用Class的名字就能点出来
        LazySingleton4.a = 2;
        // JVM 内存中的静态区，这一块的内容是公共的
    }*/
}

// 类装载到JVM中过程
// 从上往下(必须声明在前，使用在后)
// 先属性、后方法
// 先静态、后动态

/**
 * 【写法5-volatile】懒汉式单例.双重锁检查
 */
public class VolatileLazySingleton3 {
    /** 1、将构造方法私有化 **/
    private VolatileLazySingleton3() {}
    /** 2、声明静态变量保存单例引用【volatile关键字】 **/
    private static volatile VolatileLazySingleton3 single = null;
    /** 3、提供静态方法获得单例引用 **/
    /**保证多线程下的另一种实现方式:双重锁检查，性能：第一次损耗**/
    public static VolatileLazySingleton3 getInstance() {
        if (single == null) {
            synchronized (VolatileLazySingleton3.class) {
                if (single == null) {
                    single = new VolatileLazySingleton3();
                }
            }
        }
        return single;
    }
}
```
#### 3.2 饿汉式单例
```java 
/**
 * 【写法6】饿汉式：类创建时已创建静态的对象不再改变，故天生线程安全
 */
public class HungerSingleton1 {
    public static HungerSingleton1 INSTANCE = new HungerSingleton1();
    protected HungerSingleton1() {  }
    private Object readResolve() {
        return INSTANCE;
    }
}
```
#### 3.3 登记式单例
```java 
/**
 * 【写法7】类似Spring：将类名注册，下次从MAP获取
 */
public class RegisterSingleton1 {
    private static Map<String,RegisterSingleton1> map = new HashMap<String,RegisterSingleton1>();
    static {
        RegisterSingleton1 single = new RegisterSingleton1();
        map.put(single.getClass().getName(), single);
    }
    /**保护的默认构造器**/
    protected RegisterSingleton1(){}
    /**静态工厂方法：返还此类惟一实例**/
    public static RegisterSingleton1 getInstance(String name) {
        if(name == null) {
            name = RegisterSingleton1.class.getName();
        }
        if(map.get(name) == null) {
            try {
                map.put(name, (RegisterSingleton1) Class.forName(name).newInstance());
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
        return map.get(name);
    }
}
```
---
#### 2.4 委派模式
>代理模式案例，租房中介和你，是平等关系，委派模式不一样，但是他们的关系是包含或上下级关系。  
要和代理模式区分开  
2WH  
what：两个角色，受托人，委托人（社会上是平等关系），eg.公司里面项目经理和普通员工（法律上平等，工作的关系，各自的职责不一样）
代理模式，代理过程都是清楚的，委托模式，委派模式不关心过程只关心结果。
>>项目经理（委托人）：安排任务  
普通员工（受托人）：执行任务

- 特点：  
>①类似于中介功能（委托机制）；  
②持有被委托人的引用；  
③不关心过程，只关心结果。

>围绕spring来讲，spring使用最多的，来讲。目的是为了能看懂源码.  
why：主要目的就是隐藏具体实现逻辑（类似工厂模式，但是实现原理不一样）
how：  
IOC容器中，有一个Register的东西（为了告诉容器，在这个类被初始化过程中，需要做很多不同的逻辑处理，需要实现多个任务执行者，分别实现各自的功能）  
>>工厂模式保证结果的多样性，对于用户只是一种方法。

>直接看代码，很吃力，容易晕。  
设计模式一定要思考，想明白了，才会使用。  

>①代理模式前后要做增加，委派模式不做任何增加（指定谁处理）；  
②代理模式关心过程，委派模式关心结果；  

>工厂模式和委派模式不一样，工厂制定的是产品，委派制定的是方法（或执行者）。

- 案例
```java 
public interface IExector {
    /**员工干活**/
    void doing();
}

/**
 * 程序员
 */
public class Programmer implements IExector{
    @Override public void doing() {
        System.out.println("开发智能保顾里程碑需求...");
    }
}
......

/**
 * 项目经理
 */
public class PmDispatcher implements IExector{
    private IExector iExector;

    public PmDispatcher(IExector iExector) {
        this.iExector = iExector;
    }

    @Override public void doing() {
        iExector.doing();
    }
}

public class PmDispatcherTest{
    public static void main(String[] args) {
        // 表面上项目经理在干活，实际干活的是程序员——干活是我，功劳是你
        PmDispatcher dispatcher = new PmDispatcher(new Programmer());
        dispatcher.doing();
    }
}
```

---
#### 2.5 策略模式
>过程不同，但结果一样。

- 案例
>人和目的地：地图导航输入起点、目的地（结果不变），会有多种路线供选择（驾车、公交、步行、骑行->策略）。
>>策略模式非常简单。

- 案例
```java 
/**
 * 比较器
 */
public interface Comparator {
    int compareTo(Object obj1,Object obj2);
}

public class NumberComparator implements Comparator{
    @Override public int compareTo(Object obj1, Object obj2) {
        // 执行逻辑
        return 0;
    }
}
......
public class StrategyTest {
    public static void main(String[] args) {
        // 自定义的策略模式
        // new MyList().sort(new NumberComparator());

        // JDK的策略模式
        List<Long> numbers = new ArrayList<Long>();
        Collections.sort(numbers, new java.util.Comparator<Long>() {
            @Override
            // 返回值是固定的
            // 0 、-1 、1
            // 0 、 >0 、<0
            public int compare(Long o1, Long o2) {
                // 中间逻辑是不一样的
                return 0;
            }
        });
    }
}
```

---
#### 2.6 原型模式
>先有个对象-原型，拷贝出来不是同一个，但内容一样。  
过程相同，结果不一样；数据内容完全一样。
java的克隆，就是原型模式的实现。  
原型模式主要是解决数据的复制问题（成员变量如果很多，要写很多垃圾代码）。  
克隆是不走构造方法的。

9种模式  
spring-jdbc用模板模式  

#### 2.7 模板模式
- 模板模式：执行流程一样，但中间有些步骤不同。

- 案例 
>喝茶过程比较麻烦，机器就产生了，饮料冲泡机。  
1、烧开水；  
2、准备杯子，把原料放入杯子（人工干预）；  
3、用水冲泡；  
4、添加辅料（人工干预）。  
固定的执行流程就叫模板。

>Spring的JDBC是java规范，各个数据库厂商自己去实现
1、加载驱动类DriverManager；  
2、建立连接；  
3、创建语句集(标准语句集、预处理语句集)(语句集？  MySQL、Oracle、SQLServer、Access)；
4、执行语句集；
5、结果集ResultSet 游标。
ORM(?)
- 目的不是学会用，而是学会怎么去思考。


```java 
// 饮料机
public abstract class Bevegrage {
    // 不能被重写
    public final void create() {
        // 1、把水烧开
        boilWater();
        // 2、把杯子准备好、原材料放杯中
        pourInCup();
        // 3、用水冲泡
        brew();
        // 4、添加辅料
        addCoundiments();
    }

    public abstract void pourInCup();

    public abstract void addCoundiments();

    public void brew() {
        System.out.println("将开水放入杯中进行冲泡");
    }

    public void boilWater() {
        System.out.println("烧开水，烧到100度可以起锅了");
    }
}

public class Coffee  extends Bevegrage{
	// 原材料放入杯中
	@Override
	public void pourInCup() {
		System.out.println("将咖啡倒入杯中");
	}

	// 放辅料
	@Override
	public void addCoundiments() {
		System.out.println("添加牛奶和糖");
	}
}
......
public class TemplateTest {
	public static void main(String[] args) {
		Coffee coffee = new Coffee();
		coffee.create();
		Tea tea = new Tea();
		tea.create();
	}
}
```

---
### 3 总结
>从生活中，自己找例子，写9个设计模式。加深设计模式理解。  
 策略模式在spring用的比较多的是回调处理，SpringJDBC RowMap。

>面向切面编程  
面向对象编程  
面向Bean编程  
控制反转  
依赖注入/依赖查找  

>IOC用来new和存， di用来赋值。  
IOC是DI和DL的基础。  
design patterns  

- 原型模式案例  
>ORM记录行转换成一个java对象；非常复杂的java之间的数据备份。

>只要实现了序列化，就可实现数据的复制。不同类，属性相同也可实现数据拷贝。
不走构造方法，直接走字节流。

>JDBC template 模板模式  
ORM rowMapper 策略模式  
Transactions 代理模式  
AOP 代理模式  
websocket ws  
servlet http  
web mvc  
portlet 页面  

AOP 切面 cglib  
aop config  
beans BOP   
core 核心实现逻辑  