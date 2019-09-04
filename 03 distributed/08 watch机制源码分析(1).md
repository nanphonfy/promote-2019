
[toc]

### 服务端接收请求处理流程
#### NettyServerCnxn
>服务端有一个NettyServerCnxn类，用来处理客户端发送请求。

```java 
// org.apache.zookeeper.server.NettyServerCnxn
public void receiveMessage(ChannelBuffer message) {
    try {
        while(message.readable() && !throttled) {
            // ByteBuffer不为空
            if (bb != null) {
                // bb剩余空间>message中可读字节大小
                if (bb.remaining() > message.readableBytes()) {
                    int newLimit = bb.position() + message.readableBytes();
                    bb.limit(newLimit);
                }
                // 将message写入bb中
                message.readBytes(bb);
                bb.limit(bb.capacity());
                // 已读完messages
                if (bb.remaining() == 0) {
                    // 统计接收信息
                    packetReceived();
                    bb.flip();

                    ZooKeeperServer zks = this.zkServer;
                    // zookeeper是否运行
                    if (zks == null || !zks.isRunning()) {
                        throw new IOException("ZK down");
                    }
                    if (initialized) {
                        // 【处理客户端传过来的数据包】
                        zks.processPacket(this, bb);

                        if (zks.shouldThrottle(outstandingCount.incrementAndGet())) {
                            disableRecvNoWait();
                        }
                    } else {
                        LOG.debug("got conn req request from "+ getRemoteSocketAddress());
                        zks.processConnectRequest(this, bb);
                        initialized = true;
                    }
                    bb = null;
                }
            } else {
                if (message.readableBytes() < bbLen.remaining()) {
                    bbLen.limit(bbLen.position() + message.readableBytes());
                }
                message.readBytes(bbLen);
                bbLen.limit(bbLen.capacity());
                if (bbLen.remaining() == 0) {
                    bbLen.flip();

                    int len = bbLen.getInt();

                    bbLen.clear();
                    if (!initialized) {
                        if (checkFourLetterWord(channel, message, len)) {
                            return;
                        }
                    }
                    if (len < 0 || len > BinaryInputArchive.maxBuffer) {
                        throw new IOException("Len error " + len);
                    }
                    bb = ByteBuffer.allocate(len);
                }
            }
        }
    } catch(IOException e) {
        LOG.warn("Closing connection to " + getRemoteSocketAddress(), e);
        close();
    }
}
```

