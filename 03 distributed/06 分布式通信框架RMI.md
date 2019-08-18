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

```java 
// cn.nanphonfy.mi.server.UserServiceImpl
public class UserServiceImpl extends UnicastRemoteObject implements UserService{
    protected UserServiceImpl() throws RemoteException {
        super();
    }

    @Override
    public String sayWord(String msg) throws RemoteException {
        return "server:"+msg;
    }
}

public static final String BINDING_ADDRESS = "rmi://127.0.0.1/sayWord";
public static void main(String[] args) {
    try {
        UserService userService = new UserServiceImpl();
        LocateRegistry.createRegistry(1099);
        Naming.rebind(BINDING_ADDRESS,userService);
        System.out.println("Server started...");
    } catch (RemoteException e) {
        e.printStackTrace();
    } catch (MalformedURLException e) {
        e.printStackTrace();
    }
}
```

>此时判断服务的实现是否为UnicastRemoteObject的子类。若是，则直接赋值其ref->RemoteRef对象为传入的UnicastServerRef对象。反之，则调用UnicastServerRef的exportObject()方法。因UserServiceImpl继承UnicastRemoteObject，故服务启动时，会通过UnicastRemoteObject的构造方法发布该对象。


```java 
// java.rmi.server.UnicastRemoteObject
protected UnicastRemoteObject(int port) throws RemoteException{
    this.port = port;
    exportObject((Remote) this, port);
}

public static Remote exportObject(Remote obj, int port) throws RemoteException{
    // 创建UnicastServerRef对象，对象内有引用了LiveRef(tcp通信)
    return exportObject(obj, new UnicastServerRef(port));
}

private static Remote exportObject(Remote obj, UnicastServerRef sref) throws RemoteException{
    // if obj extends UnicastRemoteObject, set its ref.
    if (obj instanceof UnicastRemoteObject) {
        // java.rmi.server.RemoteObject(transient protected RemoteRef ref)
        ((UnicastRemoteObject) obj).ref = sref;
    }
    // sun.rmi.server.UnicastServerRef
    return sref.exportObject(obj, null, false);
}
```



```java 
// sun.rmi.server.UnicastServerRef
public Remote exportObject(Remote var1, Object var2, boolean var3) throws RemoteException {
    Class var4 = var1.getClass();

    Remote var5;
    try {
        // 创建远程代理类，该代理是对OperationImpl对象的代理，getClientRef提供的InvocationHandler中提供了TCP连接
        var5 = Util.createProxy(var4, this.getClientRef(), this.forceStubUse);
    } catch (IllegalArgumentException var7) {
        throw new ExportException("remote object implements illegal remote interface", var7);
    }

    if(var5 instanceof RemoteStub) {
        this.setSkeleton(var1);
    }

    // 包装实际对象，将其暴露在TCP端口上，等待客户端调用
    Target var6 = new Target(var1, this, var5, this.ref.getObjID(), var3);
    this.ref.exportObject(var6);
    this.hashToMethod_Map = (Map)hashToMethod_Maps.get(var4);
    return var5;
}
```
>服务端启动Registry服务，创建RegistryImpl对象。若服务端指定端口：1099且开启安全管理器， 则可绕过系统安全校验。真正的逻辑在this.setup(new UnicastServerRef())。

```java 
// cn.nanphonfy.mi.server.Server
LocateRegistry.createRegistry(1099);

// java.rmi.registry.LocateRegistry
public static Registry createRegistry(int port) throws RemoteException {
    return new RegistryImpl(port);
}

// sun.rmi.registry.RegistryImpl
public RegistryImpl(final int var1) throws RemoteException {
    if(var1 == 1099 && System.getSecurityManager() != null) {
        try {
            AccessController.doPrivileged(new PrivilegedExceptionAction() {
                public Void run() throws RemoteException {
                    LiveRef var1x = new LiveRef(RegistryImpl.id, var1);
                    RegistryImpl.this.setup(new UnicastServerRef(var1x));
                    return null;
                }
            }, (AccessControlContext)null, new Permission[]{new SocketPermission("localhost:" + var1, "listen,accept")});
        } catch (PrivilegedActionException var3) {
            throw (RemoteException)var3.getException();
        }
    } else {
        LiveRef var2 = new LiveRef(id, var1);
        this.setup(new UnicastServerRef(var2));
    }
}

private void setup(UnicastServerRef var1) throws RemoteException {
    this.ref = var1;
    var1.exportObject(this, (Object)null, true);
}
```
>setup方法将指向正在初始化的RegistryImpl对象的远程引用传入的UnicastServerRef对象，涉及向上转型(RemoteRef)，执行UnicastServerRef的exportObject方法。
>>以下：①为传入的RegistryImpl创建代理——服务于客户端的RegistryImpl_Stub对象；②将UnicastServerRef的skeleton对象设置为当前RegistryImpl对象；③用skeleton、stub、UnicastServerRef对象、id和一个boolean值构造了一个Target 对象，等待TCP调用；④调用UnicastServerRef的 ref（LiveRef）变量的exportObject()方法。  
【var1=RegistryImpl; var 2 = null ; var3=true】
- LiveRef与 TCP通信的类
```java 
// sun.rmi.server.UnicastServerRef
public Remote exportObject(Remote var1, Object var2, boolean var3) throws RemoteException {
    Class var4 = var1.getClass();

    Remote var5;
    try {
        var5 = Util.createProxy(var4, this.getClientRef(), this.forceStubUse);
    } catch (IllegalArgumentException var7) {
        throw new ExportException("remote object implements illegal remote interface", var7);
    }

    if(var5 instanceof RemoteStub) {
        this.setSkeleton(var1);
    }

    Target var6 = new Target(var1, this, var5, this.ref.getObjID(), var3);
    // protected LiveRef ref;
    this.ref.exportObject(var6);
    this.hashToMethod_Map = (Map)hashToMethod_Maps.get(var4);
    return var5;
}

// sun.rmi.transport.LiveRef
public void exportObject(Target var1) throws RemoteException {
    // private final Endpoint ep;
    this.ep.exportObject(var1);
}

// sun.rmi.transport.tcp.TCPEndpoint
// public class TCPEndpoint implements Endpoint
public void exportObject(Target var1) throws RemoteException {
    // private TCPTransport transport;
    this.transport.exportObject(var1);
}

// sun.rmi.transport.tcp.TCPTransport
// public class TCPTransport extends Transport
public void exportObject(Target var1) throws RemoteException {
    synchronized(this) {
        this.listen();
        ++this.exportCount;
    }

    boolean var2 = false;
    boolean var12 = false;

    try {
        var12 = true;
        // 父类 Transport
        super.exportObject(var1);
        var2 = true;
        var12 = false;
    } finally {
        if(var12) {
            if(!var2) {
                synchronized(this) {
                    this.decrementExportCount();
                }
            }

        }
    }

    if(!var2) {
        synchronized(this) {
            this.decrementExportCount();
        }
    }
}

// sun.rmi.transport.Transport
public void exportObject(Target var1) throws RemoteException {
    var1.setExportedTransport(this);
    ObjectTable.putTarget(var1);
}
```
>【连接层】追溯LiveRef的exportObject()->TCPTransport的exportObject()——将上面构造的Target对象暴露出去。调用TCPTransport的listen()方法——创建一个ServerSocket，启动一条线程等待客户端的请求。  
调用父类Transport的exportObject()将Target对象存放进ObjectTable。  
到这，已将RegistryImpl对象创建且起了服务等待客户端请求。