#### 架构发展
- 电商架构
>早期单一应用架构，随互联网发展，通过垂直伸
缩很难达到期望，投入产出比很大，水平伸缩成为主流。  
分布式架构的服务越多、规模越大、机器越多时，人工管理和维护服务及配置地址信息会越来越难，单点问题：一旦服务路由或负载均衡服务器宕机，依赖它的所有服务均将失效。  
此时，需要一个能动态注册和获取服务信息的地方，统一管理服务名称和其对应的服务器列表信息，称为服务配置中心。  
服务提供者启动时，将提供的服务名称、服务器地址注册到服务配置中心，服务消费者通过服务配置中心获得需调用服务的机器列表。    
通过负载均衡算法，选其中一台调用。当服务器宕机或下线时，相应的机器需能动态从服务配置中心移除，并通知相应服务消费者，否则可能调用到已失效服务而发生错误，在此过程，消费者第一次调用时需查询服务配置中心，再缓存到本地，后续直接使用本地缓存的服务地址列表，直到有变更（机器上线或下线）。  
这种无中心化的结构解决了负载均衡设备导致的单点故障问题，大大减轻服务配置中心的压力。

#### zookeeper
>开源的分布式协调服务，雅虎创建，google chubby的开源实现。  
设计目标：将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集（由若干条指令组成，完成一定功能的一个过程），提供简单易用的接口。

- 安装部署
>两种运行模式：集群和单机。
下载安装包：http://apache.fayea.com/zookeeper/

- 常用命令
```java 
1.启动服务:
bin/zkServer.sh start
2.查看服务状态:
bin/zkServer.sh status
3.停止服务:
bin/zkServer.sh stop
4.重启服务:
bin/zkServer.sh restart
5.连接服务器
zkCli.sh -timeout 0 -r -server ip:port
```
- 单机安装
>初次使用zookeeper，需将conf/zoo_sample.cfg复制一份重命名为zoo.cfg。  
修改dataDir（日志文件存放路径）目录。

- 集群安装
>集群中，各节点共有三种角色：leader、follower、observer。  
集群模式用3台机器搭建，分别复制、解压。

>- ①修改配置文件：修改端口  
server.1=IP1:2888:3888  
server.2=IP2.2888:3888  
server.3=IP3.2888:2888  
>>（2888：访问zookeeper的端口；3888：重新选举leader的端口）  
server.A=B：C：D  
>A：数字，第几号服务器；  
B：服务器ip地址；  
C：该服务器与集群中Leader交换信息的端口；  
D：若集群Leader挂了，需一个端口重新选举新Leader。该端口用来执行选举时服务器相互通信的端口。  
伪集群：B都一样，故不同Zookeeper实例通信端口号不能一样，需分配不同的端口号。  
集群模式，集群中每台机器都需感知到整个集群是
由哪几台机器组成，在配置文件中，按格式server.id=host:port:port，每一行代表一个机器配置。id:指的是serverID,用来标识该机器在集群中的机器序号。  
>- ②新建datadir目录：设置每台zookeeper的myid，都需在dataDir下创建一个myid文件，该文件只有一行内容，对应每台机器的Server ID，eg.server.1的myid为1。     确保每个服务器myid文件的数字不同， 且和所在机器的zoo.cfg中server.id的id值一致，范围是1~255。
>- ③启动zookeeper：Observer角色：在不影响写性能的情况下扩展zookeeper，本身zookeeper集群性能已很好，但若超大量客户端访问，必须增加数量，而随着服务器增加，zookeeper集群的写性能就会下降；  
zookeeper的znode变更需半数及以上服务器投票通过，而随着机器的增加，由于网络消耗等原因必导致投票成本增加，性能下降。