#### ZookeeperServer——zks.processPacket(this, this.bb);  
>处理客户端传送过来的数据包。
```java 
// org.apache.zookeeper.server.ZooKeeperServer
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    // We have the request, now process and setup for next
    InputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    // 反序列化客户端header头信息  
    h.deserialize(bia, "header");
    incomingBuffer = incomingBuffer.slice();
    // 判断当前操作类型
    if (h.getType() == OpCode.auth) {
        LOG.info("got auth packet " + cnxn.getRemoteSocketAddress());
        AuthPacket authPacket = new AuthPacket();
        ByteBufferInputStream.byteBuffer2Record(incomingBuffer, authPacket);
        String scheme = authPacket.getScheme();
        AuthenticationProvider ap = ProviderRegistry.getProvider(scheme);
        Code authReturn = KeeperException.Code.AUTHFAILED;
        if(ap != null) {
            try {
                authReturn = ap.handleAuthentication(cnxn, authPacket.getAuth());
            } catch(RuntimeException e) {
                LOG.warn("Caught runtime exception from AuthenticationProvider: " + scheme + " due to " + e);
                authReturn = KeeperException.Code.AUTHFAILED;                   
            }
        }
        if (authReturn!= KeeperException.Code.OK) {
            if (ap == null) {
                LOG.warn("No authentication provider for scheme: "+ scheme + " has "+ ProviderRegistry.listProviders());
            } else {
                LOG.warn("Authentication failed for scheme: " + scheme);
            }
            // send a response...
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.AUTHFAILED.intValue());
            cnxn.sendResponse(rh, null, null);
            // ... and close connection
            cnxn.sendBuffer(ServerCnxnFactory.closeConn);
            cnxn.disableRecv();
        } else {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Authentication succeeded for scheme: "+ scheme);
            }
            LOG.info("auth success " + cnxn.getRemoteSocketAddress());
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh, null, null);
        }
        return;
    } else {
        // 若不是授权操作，再判断是否为sasl操作
        if (h.getType() == OpCode.sasl) {
            Record rsp = processSasl(incomingBuffer,cnxn);
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh,rsp, "response"); // not sure about 3rd arg..what is it?
            return;
        }
        // 最终进入该代码块
        else {
            // 封装请求对象
            Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(),h.getType(), incomingBuffer, cnxn.getAuthInfo());
            si.setOwner(ServerCnxn.me);
            // 【提交请求】
            submitRequest(si);
        }
    }
    cnxn.incrOutstandingRequests(h);
}
```
#### submitRequest
- 负责在服务端提交当前请求
```java 
// org.apache.zookeeper.server.ZooKeeperServer
public void submitRequest(Request si) {
    // processor处理器（org.apache.zookeeper.server.RequestProcessor）
    // protected RequestProcessor firstProcessor;
    if (firstProcessor == null) {
        synchronized (this) {
            try {
                while (state == State.INITIAL) {
                    wait(1000);
                }
            } catch (InterruptedException e) {
                LOG.warn("Unexpected interruption", e);
            }
            if (firstProcessor == null || state != State.RUNNING) {
                throw new RuntimeException("Not started");
            }
        }
    }
    try {
        touch(si.cnxn);
        boolean validpacket = Request.isValid(si.type);
        if (validpacket) {
            // 【调用firstProcessor】
            firstProcessor.processRequest(si);
            if (si.cnxn != null) {
                incInProcess();
            }
        } else {
            LOG.warn("Received packet at server of unknown type " + si.type);
            new UnimplementedRequestProcessor().processRequest(si);
        }
    } catch (MissingSessionException e) {
    } catch (RequestProcessorException e) {
        LOG.error("Unable to process request:" + e.getMessage(), e);
    }
}
```
##### firstProcessor请求链
> firstProcessor初始化：ZookeeperServer.setupRequestProcessor
```java 
// org.apache.zookeeper.server.ZooKeeperServer
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    // 构造方法传递FinalRequestProcessor
    RequestProcessor syncProcessor = new SyncRequestProcessor(this,finalProcessor);
    ((SyncRequestProcessor)syncProcessor).start();
    // 【firstProcessor是一个PrepRequestProcessor（构造方法传递1个Processor）构成1个调用链】
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}
```
>整个调用链：PrepRequestProcessor->SyncRequestProcessor ->FinalRequestProcessor

#### PredRequestProcessor.processRequest
```java 
// org.apache.zookeeper.server.PrepRequestProcessor
public PrepRequestProcessor(ZooKeeperServer zks,RequestProcessor nextProcessor) {
    super("ProcessThread(sid:" + zks.getServerId() + " cport:"
            + zks.getClientPort() + "):", zks.getZooKeeperServerListener());
    this.nextProcessor = nextProcessor;
    this.zks = zks;
}

// nextProcessor.processRequest(request); FinalRequestProcessor

public void processRequest(Request request) {
    // LinkedBlockingQueue<Request> submittedRequests = new LinkedBlockingQueue<Request>();
    submittedRequests.add(request);
}
```
>通过如上调用链关系，看到firstProcessor.processRequest(si)会调用到PrepRequestProcessor，processRequest只把request添加到submittedRequests中（异步操作）。而subittedRequests又是一个阻塞队列。PrepRequestProcessor类继承了线程类（public class PrepRequestProcessor extends ZooKeeperCriticalThread implements RequestProcessor），其run方法如下：

```java 
// org.apache.zookeeper.server.PrepRequestProcessor
public void run() {
    try {
        while (true) {
            // 从队列中拿到请求进行处理
            Request request = submittedRequests.take();
            if (Request.requestOfDeath == request) {
                break;
            }
            // 【调用pRequest进行预处理】
            pRequest(request);
        }
    } catch (RequestProcessorException e) {
        handleException(this.getName(), e);
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
}
```
##### pRequest
>预处理代码很长。前面N行代码都根据当前OP类型判断和做相应处理，该方法最后一行nextProcessor.processRequest(request)（SyncRequestProcessor）

