
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
    // 对于exists请求，需监听data变化事件，添加watcher
    Stat stat = zks.getZKDatabase().statNode(path, existsRequest.getWatch() ? cnxn : null);
    // 在服务端内存数据库中根据路径得到结果进行组装，设置为ExistsResponse
    rsp = new ExistsResponse(stat);
    break;
}
```