#### watch基本流程
>ZooKeeper Watcher机制分三过程：  
①客户端注册Watcher、 ②服务器处理Watcher 、③客户端回调Watcher。  
客户端注册watcher 3种方式：getData、exists、
getChildren。
```java 
String path = "/np-test";
String connectString = "192.168.25.154:2181";

ZooKeeper zookeeper = new ZooKeeper(connectString, 4000, new Watcher() {
    @Override public void process(WatchedEvent watchedEvent) {
        System.out.println("watched client");
    }
});
// 创建节点
zookeeper.create(path, "1".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
// 注册监听
zookeeper.exists(path, true);
// 修改节点的值触发监听
zookeeper.setData(path, "2".getBytes(), 1);
```
>创建ZooKeeper客户端实例时，通过new Watcher()传入一个默认Watcher（整个ZooKeeper会话期间默认Watcher）,一直被保存在客户端ZKWatchManager的defaultWatcher中，代码如下：

```java 
// org.apache.zookeeper.ZooKeeper
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, boolean canBeReadOnly) throws IOException {
    this.watchManager = new ZooKeeper.ZKWatchManager((ZooKeeper.SyntheticClass_1)null);
    LOG.info("Initiating client connection, connectString=" + connectString + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);
    this.watchManager.defaultWatcher = watcher;
    // 将watcher设置到ZKWatchManager
    ConnectStringParser connectStringParser = new ConnectStringParser(connectString);
    StaticHostProvider hostProvider = new StaticHostProvider(connectStringParser.getServerAddresses());
    // 初始化ClientCnxn，并调用cnxn.start  
    // org.apache.zookeeper.ClientCnxn
    this.cnxn = new ClientCnxn(connectStringParser.getChrootPath(), hostProvider, sessionTimeout, this, this.watchManager, getClientCnxnSocket(), canBeReadOnly);
    this.cnxn.start();
}
```
>ClientCnxn:Zookeeper客户端和服务器端通信和事件通知处理的主要类，内部包含两个类：
>>- SendThread：复制client和server数据通信, 包括事件信息传输；  
>>- EventThread:client回调注册的Watchers进行通知处理。  

- ClientCnxn初始化
```java 
// org.apache.zookeeper.ClientCnxn
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper, ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket, long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.authInfo = new CopyOnWriteArraySet();
    this.pendingQueue = new LinkedList();
    this.outgoingQueue = new LinkedList();
    this.sessionPasswd = new byte[16];
    this.closing = false;
    this.seenRwServerBefore = false;
    this.eventOfDeath = new Object();
    this.xid = 1;
    this.state = States.NOT_CONNECTED;
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;
    this.connectTimeout = sessionTimeout / hostProvider.size();
    this.readTimeout = sessionTimeout * 2 / 3;
    this.readOnly = canBeReadOnly;
    // 初始化sendThread
    // org.apache.zookeeper.ClientCnxn.SendThread
    this.sendThread = new ClientCnxn.SendThread(clientCnxnSocket);
    // 初始化eventThread
    // org.apache.zookeeper.ClientCnxn.EventThread
    this.eventThread = new ClientCnxn.EventThread();
}

public void start() {
    this.sendThread.start();
    this.eventThread.start();
}
```