#### SyncRequestProcessor.processRequest
>该方法也是基于异步化操作，把请求添加到queuedRequets中，继续在当前类找到run方法
```java 
// org.apache.zookeeper.server.SyncRequestProcessor
public void processRequest(Request request) {
    // private final LinkedBlockingQueue<Request> queuedRequests = new LinkedBlockingQueue<Request>();
    queuedRequests.add(request);
}

public void run() {
    try {
        int logCount = 0;
        // private final Random r = new Random(System.nanoTime());
        setRandRoll(r.nextInt(snapCount/2));
        while (true) {
            Request si = null;
            // 从阻塞队列中获取请求
            if (toFlush.isEmpty()) {
                si = queuedRequests.take();
            } else {
                si = queuedRequests.poll();
                if (si == null) {
                    flush(toFlush);
                    continue;
                }
            }
            if (si == requestOfDeath) {
                break;
            }
            if (si != null) {
                // 触发快照操作，启动一个处理快照线程
                // track the number of records written to the log
                if (zks.getZKDatabase().append(si)) {
                    logCount++;
                    if (logCount > (snapCount / 2 + randRoll)) {
                        setRandRoll(r.nextInt(snapCount/2));
                        // roll the log
                        zks.getZKDatabase().rollLog();
                        // take a snapshot
                        // private Thread snapInProcess = null;
                        if (snapInProcess != null && snapInProcess.isAlive()) {
                            LOG.warn("Too busy to snap, skipping");
                        } else {
                            snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                    public void run() {
                                        try {
                                            zks.takeSnapshot();
                                        } catch(Exception e) {
                                            LOG.warn("Unexpected exception", e);
                                        }
                                    }
                                };
                            snapInProcess.start();
                        }
                        logCount = 0;
                    }
                } 
                // private final LinkedList<Request> toFlush = new LinkedList<Request>();
                // Transactions that have been written and are waiting to be flushed to disk
                else if (toFlush.isEmpty()) {
                    // 继续调用下一个处理器处理请求
                    if (nextProcessor != null) {
                        【FinalRequestProcessor.processRequest】
                        nextProcessor.processRequest(si);
                        if (nextProcessor instanceof Flushable) {
                            ((Flushable)nextProcessor).flush();
                        }
                    }
                    continue;
                }
                toFlush.add(si);
                if (toFlush.size() > 1000) {
                    flush(toFlush);
                }
            }
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
        running = false;
    }
    LOG.info("SyncRequestProcessor exited!");
}
```
#### FinalRequestProcessor.processRequest
>该方法根据Request对象中的操作更新内存中Session信息或znode数据。  
有300多行，直接定位到关键代码：

```java 
// org.apache.zookeeper.server.FinalRequestProcessor
case OpCode.exists: {
    lastOp = "EXIS";
    // TODO we need to figure out the security requirement for this!
    ExistsRequest existsRequest = new ExistsRequest();
    // 反序列化(将ByteBuffer反序列化成为ExitsRequest。即客户端发起请求传递过来的Request对象
    ByteBufferInputStream.byteBuffer2Record(request.request,existsRequest);
    // 得到请求的路径
    String path = existsRequest.getPath();
    if (path.indexOf('\0') != -1) {
        throw new KeeperException.BadArgumentsException();
    }
    //【很关键的代码】判断请求的getWatch是否存在，若存在，则传递cnxn（servercnxn）
    // 对于exists请求，需监听data变化事件，添加watcher【statNode】
    Stat stat = zks.getZKDatabase().statNode(path, existsRequest.getWatch() ? cnxn : null);
    // 在服务端内存数据库中根据路径得到结果进行组装，设置为ExistsResponse
    rsp = new ExistsResponse(stat);
    break;
}
```
- statNode

