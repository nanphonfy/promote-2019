### 架构发展带来的问题
>- ①当服务越来越多时，服务URL配置管理变得很困难，F5硬件负载均衡器单点压力也越来越大。此时需要一个服务注册中心，动态注册、发现服务，使
服务位置透明。并通过在消费方获取服务提供方地址列表，实现软负载均衡和Failover，降低对F5硬件负载均衡器依赖，也减少部分成本；  
>- ②当进一步发展，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在谁之前启动。这时，需自动画出应用间的依赖关系图，帮助架构师理清关系；  
>- ③服务调用量越来越大，暴露服务的容量问题：该服务需多少机器支撑？该加几台机器？  
解决：第一步、统计服务每天调用量、响应时间（容量规划参考指标）；第二步，要可动态调整权重，在线上，将某机器权重加大，该过程中记录响应时间的变化，直到响应时间到达阀值，记录此时访问量，再以此访问量乘以机器数反推总容量。

#### 多版本支持
>设置不同版本的目的：接口升级后的兼容问题。Dubbo中配置不同版本的接口，会在Zookeeper地址中有多个协议url的体现：

```java 
dubbo://192.168.11.1:20880%2Fcom.gupaoedu.dubbo.IGpHello%3Fanyhost%3Dtrue%26application%3Dhello-world-app%26dubbo%3D2.5.6%26generic%3Dfalse%26interface%3Dcom.gupaoedu.dubbo.IGpHello%26methods%3DsayHello%26pid%3D60700%26revision%3D1.0.0%26side%3Dprovider%26timestamp%3D1529498478644%26version%3D1.0.0

dubbo://192.168.11.1%3A20880%2Fcom.gupaoedu.dubbo.IGpHello2%3Fanyhost%3Dtrue%26application%3Dhello-world-app%26dubbo%3D2.5.6%26generic%3Dfalse%26interface%3Dcom.gupaoedu.dubbo.IGpHello%26methods%3DsayHello%26pid%3D60700%26revision%3D1.0.1%26side%3Dprovider%26timestamp%3D1529498488747%26version%3D1.0.1
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
#### 服务降级
>目的：保证核心服务可用。  
降级有几个层面分类：自动降级和人工降级；   按功能分：读服务降级和写服务降级；  
>- 1.对一些非核心服务做人工降级，大促前通过降级开关关闭哪些推荐内容、评价等对主流程无影响的功能；  
>- 2.故障降级，eg.调用远程服务挂了、网络故障、或RPC服务异常。可直接降级，方案eg.设置默认值、采用兜底数据（推荐行为的广告挂了，可提前准备静态页面返回）等；  
>- 3.限流降级，在秒杀中流量较集中且流量特大的情况，突发访问量特大可能会导致系统支撑不了。此时可采用限流限制访问量。当达到阀值时，后续请求被降级，eg.进入排队页面、跳转到错误页。

