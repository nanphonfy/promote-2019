#### RPC
>Remote Procedure Call——远程过程调用。实现不同机器系统间的方法调用，使程序像访问本地系统一样，通过网络传输访问远端系统资源。对于客户端， 传输层用啥协议，序列化、反序列化都是透明的。

- Java RMI
>remote method invocation——远程方法调用，是应用程序编程接口，网络分布式应用的核心解决方案之一。  
其使用JRMP （Java Remote Messageing Protocol——Java 远程消息交换协议）进行通信。由于JRMP专为Java对象制定的，不能与非JAVA对象通信。

- Java RMI代码实践
>远程对象必须实现UnicastRemoteObject，才能保证客户端访问获得远程对象时，该远程对象把自身拷贝以 Socket传给client，client获得的拷贝称为stub，而server已存在的远程对象称为skeleton。   
stub：client的代理，用于server通信；  
skeleton：server的代理，用于接收client请求后调用远程方法响应客户端的请求。

#### 发布远程对象
>如上，此处会发布两个远程对象：①RegistryImpl；②UserServiceImpl；  
UserServiceImpl的构造函数调用父类UnicastRemoteObject的构造方法，追溯到UnicastRemoteObject的私有方法exportObject()。  
此时判 断服务的实现是否为UnicastRemoteObject的子类。若是，则直接赋值其ref（RemoteRef）对象为传入的 UnicastServerRef对象。反之，则调用UnicastServerRef的exportObject()方法。因UserServiceImpl继承UnicastRemoteObject，故服务启动时，会通过UnicastRemoteObject的构造方法发布该对象。