```java 
// org.apache.zookeeper.server.ZKDatabase
public Stat statNode(String path, ServerCnxn serverCnxn) throws KeeperException.NoNodeException {
    // org.apache.zookeeper.server.DataTree
    // protected DataTree dataTree;
    return dataTree.statNode(path, serverCnxn);
}

// org.apache.zookeeper.server.DataTree
public Stat statNode(String path, Watcher watcher) throws KeeperException.NoNodeException {
    Stat stat = new Stat();
    // 获得节点数据
    DataNode n = nodes.get(path);
    // 若watcher不为空，则将当前watcher和path进行绑定
    if (watcher != null) {
        // private final WatchManager dataWatches = new WatchManager();【addWatch】
        dataWatches.addWatch(path, watcher);
    }
    if (n == null) {
        throw new KeeperException.NoNodeException();
    }
    synchronized (n) {
        n.copyStat(stat);
        return stat;
    }
}
```
- addWatch
```java 
// org.apache.zookeeper.server.WatchManager
public synchronized void addWatch(String path, Watcher watcher) {
    // private final HashMap<String, HashSet<Watcher>> watchTable = new HashMap<String,HashSet<Watcher>>();
    HashSet<Watcher> list = watchTable.get(path);
    // 判断watcherTable中是否存在当前路径对应的watcher
    if (list == null) {
        // don't waste memory if there are few watches on a node, rehash when the 4th entry is added, doubling size thereafter seems like a good compromise
        list = new HashSet<Watcher>(4);
        watchTable.put(path, list);
    }
    list.add(watcher);

    // private final HashMap<Watcher, HashSet<String>> watch2Paths = new HashMap<Watcher, HashSet<String>>();
    HashSet<String> paths = watch2Paths.get(watcher);
    if (paths == null) {
        // cnxns typically have many watches, so use default cap here
        paths = new HashSet<String>();
         // 设置watcher到节点路径的映射
        watch2Paths.put(watcher, paths);
    }
    // 将路径添加至paths集合
    paths.add(path);
}
```
>大致流程：  
①通过传入的path（节点路径）从watchTable获取相应的watcher集合，进入②；  
②判断①中的watcher是否为空，若为空则进入③，否则进入④；  
③新生成watcher集合，并将路径path和此集合添加至watchTable中，进入④；  
④将传入的watcher添加至watcher集合，即完成了path和watcher添加至watchTable的步骤，进入⑤；  
⑤通过传入的watcher从watch2Paths中获取相应的path集合，进入⑥；  
⑥判断path集合是否为空，若为空则进入⑦，否则进入⑧；  
⑦新生成path集合，并将watcher和paths添加至watch2Paths中，进入⑧；  
⑧将传入的path（节点路径）添加至path集合，即完成了path和watcher添加至watch2Paths的步骤。

- 调用链关系图


### 客户端接收服务端处理完成的响应
>ClientCnxnSocketNIO.doIO服务端处理完后，通过NettyServerCnxn.sendResponse（org.apache.zookeeper.server.NettyServerCnxn）发送返回的响应信息，客户端会在ClientCnxnSocketNIO.doIO接收服务端的返回。

```java 
// org.apache.zookeeper.ClientCnxnSocketNIO
void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn) throws InterruptedException, IOException {
    SocketChannel sock = (SocketChannel) sockKey.channel();
    if (sock == null) {
        throw new IOException("Socket is null!");
    }
    if (sockKey.isReadable()) {
        int rc = sock.read(incomingBuffer);
        if (rc < 0) {
            throw new EndOfStreamException("Unable to read additional data from server sessionid 0x" + Long.toHexString(sessionId) + ", likely server has closed socket");
        }
        if (!incomingBuffer.hasRemaining()) {
            incomingBuffer.flip();
            if (incomingBuffer == lenBuffer) {
                recvCount++;
                readLength();
            } else if (!initialized) {
                readConnectResult();
                enableRead();
                if (findSendablePacket(outgoingQueue, cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    enableWrite();
                }
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                updateLastHeard();
                initialized = true;
            } else {
                //【收到消息后触发SendThread.readResponse方法】
                sendThread.readResponse(incomingBuffer);
                // protected final ByteBuffer lenBuffer = ByteBuffer.allocateDirect(4);
                // protected ByteBuffer incomingBuffer = lenBuffer;
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                updateLastHeard();
            }
        }
    }
    if (sockKey.isWritable()) {
        synchronized(outgoingQueue) {
            Packet p = findSendablePacket(outgoingQueue,cnxn.sendThread.clientTunneledAuthenticationInProgress());

            if (p != null) {
                updateLastSend();
                // If we already started writing p, p.bb will already exist
                if (p.bb == null) {
                    if ((p.requestHeader != null) && (p.requestHeader.getType() != OpCode.ping) &&\ (p.requestHeader.getType() != OpCode.auth)) {
                        p.requestHeader.setXid(cnxn.getXid());
                    }
                    p.createBB();
                }
                sock.write(p.bb);
                if (!p.bb.hasRemaining()) {
                    sentCount++;
                    outgoingQueue.removeFirstOccurrence(p);
                    if (p.requestHeader != null && p.requestHeader.getType() != OpCode.ping && p.requestHeader.getType() != OpCode.auth) {
                        synchronized (pendingQueue) {
                            pendingQueue.add(p);
                        }
                    }
                }
            }
            if (outgoingQueue.isEmpty()) {
                // No more packets to send: turn off write interest flag.
                disableWrite();
            } else if (!initialized && p != null && !p.bb.hasRemaining()) {
                disableWrite();
            } else {
                // Just in case
                enableWrite();
            }
        }
    }
}
```
#### SendThread.readResponse
>该方法里主要流程：  
①读取header，若xid=-2，是一个ping的response，return；    
②若xid=-4，是一个AuthPacket的response，return；  
③若xid=-1，是一个notification，此时继续读取并构造一个enent，通过EventThread.queueEvent发送，return；  
④其它情况：从pendingQueue拿出一个Packet，校验后更新packet信息。

