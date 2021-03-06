#### RPC
>Remote Procedure Call——远程过程调用。实现不同机器系统间的方法调用，使程序像访问本地系统一样，通过网络传输访问远端系统资源。对于客户端， 传输层用啥协议，序列化、反序列化都是透明的。

- Java RMI
>remote method invocation——远程方法调用，是应用程序编程接口，网络分布式应用的核心解决方案之一。  
其使用JRMP （Java Remote Messageing Protocol——Java 远程消息交换协议）进行通信。由于JRMP专为Java对象制定的，不能与非JAVA对象通信。

- Java RMI代码实践
>远程对象必须实现UnicastRemoteObject，才能保证客户端访问获得远程对象时，该远程对象把自身拷贝以 Socket传给client，client获得的拷贝称为stub，而server已存在的远程对象称为skeleton。   
stub：client的代理，用于server通信；  
skeleton：server的代理，用于接收client请求后调用远程方法响应客户端的请求。

#### Java RMI源码分析
##### 源码结构图  
![image](https://raw.githubusercontent.com/nanphonfy/note-images/master/promote-2019/distributed/06/UnicastRemoteObject.png)

![image](https://raw.githubusercontent.com/nanphonfy/note-images/master/promote-2019/distributed/06/UnicastServerRef.png)

##### 远程对象发布
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

// cn.nanphonfy.mi.server.Server
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


##### 客户端获取服务端Registry代理

```java 
// cn.nanphonfy.mi.client.Client
UserService userService = (UserService) Naming.lookup(Server.BINDING_ADDRESS);
```
>追溯到LocateRegistry的getRegistry()——通过传入的host、port构造RemoteRef对象，并创建一个本地代理(RegistryImpl_Stub对象)。  
客户端有了RegistryImpl的代理，此时该代理还没
和服务端关联（两个VM对象），代理和远程的Registry对象间通过socket完成。

```java 
// java.rmi.Naming
public static Remote lookup(String name)
    throws NotBoundException,
        java.net.MalformedURLException,
        RemoteException{
    ParsedNamingURL parsed = parseURL(name);
    Registry registry = getRegistry(parsed);

    if (parsed.name == null)
        return registry;
    return registry.lookup(parsed.name);
}

// java.rmi.registry.LocateRegistry
public static Registry getRegistry(String host, int port,RMIClientSocketFactory csf) throws RemoteException{
    Registry registry = null;
    // 获取仓库地址
    if (port <= 0)
        port = Registry.REGISTRY_PORT;

    if (host == null || host.length() == 0){
        // If host is blank (as returned by "file:" URL in 1.0.2 used in
        // java.rmi.Naming), try to convert to real local host name so
        // that the RegistryImpl's checkAccess will not fail.
        try {
            host = java.net.InetAddress.getLocalHost().getHostAddress();
        } catch (Exception e) {
            // If that failed, at least try "" (localhost) anyway...
            host = "";
        }
    }
    // 与TCP通信的类
    LiveRef liveRef = new LiveRef(new ObjID(ObjID.REGISTRY_ID),new TCPEndpoint(host, port, csf, null),false);
    RemoteRef ref = (csf == null) ? new UnicastRef(liveRef) : new UnicastRef2(liveRef);

    // 创建远程代理类，赋引用liveref，动态代理时能TCP通信
    return (Registry) Util.createProxy(RegistryImpl.class, ref, false);
}
```
>调用RegistryImpl_Stub的ref（RemoteRef）对象的newCall()方法，将RegistryImpl_Stub对象传进去， 把服务器相关的信息也传进newCall()——建立跟远程RegistryImpl的Skeleton对象的连接。（服务端通过 CPTransport 的exportObject()
等待客户端请求）


```java  
// sun.rmi.server.UnicastRef
public RemoteCall newCall(RemoteObject var1, Operation[] var2, int var3, long var4) throws RemoteException {
    clientRefLog.log(Log.BRIEF, "get connection");
    Connection var6 = this.ref.getChannel().newConnection();

    try {
        clientRefLog.log(Log.VERBOSE, "create call context");
        if(clientCallLog.isLoggable(Log.VERBOSE)) {
            this.logClientCall(var1, var2[var3]);
        }

        StreamRemoteCall var7 = new StreamRemoteCall(var6, this.ref.getObjID(), var3, var4);

        try {
            this.marshalCustomCallData(var7.getOutputStream());
        } catch (IOException var9) {
            throw new MarshalException("error marshaling custom call data");
        }

        return var7;
    } catch (RemoteException var10) {
        this.ref.getChannel().free(var6, false);
        throw var10;
    }
}
```
>连接建立后，发送请求。客户端拥有Registry对象的代理，不是真正位于服务端的Registry对象本身，他们位于不同的虚拟机中，无法直接调用，通过消息进行交互。super.ref.invoke() ——追溯到
StreamRemoteCall的executeCall()方法。看似本地调用，但是通过tcp连接发送消息到服务端。由服务端解析并且处理调用。   
>至此，已将客户端的请求发出。

```java 
// sun.rmi.server.UnicastRef
public void invoke(RemoteCall var1) throws Exception {
    try {
        clientRefLog.log(Log.VERBOSE, "execute call");
        var1.executeCall();
    } catch (RemoteException var3) {
        clientRefLog.log(Log.BRIEF, "exception: ", var3);
        this.free(var1, false);
        throw var3;
    } catch (Error var4) {
        clientRefLog.log(Log.BRIEF, "error: ", var4);
        this.free(var1, false);
        throw var4;
    } catch (RuntimeException var5) {
        clientRefLog.log(Log.BRIEF, "exception: ", var5);
        this.free(var1, false);
        throw var5;
    } catch (Exception var6) {
        clientRefLog.log(Log.BRIEF, "exception: ", var6);
        this.free(var1, true);
        throw var6;
    }
}

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
public void exportObject(Target var1) throws RemoteException {
    // private TCPTransport transport;
    this.transport.exportObject(var1);
}

// sun.rmi.transport.tcp.TCPTransport
public void exportObject(Target var1) throws RemoteException {
    synchronized(this) {
        this.listen();
        ++this.exportCount;
    }

    boolean var2 = false;
    boolean var12 = false;

    try {
        var12 = true;
        super.exportObject(var1);
        ......

}
```
>TCP协议层发起socket监听，多线程循环接收请
求：TCPTransport.AcceptLoop(this.server)

```java 
// sun.rmi.transport.tcp.TCPTransport
private void listen() throws RemoteException {
    assert Thread.holdsLock(this);

    TCPEndpoint var1 = this.getEndpoint();
    int var2 = var1.getPort();
    if(this.server == null) {
        if(tcpLog.isLoggable(Log.BRIEF)) {
            tcpLog.log(Log.BRIEF, "(port " + var2 + ") create server socket");
        }
        try {
            this.server = var1.newServerSocket();
            // sun.rmi.transport.tcp.TCPTransport
            Thread var3 = (Thread)AccessController.doPrivileged(new NewThreadAction(new TCPTransport.AcceptLoop(this.server), "TCP Accept-" + var2, true));
            var3.start();
        } catch (BindException var4) {
            throw new ExportException("Port already in use: " + var2, var4);
        } catch (IOException var5) {
            throw new ExportException("Listen failed on port: " + var2, var5);
        }
    } else {
        SecurityManager var6 = System.getSecurityManager();
        if(var6 != null) {
            var6.checkListen(var2);
        }
    }
}

// sun.rmi.transport.tcp.TCPTransport
private class AcceptLoop implements Runnable {
    private final ServerSocket serverSocket;
    private long lastExceptionTime = 0L;
    private int recentExceptionCount;

    AcceptLoop(ServerSocket var2) {
        this.serverSocket = var2;
    }

    public void run() {
        try {
            this.executeAcceptLoop();
        } finally {
            try {
                this.serverSocket.close();
            } catch (IOException var7) {
                ;
            }
        }
    }
    ......
    
    
private void executeAcceptLoop() {
    if(TCPTransport.tcpLog.isLoggable(Log.BRIEF)) {
        TCPTransport.tcpLog.log(Log.BRIEF, "listening on port " + TCPTransport.this.getEndpoint().getPort());
    }

    while(true) {
        Socket var1 = null;

        try {
            var1 = this.serverSocket.accept();
            InetAddress var16 = var1.getInetAddress();
            String var3 = var16 != null?var16.getHostAddress():"0.0.0.0";

            try {
                TCPTransport.connectionThreadPool.execute(TCPTransport.this.new ConnectionHandler(var1, var3));
            } catch 
            ......
            
// sun.rmi.transport.tcp.TCPTransport.ConnectionHandler
private class ConnectionHandler implements Runnable {
    ......
    public void run() {
        Thread var1 = Thread.currentThread();
        String var2 = var1.getName();

        try {
            var1.setName("RMI TCP Connection(" + TCPTransport.connectionCount.incrementAndGet() + ")-" + this.remoteHost);
            AccessController.doPrivileged(() -> {
                this.run0();
                return null;
            }, TCPTransport.NOPERMS_ACC);
        } finally {
            var1.setName(var2);
        }

    }
    ......
```
>run0方法对不同协议做处理。最终调用:TCPTransport.this.handleMessages(var14, true);

```java 
// sun.rmi.transport.tcp.TCPTransport
// var15 = this.socket.getInputStream()
......
switch(var15) {
case 75:
    var10.writeByte(78);
    if(TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
        TCPTransport.tcpLog.log(Log.VERBOSE, "(port " + var2 + ") " + "suggesting " + this.remoteHost + ":" + var11);
    }

    var10.writeUTF(this.remoteHost);
    var10.writeInt(var11);
    var10.flush();
    String var16 = var5.readUTF();
    int var17 = var5.readInt();
    if(TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
        TCPTransport.tcpLog.log(Log.VERBOSE, "(port " + var2 + ") client using " + var16 + ":" + var17);
    }

    var12 = new TCPEndpoint(this.remoteHost, this.socket.getLocalPort(), var1.getClientSocketFactory(), var1.getServerSocketFactory());
    var13 = new TCPChannel(TCPTransport.this, var12);
    var14 = new TCPConnection(var13, this.socket, (InputStream)var4, var9);
    TCPTransport.this.handleMessages(var14, true);
    return;
......  

void handleMessages(Connection var1, boolean var2) {
    int var3 = this.getEndpoint().getPort();

    try {
        DataInputStream var4 = new DataInputStream(var1.getInputStream());

        do {
            int var5 = var4.read();
            if(var5 == -1) {
                if(tcpLog.isLoggable(Log.BRIEF)) {
                    tcpLog.log(Log.BRIEF, "(port " + var3 + ") connection closed");
                }
                break;
            }

            if(tcpLog.isLoggable(Log.BRIEF)) {
                tcpLog.log(Log.BRIEF, "(port " + var3 + ") op = " + var5);
            }

            switch(var5) {
            case 80:
                StreamRemoteCall var6 = new StreamRemoteCall(var1);
                if(!this.serviceCall(var6)) {
                    return;
                }
                break;
    ......
```
>这个地方也做了判断，若不知道怎么走，在这里加断点，会走到 case 80的段落，执行serviceCall()。

>Transport的serviceCall(关键方法)。到ObjectTable.getTarget()为止：从socket流获取ObjId，通过ObjId和Transport对象获取Target对象（已是服务端对象），再借由Target的派发器Dispatcher，传入参数服务实现和请求对象RemoteCall，将请求派发给服务端真正提供服务的RegistryImpl的lookUp()方法，即Skeleton（负责底层操作）移交给具体实现的过程。

```java 
// sun.rmi.transport.Transport
public boolean serviceCall(final RemoteCall var1) {
try {
    ObjID var39;
    try {
        var39 = ObjID.read(var1.getInputStream());
    } catch (IOException var33) {
        throw new MarshalException("unable to read objID", var33);
    }

    Transport var40 = var39.equals(dgcID)?null:this;
    // 此处的Target为服务端对象
    Target var5 = ObjectTable.getTarget(new ObjectEndpoint(var39, var40));
    final Remote var37;
    if(var5 != null && (var37 = var5.getImpl()) != null) {
        final Dispatcher var6 = var5.getDispatcher();
        var5.incrementCallCount();

        boolean var8;
        try {
            transportLog.log(Log.VERBOSE, "call dispatcher");
            final AccessControlContext var7 = var5.getAccessControlContext();
            ClassLoader var41 = var5.getContextClassLoader();
            ClassLoader var9 = Thread.currentThread().getContextClassLoader();

            try {
                setContextClassLoader(var41);
                currentTransport.set(this);

                try {
                    AccessController.doPrivileged(new PrivilegedExceptionAction() {
                        public Void run() throws IOException {
                            Transport.this.checkAcceptPermission(var7);
                            var6.dispatch(var37, var1);
                            return null;
                        }
                    }, var7);
                    return true;
                } catch (PrivilegedActionException var31) {
                    throw (IOException)var31.getException();
                }
            } finally {
                setContextClassLoader(var9);
                currentTransport.set((Object)null);
            }
        } catch (IOException var34) {
        ......

```
>客户端通过如下：①创建一个RegistryImpl_Stub的代理类；②通过代理类socket请求，将lookup发送到服务端；③服务端接收请求后，通过RegistryImpl_Stub（Skeleton）执行RegistryImpl的lookUp。
>>服务端的RegistryImpl返回：服务端的UserServiceImpl实现类。
```java
// cn.nanphonfy.mi.client.Client
UserService userService = (UserService) Naming.lookup(Server.BINDING_ADDRESS);

// sun.rmi.registry.RegistryImpl
public Remote lookup(String var1) throws RemoteException, NotBoundException {
    Hashtable var2 = this.bindings;
    synchronized(this.bindings) {
        Remote var3 = (Remote)this.bindings.get(var1);
        if(var3 == null) {
            throw new NotBoundException(var1);
        } else {
            return var3;
        }
    }
}
```
>客户端通过lookUp查询获得的客户端HelloServiceImpl的Stub对象（Skeleton屏蔽了，故为透明）。后续的处理：通过UserServiceImpl_Stub代理对象socket网络请求服务端，通过服务端的HelloServiceImpl_Stub(Skeleton) 代理，将请求通过Dispatcher 转发到对应的服务端方法，获得结果后再通过 socket把结果返回客户端。

#### RMI做了啥
>有两个代理类：①RegistryImpl的代理类；②HelloServiceImpl的代理类。

- 时序图
![image](https://raw.githubusercontent.com/nanphonfy/note-images/master/promote-2019/distributed/06/RMI.png)

- RMI基本原理
>Client的RMI调用前，必须通过LocateRegistry或Naming方式到RMI注册表寻找要调用的RMI注册信息。找到后，Client从RMI注册表获取该RMI Remote Service的Stub信息。然后才能正式调用。  
正式调用的过程不是由RMI Client直接访问Remote Service，而是由客户端获取Stub作为RMI Client 的代理访问Remote Service的代理Skeleton。即真实的请求调用在Stub-Skeleton间进行。  
Registry不参与具体的Stub-Skeleton的调用过程，只负责记录"哪个服务名"使用哪个Stub，并在 Remote Client询问它时将该Stub返回（无则报错）。