- 客户端通过exists注册监听
```java 
org.apache.zookeeper.ZooKeeper
public Stat exists(String path, Watcher watcher) throws KeeperException, InterruptedException {
    PathUtils.validatePath(path);
    ZooKeeper.ExistsWatchRegistration wcb = null;
    // 构建ExistWatchRegistration
    if(watcher != null) {
        wcb = new ZooKeeper.ExistsWatchRegistration(watcher, path);
    }

    String serverPath = this.prependChroot(path);
    RequestHeader h = new RequestHeader();
    // 设 置操作类型为exists
    h.setType(3);
    // 构造 ExistsRequest
    ExistsRequest request = new ExistsRequest();
    request.setPath(serverPath);
    // 是否注册监听
    request.setWatch(watcher != null);
    // //设置服务端响应的接收类
    SetDataResponse response = new SetDataResponse();
    // 将封装的RequestHeader、ExistsReques、SetDataResponse、WatchRegistration添加到发送队列
    ReplyHeader r = this.cnxn.submitRequest(h, request, response, wcb);
    if(r.getErr() != 0) {
        if(r.getErr() == Code.NONODE.intValue()) {
            return null;
        } else {
            throw KeeperException.create(Code.get(r.getErr()), path);
        }
    } else {
        // 返回 exists 得到的结果（Stat 信息）
        return response.getStat().getCzxid() == -1L?null:response.getStat();
    }
}

// org.apache.zookeeper.ClientCnxn
public ReplyHeader submitRequest(RequestHeader h, Record request, Record response, WatchRegistration watchRegistration) throws InterruptedException {
    ReplyHeader r = new ReplyHeader();
    // 将消息添加到队列,并构造一个Packet传输对象
    ClientCnxn.Packet packet = this.queuePacket(h, r, request, response, (AsyncCallback)null, (String)null, (String)null, (Object)null, watchRegistration);
    synchronized(packet) {
        while(!packet.finished) {
            // 在数据包没处理完前一直阻塞
            packet.wait();
        }
        return r;
    }
}

ClientCnxn.Packet queuePacket(RequestHeader h, ReplyHeader r, Record request, Record response, AsyncCallback cb, String clientPath, String serverPath, Object ctx, WatchRegistration watchRegistration) {
    ClientCnxn.Packet packet = null;
    LinkedList var11 = this.outgoingQueue;
    synchronized(this.outgoingQueue) {
        packet = new ClientCnxn.Packet(h, r, request, response, watchRegistration);
        packet.cb = cb;
        packet.ctx = ctx;
        packet.clientPath = clientPath;
        packet.serverPath = serverPath;
        if(this.state.isAlive() && !this.closing) {
            if(h.getType() == -11) {
                this.closing = true;
            }

            this.outgoingQueue.add(packet);
        } else {
            this.conLossPacket(packet);
        }
    }

    this.sendThread.getClientCnxnSocket().wakeupCnxn();
    return packet;
}

ClientCnxn.Packet queuePacket(RequestHeader h, ReplyHeader r, Record request, Record response, AsyncCallback cb, String clientPath, String serverPath, Object ctx, WatchRegistration watchRegistration) {
    // 将相关传输对象转化成 Packet
    ClientCnxn.Packet packet = null;
    LinkedList var11 = this.outgoingQueue;
    synchronized(this.outgoingQueue) {
        packet = new ClientCnxn.Packet(h, r, request, response, watchRegistration);
        packet.cb = cb;
        packet.ctx = ctx;
        packet.clientPath = clientPath;
        packet.serverPath = serverPath;
        if(this.state.isAlive() && !this.closing) {
            if(h.getType() == -11) {
                this.closing = true;
            }
            // 添加到outgoingQueue
            this.outgoingQueue.add(packet);
        } else {
            this.conLossPacket(packet);
        }
    }

    //此处多路复用机制,唤醒Selector，告诉有数据包添加了
    this.sendThread.getClientCnxnSocket().wakeupCnxn();
    return packet;
}
```
>ZooKeeper的最小通信协议单元是Packet，即数据包。Pakcet：客户端与服务端间的网络传输，任何需传输的对象都需包装成一个Packet对象。  
ClientCnxn中WatchRegistration也会被封装到Pakcet中，由SendThread线程调用queuePacket方法把 Packet放入发送队列中等待客户端发送，这又是一个异步过程。