```java 
// org.apache.zookeeper.ClientCnxn
// class SendThread extends ZooKeeperThread {
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    ByteBufferInputStream bbis = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
    ReplyHeader replyHdr = new ReplyHeader();
    // 反序列化header
    replyHdr.deserialize(bbia, "header");
    if (replyHdr.getXid() == -2) {
        // -2 is the xid for pings
        return;
    }
    if (replyHdr.getXid() == -4) {
        // -4 is the xid for AuthPacket               
        if(replyHdr.getErr() == KeeperException.Code.AUTHFAILED.intValue()) {
            state = States.AUTH_FAILED;                    
            eventThread.queueEvent( new WatchedEvent(Watcher.Event.EventType.None,Watcher.Event.KeeperState.AuthFailed, null) );            		            		
        }
        return;
    }
    // 表示当前消息类型为notification(服务端的一个响应事件)
    if (replyHdr.getXid() == -1) {
        // -1 means notification
        WatcherEvent event = new WatcherEvent();
        // 反序列化响应信息
        event.deserialize(bbia, "response");
        // convert from a server path to a client path
        if (chrootPath != null) {
            String serverPath = event.getPath();
            if(serverPath.compareTo(chrootPath)==0)
                event.setPath("/");
            else if (serverPath.length() > chrootPath.length())
                event.setPath(serverPath.substring(chrootPath.length()));
            else {
            	LOG.warn("Got server path " + event.getPath()+ " which is too short for chroot path "+ chrootPath);
            }
        }

        WatchedEvent we = new WatchedEvent(event);
        eventThread.queueEvent( we );
        return;
    }
    if (clientTunneledAuthenticationInProgress()) {
        GetSASLRequest request = new GetSASLRequest();
        request.deserialize(bbia,"token");
        zooKeeperSaslClient.respondToServer(request.getToken(), ClientCnxn.this);
        return;
    }

    Packet packet;
    // private final LinkedList<Packet> pendingQueue = new LinkedList<Packet>();
    synchronized (pendingQueue) {
        if (pendingQueue.size() == 0) {
            throw new IOException("Nothing in the queue, but got " + replyHdr.getXid());
        }
        // 当前数据包已收到响应，故将其从pendingQueued移除
        packet = pendingQueue.remove();
    }
    try {
        // 校验数据包信息，成功后将数据包信息更新（替换为服务端信息）
        if (packet.requestHeader.getXid() != replyHdr.getXid()) {
            packet.replyHeader.setErr( KeeperException.Code.CONNECTIONLOSS.intValue());
            throw new IOException("Xid out of order. Got Xid "replyHdr.getXid() + " with err " ++ replyHdr.getErr() +" expected Xid "+ packet.requestHeader.getXid()+ " for a packet with details: "+ packet );
        }

        packet.replyHeader.setXid(replyHdr.getXid());
        packet.replyHeader.setErr(replyHdr.getErr());
        packet.replyHeader.setZxid(replyHdr.getZxid());
        if (replyHdr.getZxid() > 0) {
            lastZxid = replyHdr.getZxid();
        }
        if (packet.response != null && replyHdr.getErr() == 0) {
            // 获得服务端响应，反序列化后设置到packet.response属性中。可在exists方法最后一行通过packet.response拿到该请求返回结果
            packet.response.deserialize(bbia, "response");
        }
    } finally {
        // 最后调用【finishPacket】方法完成处理
        finishPacket(packet);
    }
}
```
#### finishPacket
>主要功能：从Packet中取出对应的Watcher并注册到ZKWatchManager。

```java 
// org.apache.zookeeper.ClientCnxn
private void finishPacket(Packet p) {
    if (p.watchRegistration != null) {
        // 将事件注册到zkwatchemanager中
        // org.apache.zookeeper.ClientCnxn.Packet->WatchRegistration watchRegistration;
        // 在组装请求时，初始化对象，把watchRegistration子类里面的Watcher实例放到ZKWatchManager的existsWatches中存储起来。
        // org.apache.zookeeper.ZooKeeper.ZKWatchManager
        // 【watchRegistration.register】
        p.watchRegistration.register(p.replyHeader.getErr());
    }
    // 若为null，表明是同步调用的接口，不需异步回调，直接notifyAll
    // AsyncCallback cb;
    if (p.cb == null) {
        synchronized (p) {
            p.finished = true;
            p.notifyAll();
        }
    } else {
        p.finished = true;
        // org.apache.zookeeper.ClientCnxn.EventThread
        // private final LinkedBlockingQueue<Object> waitingEvents = new LinkedBlockingQueue<Object>();
        // 将所有移除的监视事件添加到事件队列, 这样客户端能收到 “data/child 事件被移除”的事件类型
        // 【eventThread.queuePacket】
        eventThread.queuePacket(p);
    }
}
```
##### watchRegistration

