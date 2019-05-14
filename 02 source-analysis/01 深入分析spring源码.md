class A{

}

//实例化后，用变量保存（匿名对象）
A a = new A();

//spring初始化，实例化（控制权）
@Autoried
A a;

//一定要初始化，否则报null
a.execute();

IOC容器（存java bean）
容器用来装东西，eg.装水容器（水桶、杯子）
web容器，用来装servlet

spring控制反转后，最终目的：实现依赖注入。

//依赖链中所有对象，都在IOC容器里初始化
实例化先后顺序：b a c
class C{
    private A a;
    public void func(){
      a.xxx();
    }
}
class A{
    private B b;
}

spring注入方式
setter
构造方法
强制赋值


面向Bean编程：bean oriented programming BOP
依赖注入：dependency injection DI
IOC

如果两个模块间不能满足一定规则（切面就是规则），那就是说这两个模块无法合并组装在一起的。

AOP（解耦）核心思想：
①先要把一个整体给拆了，分别开发；
②等到发布时，再组装到一起运行。

一个切面就有一个规则。
AOP的重点是解耦（面向规则）。

①苹果A、B，各切成两半，A的左边无法和B的右边无缝合并；
②螺丝、螺帽，二者丝纹紧紧咬合才能无缝合并；
③小汽车的轮子大小30，拿大小40的轮子，无法装进去；

④事务（开启一个事务、执行提交事务、事务回滚、关闭事务），这种有规律的东西，可认为它是一个固定的规则，这时可单独具有一定规律的规则单独分离出来，做为一个独立模块。

面向切面就是面向规则编程。