- SendThread发送过程
>初始化连接时，zookeeper初始化并启动了两线程。分析SendThread发送过程：
```java 
// org.apache.zookeeper.ClientCnxn.SendThread+
public void run() {
    this.clientCnxnSocket.introduce(this, ClientCnxn.this.sessionId);
    this.clientCnxnSocket.updateNow();
    this.clientCnxnSocket.updateLastSendAndHeard();
    long lastPingRwServer = System.currentTimeMillis();
    boolean MAX_SEND_PING_INTERVAL = true;

    while(ClientCnxn.this.state.isAlive()) {
        try {
            // 若无连接：发起连接
            if(!this.clientCnxnSocket.isConnected()) {
                if(!this.isFirstConnect) {
                    try {
                       Thread.sleep((long)this.r.nextInt(1000));
                    } catch (InterruptedException var9) {
                        ClientCnxn.LOG.warn("Unexpected exception", var9);
                    }
                }

                if(ClientCnxn.this.closing || !ClientCnxn.this.state.isAlive()) {
                    break;
                }
                // 发起连接
                this.startConnect();
                this.clientCnxnSocket.updateLastSendAndHeard();
            }

            int to;
            // 若处于连接状态，则处理sasl认证授权
            if(ClientCnxn.this.state.isConnected()) {
                if(ClientCnxn.this.zooKeeperSaslClient != null) {
                    boolean e = false;
                    if(ClientCnxn.this.zooKeeperSaslClient.getSaslState() == SaslState.INITIAL) {
                        try {
                            ClientCnxn.this.zooKeeperSaslClient.initialize(ClientCnxn.this);
                        } catch (SaslException var8) {
                            ClientCnxn.LOG.error("SASL authentication with Zookeeper Quorum member failed: " + var8);
                            ClientCnxn.this.state = States.AUTH_FAILED;
                            e = true;
                        }
                    }

                    KeeperState authState = ClientCnxn.this.zooKeeperSaslClient.getKeeperState();
                    if(authState != null) {
                        if(authState == KeeperState.AuthFailed) {
                            ClientCnxn.this.state = States.AUTH_FAILED;
                            e = true;
                        } else if(authState == KeeperState.SaslAuthenticated) {
                            e = true;
                        }
                    }

                    if(e) {
                        ClientCnxn.this.eventThread.queueEvent(new WatchedEvent(EventType.None, authState, (String)null));
                    }
                }

                to = ClientCnxn.this.readTimeout - this.clientCnxnSocket.getIdleRecv();
            } else {
                to = ClientCnxn.this.connectTimeout - this.clientCnxnSocket.getIdleRecv();
            }
            // to：表示客户端距离timeout还剩多少时间，准备发起ping连接
            if(to <= 0) {
                throw new ClientCnxn.SessionTimeoutException("Client session timed out, have not heard from server in " + this.clientCnxnSocket.getIdleRecv() + "ms" + " for sessionid 0x" + Long.toHexString(ClientCnxn.this.sessionId));
            }
            // 计算下一次ping请求的时间
            if(ClientCnxn.this.state.isConnected()) {
                int e1 = ClientCnxn.this.readTimeout / 2 - this.clientCnxnSocket.getIdleSend() - (this.clientCnxnSocket.getIdleSend() > 1000?1000:0);
                if(e1 > 0 && this.clientCnxnSocket.getIdleSend() <= 10000) {
                    if(e1 < to) {
                        to = e1;
                    }
                } else {
                    // 发送ping请求
                    this.sendPing();
                    this.clientCnxnSocket.updateLastSend();
                }
            }

            if(ClientCnxn.this.state == States.CONNECTEDREADONLY) {
                long e2 = System.currentTimeMillis();
                int idlePingRwServer = (int)(e2 - lastPingRwServer);
                if(idlePingRwServer >= this.pingRwTimeout) {
                    lastPingRwServer = e2;
                    idlePingRwServer = 0;
                    this.pingRwTimeout = Math.min(2 * this.pingRwTimeout, '\uea60');
                    this.pingRwServer();
                }

                to = Math.min(to, this.pingRwTimeout - idlePingRwServer);
            }
// 调用clientCnxnSocket，发起传输。pendingQueue：用来存放已发送、等待回应的Packet队列，
clientCnxnSocket默认使用ClientCnxnSocketNIO（在实例化zookeeper时）
            this.clientCnxnSocket.doTransport(to, ClientCnxn.this.pendingQueue, ClientCnxn.this.outgoingQueue, ClientCnxn.this);
        } catch (Throwable var10) {
            if(ClientCnxn.this.closing) {
                if(ClientCnxn.LOG.isDebugEnabled()) {
                    ClientCnxn.LOG.debug("An exception was thrown while closing send thread for session 0x" + Long.toHexString(ClientCnxn.this.getSessionId()) + " : " + var10.getMessage());
                }
                break;
            }

            if(var10 instanceof ClientCnxn.SessionExpiredException) {
                ClientCnxn.LOG.info(var10.getMessage() + ", closing socket connection");
            } else if(var10 instanceof ClientCnxn.SessionTimeoutException) {
                ClientCnxn.LOG.info(var10.getMessage() + ", closing socket connection and attempting reconnect");
            } else if(var10 instanceof ClientCnxn.EndOfStreamException) {
                ClientCnxn.LOG.info(var10.getMessage() + ", closing socket connection and attempting reconnect");
            } else if(var10 instanceof ClientCnxn.RWServerFoundException) {
                ClientCnxn.LOG.info(var10.getMessage());
            } else {
                ClientCnxn.LOG.warn("Session 0x" + Long.toHexString(ClientCnxn.this.getSessionId()) + " for server " + this.clientCnxnSocket.getRemoteSocketAddress() + ", unexpected error" + ", closing socket connection and attempting reconnect", var10);
            }

            this.cleanup();
            if(ClientCnxn.this.state.isAlive()) {
                ClientCnxn.this.eventThread.queueEvent(new WatchedEvent(EventType.None, KeeperState.Disconnected, (String)null));
            }

            this.clientCnxnSocket.updateNow();
            this.clientCnxnSocket.updateLastSendAndHeard();
        }
    }

    this.cleanup();
    this.clientCnxnSocket.close();
    if(ClientCnxn.this.state.isAlive()) {
        ClientCnxn.this.eventThread.queueEvent(new WatchedEvent(EventType.None, KeeperState.Disconnected, (String)null));
    }

    ZooTrace.logTraceMessage(ClientCnxn.LOG, ZooTrace.getTextTraceLevel(), "SendThread exitedloop.");
}
```
- client和server的网络交互