```java 
// org.apache.zookeeper.ZooKeeper.WatchRegistration
public void register(int rc) {
    if (shouldAddWatch(rc)) {
        // 通过子类实现取得ZKWatchManager中的existsWatches
        // org.apache.zookeeper.ZooKeeper.ExistsWatchRegistration\ChildWatchRegistration\DataWatchRegistration
        Map<String, Set<Watcher>> watches = getWatches(rc);
        synchronized(watches) {
            Set<Watcher> watchers = watches.get(clientPath);
            if (watchers == null) {
                watchers = new HashSet<Watcher>();
                watches.put(clientPath, watchers);
            }
            // 将Watcher对象放到ZKWatchManager中的existsWatches
            // private final Map<String, Set<Watcher>> existWatches = new HashMap<String, Set<Watcher>>();
            watchers.add(watcher);
        }
    }
}
```
>以下为客户端存储watcher的几个map集合，分别对应三种注册监听事件。

```java 
// org.apache.zookeeper.ZooKeeper.ZKWatchManager
private static class ZKWatchManager implements ClientWatchManager {
    private final Map<String, Set<Watcher>> dataWatches = new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> existWatches = new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> childWatches = new HashMap<String, Set<Watcher>>();
    ......
```
>总结：当用ZooKeeper构造方法或用getData、exists和getChildren来向ZooKeeper服务器注册Watcher时，①首先将此消息传递给服务端，传递成功后，服务端会通知客户端；②然后客户端将该路径和Watcher对应关系存储起来备用。

##### EventThread.queuePacket
>finishPacket最终调用eventThread.queuePacket，将当前数据包添加到等待事件通知队列。
```java 
// org.apache.zookeeper.ClientCnxn.EventThread
public void queuePacket(Packet packet) {
  if (wasKilled) {
     // private final LinkedBlockingQueue<Object> waitingEvents = new LinkedBlockingQueue<Object>();
     synchronized (waitingEvents) {
        if (isRunning) waitingEvents.add(packet);
        else processEvent(packet);
     }
  } else {
     waitingEvents.add(packet);
  }
}
```
### 事件触发
>前面介绍了事件的注册流程，最终触发还需通过事务型操作完成。  
eg.zookeeper.setData("/test","1".getByte(),-1);  
//修改节点值触发监听

#### 服务端的事件响应DataTree
```java 
public Stat setData(String path, byte data[], int version, long zxid, long time) throws KeeperException.NoNodeException {
    Stat s = new Stat();
    DataNode n = nodes.get(path);
    if (n == null) {
        throw new KeeperException.NoNodeException();
    }
    byte lastdata[] = null;
    synchronized (n) {
        lastdata = n.data;
        n.data = data;
        n.stat.setMtime(time);
        n.stat.setMzxid(zxid);
        n.stat.setVersion(version);
        n.copyStat(s);
    }
    // now update if the path is in a quota subtree.
    String lastPrefix;
    if((lastPrefix = getMaxPrefixWithQuota(path)) != null) {
      this.updateBytes(lastPrefix, (data == null ? 0 : data.length) - (lastdata == null ? 0 : lastdata.length));
    }
    // 触发对应节点的NodeDataChanged事件
    dataWatches.triggerWatch(path, EventType.NodeDataChanged);
    return s;
}
```
##### WatcherManager.triggerWatch
```java 
// org.apache.zookeeper.server.WatchManager
public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    // 根据事件类型、连接状态、节点路径创建 WatchedEvent
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        // 从watcher表中移除path，并返回其对应的watcher集合
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) {
            return null;
        }
        // 遍历watcher集合
        for (Watcher w : watchers) {
            // 根据watcher从watcher表中取出路径集合
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                // 移除路径
                paths.remove(path);
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        // 【process】
        w.process(e);
    }
    return watchers;
}
```
##### Watcher.process(WatchedEvent e);
>在服务端绑定事件时，watcher绑定了ServerCnxn，故w.process(e)实际调用ServerCnxn的process。而 servercnxn是一个抽象方法，有两个实现类：NIOServerCnxn 和 NettyServerCnxn。 

```java 
public void process(WatchedEvent event) {
    ReplyHeader h = new ReplyHeader(-1, -1L, 0);
    // Convert WatchedEvent to a type that can be sent over the wire
    WatcherEvent e = event.getWrapper();

    try {
        // 发送了一个事件，事件对象为WatcherEvent
        sendResponse(h, e, "notification");
    } catch (IOException e1) {
        close();
    }
}
```
>接下来，客户端会收到该response，触发SendThread.readResponse。

#### 客户端处理事件响应
##### SendThread.readResponse

```java 
// org.apache.zookeeper.ClientCnxn.SendThread
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    ByteBufferInputStream bbis = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
    ReplyHeader replyHdr = new ReplyHeader();

    replyHdr.deserialize(bbia, "header");
    if (replyHdr.getXid() == -2) {
        // -2 is the xid for pings
        return;
    }
    if (replyHdr.getXid() == -4) {
        // -4 is the xid for AuthPacket               
        if(replyHdr.getErr() == KeeperException.Code.AUTHFAILED.intValue()) {
            state = States.AUTH_FAILED;                    
            eventThread.queueEvent( new WatchedEvent(Watcher.Event.EventType.None, Watcher.Event.KeeperState.AuthFailed, null) );            		            		
        }
        return;
    }
    if (replyHdr.getXid() == -1) {
        // -1 means notification
        WatcherEvent event = new WatcherEvent();
        // 【反序列化服务端的WatcherEvent事件】
        event.deserialize(bbia, "response");

        // convert from a server path to a client path
        if (chrootPath != null) {
            String serverPath = event.getPath();
            if(serverPath.compareTo(chrootPath)==0)
                event.setPath("/");
            else if (serverPath.length() > chrootPath.length())
                event.setPath(serverPath.substring(chrootPath.length()));
            else {
            	LOG.warn("Got server path " + event.getPath()+ " which is too short for chroot path "+ chrootPath);
            }
        }
        // 组装watchedEvent对象
        WatchedEvent we = new WatchedEvent(event);
        // 【通过eventTherad进行事件处理】
        eventThread.queueEvent( we );
        return;
    }

    if (clientTunneledAuthenticationInProgress()) {
        GetSASLRequest request = new GetSASLRequest();
        request.deserialize(bbia,"token");
        zooKeeperSaslClient.respondToServer(request.getToken(),ClientCnxn.this);
        return;
    }

    Packet packet;
    synchronized (pendingQueue) {
        if (pendingQueue.size() == 0) {
            throw new IOException("Nothing in the queue, but got "+ replyHdr.getXid());
        }
        packet = pendingQueue.remove();
    }
    try {
        if (packet.requestHeader.getXid() != replyHdr.getXid()) {
            packet.replyHeader.setErr(KeeperException.Code.CONNECTIONLOSS.intValue());
            throw new IOException("Xid out of order. Got Xid "+ replyHdr.getXid() + " with err " + replyHdr.getErr() +" expected Xid "+ packet.requestHeader.getXid()+ " for a packet with details: "+ packet );
        }

        packet.replyHeader.setXid(replyHdr.getXid());
        packet.replyHeader.setErr(replyHdr.getErr());
        packet.replyHeader.setZxid(replyHdr.getZxid());
        if (replyHdr.getZxid() > 0) {
            lastZxid = replyHdr.getZxid();
        }
        if (packet.response != null && replyHdr.getErr() == 0) {
            packet.response.deserialize(bbia, "response");
        }
    } finally {
        finishPacket(packet);
    }
}
```
#### eventThread.queueEvent
>SendThread收到服务端的通知事件后，会调用EventThread.queueEvent将事件传给EventThread线程，queueEvent方法根据该通知事件，从ZKWatchManager中取出所有相关的Watcher，若获取到相应的Watcher，会让Watcher移除失效。