```java 
// org.apache.zookeeper.ClientCnxnSocketNIO
void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn) throws IOException, InterruptedException {
    // java.nio.channels.Selector
    this.selector.select((long)waitTimeOut);
    Set selected;
    synchronized(this) {
        selected = this.selector.selectedKeys();
    }

    this.updateNow();
    Iterator i$ = selected.iterator();

    while(i$.hasNext()) {
        SelectionKey k = (SelectionKey)i$.next();
        SocketChannel sc = (SocketChannel)k.channel();
        // 若之前没立马连上，则处理OP_CONNECT事件
        if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
            if(sc.finishConnect()) {
                this.updateLastSendAndHeard();
                this.sendThread.primeConnection();
            }
        } 
        // 若读写就位，则处理
        else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
            this.doIO(pendingQueue, outgoingQueue, cnxn);
        }
    }

    if(this.sendThread.getZkState().isConnected()) {
        synchronized(outgoingQueue) {
            // 找到连接Packet并将其放到队列头
            if(this.findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                // 将Channecl设置为可读
                this.enableWrite();
            }
        }
    }

    selected.clear();
}

private Packet findSendablePacket(LinkedList<Packet> outgoingQueue,boolean clientTunneledAuthenticationInProgress) {
    synchronized (outgoingQueue) {
        ...
        // 因为Conn Packet需发送到SASL authentication进行处理，其他Packet都需等待直到该处理完成。Conn Packet必须第一个处理，故找出并放到OutgoingQueue头
        ListIterator<Packet> iter = outgoingQueue.listIterator();
        while (iter.hasNext()) {
            Packet p = iter.next();
            if (p.requestHeader == null) {
                iter.remove();  
                outgoingQueue.add(0, p); 
                // 将连接放到outgogingQueue第一个元素
                return p;
            } else {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("deferring non-priming packet: " + p +
                            "until SASL authentication completes.");
                }
            }
        }
        // no sendable packet found.
        return null;
    }
}

// 最重要的IO部分：需处理两类网络事件（读、写）
void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn) throws InterruptedException, IOException {
    SocketChannel sock = (SocketChannel)this.sockKey.channel();
    if(sock == null) {
        throw new IOException("Socket is null!");
    } else {
        if(this.sockKey.isReadable()) {
            // 先从Channel读4个字节，代表头  
            int rc = sock.read(this.incomingBuffer);
            if(rc < 0) {
                throw new EndOfStreamException("Unable to read additional data from server sessionid 0x" + Long.toHexString(this.sessionId) + ", likely server has closed socket");
            }

            if(!this.incomingBuffer.hasRemaining()) {
                this.incomingBuffer.flip();
                if(this.incomingBuffer == this.lenBuffer) {
                    ++this.recvCount;
                    this.readLength();
                } 
                // 初始化
                else if(!this.initialized) {
                    // 读取连接结果
                    this.readConnectResult();
                    // Channel可读
                    this.enableRead();
                    if(this.findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                        this.enableWrite();
                    }

                    this.lenBuffer.clear();
                    this.incomingBuffer = this.lenBuffer;
                    this.updateLastHeard();
                    this.initialized = true;
                } else {
                    // 处理其他请求
                    this.sendThread.readResponse(this.incomingBuffer);
                    this.lenBuffer.clear();
                    this.incomingBuffer = this.lenBuffer;
                    this.updateLastHeard();
                }
            }
        }

        if(this.sockKey.isWritable()) {
            synchronized(outgoingQueue) {
                // 获得packet
                Packet p = this.findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress());
                if(p != null) {
                    this.updateLastSend();
                    if(p.bb == null) {
                    // 若不是连接事件、ping 事件、认证时间 
                    if((p.requestHeader != null) &&      (p.requestHeader.getType() != OpCode.ping) &&     (p.requestHeader.getType() != OpCode.auth)) {    p.requestHeader.setXid(cnxn.getXid());
                        }
                        // 序列化
                        p.createBB();
                    }
                    // 将数据写入Channel
                    sock.write(p.bb);
                    // p.bb中若无内容则发送成功
                    if(!p.bb.hasRemaining()) {
                        // 发送数+1
                        ++this.sentCount;
                      // 将p从队列移除 outgoingQueue.removeFirstOccurrence(p);
                      // 若该事件不是连接事件、ping事件、认证事件， 则加入pending队列
                        if(p.requestHeader != null && p.requestHeader.getType() != 11 && p.requestHeader.getType() != 100) {
                            synchronized(pendingQueue) {
                                pendingQueue.add(p);
                            }
                        }
                    }
                }

                if(outgoingQueue.isEmpty()) {
                    this.disableWrite();
                } else if(!this.initialized && p != null && !p.bb.hasRemaining()) {
                    this.disableWrite();
                } else {
                    this.enableWrite();
                }
            }
        }
    }
}
```