```java 
// org.apache.zookeeper.ClientCnxn.EventThread
public void queueEvent(WatchedEvent event) {
    // 判断类型
    if (event.getType() == EventType.None && sessionState == event.getState()) {
        return;
    }
    sessionState = event.getState();
    //封装WatcherSetEventPair对象，添加到waitngEvents队列中
    // materialize the watchers based on the event
    // 【Meterialize方法】 private final ClientWatchManager watcher;
    WatcherSetEventPair pair = new WatcherSetEventPair(watcher.materialize(event.getState(),event.getType(),event.getPath()),event);
    // queue the pair (watch set & event) for later processing
    // private final LinkedBlockingQueue<Object> waitingEvents = new LinkedBlockingQueue<Object>();
    // 【waitingEvents.add】
    waitingEvents.add(pair);
}
```
#### Meterialize方法
>通过dataWatches或existWatches或childWatches remove出对应watch，表明客户端watch也是注册一次就移除
同时需根据keeperState、eventType和path返回应被通知的Watcher集合。  
```java 
// org.apache.zookeeper.ZooKeeper.ZKWatchManager
public Set<Watcher> materialize(Watcher.Event.KeeperState state, Watcher.Event.EventType type, String clientPath) {
    /*private final Map<String, Set<Watcher>> dataWatches = new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> existWatches = new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> childWatches = new HashMap<String, Set<Watcher>>();
    private volatile Watcher defaultWatcher;*/
    Set<Watcher> result = new HashSet<>();

    switch (type) {
    case None:
        result.add(defaultWatcher);
        boolean clear = ClientCnxn.getDisableAutoResetWatch() && state != Watcher.Event.KeeperState.SyncConnected;

        synchronized (dataWatches) {
            for (Set<Watcher> ws : dataWatches.values()) {
                result.addAll(ws);
            }
            if (clear) {
                dataWatches.clear();
            }
        }

        synchronized (existWatches) {
            for (Set<Watcher> ws : existWatches.values()) {
                result.addAll(ws);
            }
            if (clear) {
                existWatches.clear();
            }
        }

        synchronized (childWatches) {
            for (Set<Watcher> ws : childWatches.values()) {
                result.addAll(ws);
            }
            if (clear) {
                childWatches.clear();
            }
        }

        return result;
    case NodeDataChanged:
    case NodeCreated:
        synchronized (dataWatches) {
            addTo(dataWatches.remove(clientPath), result);
        }
        synchronized (existWatches) {
            addTo(existWatches.remove(clientPath), result);
        }
        break;
    case NodeChildrenChanged:
        synchronized (childWatches) {
            addTo(childWatches.remove(clientPath), result);
        }
        break;
    case NodeDeleted:
        synchronized (dataWatches) {
            addTo(dataWatches.remove(clientPath), result);
        }
        // XXX This shouldn't be needed, but just in case
        synchronized (existWatches) {
            Set<Watcher> list = existWatches.remove(clientPath);
            if (list != null) {
                addTo(list, result);
                LOG.warn("We are triggering an exists watch for delete! Shouldn't happen!");
            }
        }
        synchronized (childWatches) {
            addTo(childWatches.remove(clientPath), result);
        }
        break;
    default:
        String msg = "Unhandled watch event type " + type + " with state " + state + " on path " + clientPath;
        LOG.error(msg);
        throw new RuntimeException(msg);
    }

    return result;
}
}
```
#### waitingEvents.add
>waitingEvents是EventThread线程的阻塞队列。waitingEvents是一个待处理Watcher队列，EventThread的run() 方法不断从队列中取数据，交由processEvent方法处理。
```java 
// org.apache.zookeeper.ClientCnxn.EventThread
public void run() {
    try {
        isRunning = true;
        while (true) {
            Object event = waitingEvents.take();
            if (event == eventOfDeath) {
                wasKilled = true;
            } else {
                // 【processEvent方法】
                processEvent(event);
            }
            if (wasKilled)
                synchronized (waitingEvents) {
                    if (waitingEvents.isEmpty()) {
                        isRunning = false;
                        break;
                    }
                }
        }
    } catch (InterruptedException e) {
        LOG.error("Event thread exiting due to interruption", e);
    }

    LOG.info("EventThread shut down for session: 0x{}", Long.toHexString(getSessionId()));
}
```
##### ProcessEvent
>只贴核心代码
```java 
// org.apache.zookeeper.ClientCnxn.EventThread
private void processEvent(Object event) {
   try {
       // 判断事件类型
       if (event instanceof WatcherSetEventPair) {
           // each watcher will process the event
           // 得到watcherseteventPair
           WatcherSetEventPair pair = (WatcherSetEventPair) event;
           // 拿到符合触发机制的所有watcher列表，循环调用
           for (Watcher watcher : pair.watchers) {
               try {
                    // 调用客户端回调process
                   watcher.process(pair.event);
               } catch (Throwable t) {
                   LOG.error("Error while calling watcher ", t);
               }
           }
       } else {
       ......
```