- createBB()
```java 
// org.apache.zookeeper.ClientCnxn
public void createBB() {
    try {
        ByteArrayOutputStream e = new ByteArrayOutputStream();
        BinaryOutputArchive boa = BinaryOutputArchive.getArchive(e);
        boa.writeInt(-1, "len");
        // 若不是连接事件则设置协议头
        if(this.requestHeader != null) {
            this.requestHeader.serialize(boa, "header");
        }

        // 设置协议体
        if(this.request instanceof ConnectRequest) {
            this.request.serialize(boa, "connect");
            boa.writeBool(this.readOnly, "readOnly");
        } else if(this.request != null) {
            this.request.serialize(boa, "request");
        }

        e.close();
        // 生成ByteBuffer
        this.bb = ByteBuffer.wrap(e.toByteArray());
        // 将bytebuffer的前4个字节修改成真正的长度，总长度减去一个int的长度头 
        this.bb.putInt(this.bb.capacity() - 4);
        // 准备后续读，让buffer position = 0
        this.bb.rewind();
    } catch (IOException var3) {
        ClientCnxn.LOG.warn("Ignoring unexpected exception", var3);
    }
}
```

>还有一个比较关键的函数：readResponse函数，用来消费PendingQueue，处理消息分三类：  
 ping 消息 XID=-2；  
 auth认证消息 XID=-4 ；  
 订阅的消息，即各种变化的通知，eg.子节点变化、节点内容变化，由服务器推过来的消息，获取到这类消息或通过eventThread.queueEvent将消息推入事件队列 。



https://blog.csdn.net/cnh294141800/article/details/53039482

