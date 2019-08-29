#### 服务端接收请求处理流程
>服务端有一个NettyServerCnxn类，用来处理客户端发送请求。

```java 
// org.apache.zookeeper.server.NettyServerCnxn
public void receiveMessage(ChannelBuffer message) {
    while(true) {
        try {
            if(message.readable() && !this.throttled) {
                ByteBuffer e;
                int e1;
                // ByteBuffer不为空
                if(this.bb != null) {
                    if(this.LOG.isTraceEnabled()) {
                        this.LOG.trace("message readable " + message.readableBytes() + " bb len " + this.bb.remaining() + " " + this.bb);
                        e = this.bb.duplicate();
                        e.flip();
                        this.LOG.trace(Long.toHexString(this.sessionId) + " bb 0x" + ChannelBuffers.hexDump(ChannelBuffers.copiedBuffer(e)));
                    }

                    // bb剩余空间大于message 中可读字节大小
                    if(this.bb.remaining() > message.readableBytes()) {
                        e1 = this.bb.position() + message.readableBytes();
                        this.bb.limit(e1);
                    }

                   // 将message写入bb中 message.readBytes(this.bb);
                    this.bb.limit(this.bb.capacity());
                    if(this.LOG.isTraceEnabled()) {
                        this.LOG.trace("after readBytes message readable " + message.readableBytes() + " bb len " + this.bb.remaining() + " " + this.bb);
                        e = this.bb.duplicate();
                        e.flip();
                        this.LOG.trace("after readbytes " + Long.toHexString(this.sessionId) + " bb 0x" + ChannelBuffers.hexDump(ChannelBuffers.copiedBuffer(e)));
                    }
                    // 已读完message
                    if(this.bb.remaining() != 0) {
                        continue;
                    }
                    // 统计接收信息
                    this.packetReceived();
                    this.bb.flip();
                    ZooKeeperServer e2 = this.zkServer;
                    if(e2 != null && e2.isRunning()) {
                        if(this.initialized) {
                            // 处理客户端传过来的数据包
                            e2.processPacket(this, this.bb);
                            if(e2.shouldThrottle(this.outstandingCount.incrementAndGet())) {
                                this.disableRecvNoWait();
                            }
                        } else {
                            this.LOG.debug("got conn req request from " + this.getRemoteSocketAddress());
                            e2.processConnectRequest(this, this.bb);
                            this.initialized = true;
                        }

                        this.bb = null;
                        continue;
                    }

                    throw new IOException("ZK down");
                }

                if(this.LOG.isTraceEnabled()) {
                    this.LOG.trace("message readable " + message.readableBytes() + " bblenrem " + this.bbLen.remaining());
                    e = this.bbLen.duplicate();
                    e.flip();
                    this.LOG.trace(Long.toHexString(this.sessionId) + " bbLen 0x" + ChannelBuffers.hexDump(ChannelBuffers.copiedBuffer(e)));
                }

                if(message.readableBytes() < this.bbLen.remaining()) {
                    this.bbLen.limit(this.bbLen.position() + message.readableBytes());
                }

                message.readBytes(this.bbLen);
                this.bbLen.limit(this.bbLen.capacity());
                if(this.bbLen.remaining() != 0) {
                    continue;
                }

                this.bbLen.flip();
                if(this.LOG.isTraceEnabled()) {
                    this.LOG.trace(Long.toHexString(this.sessionId) + " bbLen 0x" + ChannelBuffers.hexDump(ChannelBuffers.copiedBuffer(this.bbLen)));
                }

                e1 = this.bbLen.getInt();
                if(this.LOG.isTraceEnabled()) {
                    this.LOG.trace(Long.toHexString(this.sessionId) + " bbLen len is " + e1);
                }

                this.bbLen.clear();
                if(!this.initialized && this.checkFourLetterWord(this.channel, message, e1)) {
                    return;
                }

                if(e1 >= 0 && e1 <= BinaryInputArchive.maxBuffer) {
                    this.bb = ByteBuffer.allocate(e1);
                    continue;
                }

                throw new IOException("Len error " + e1);
            }
        } catch (IOException var3) {
            this.LOG.warn("Closing connection to " + this.getRemoteSocketAddress(), var3);
            this.close();
        }

        return;
    }
}
```

- ZookeeperServer——e2.processPacket(this, this.bb);  
>处理客户端传送过来的数据包。
```java 
// org.apache.zookeeper.server.ZooKeeperServer
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    ByteBufferInputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    // 反序列化客户端header头信息
    h.deserialize(bia, "header");
    incomingBuffer = incomingBuffer.slice();
    // 判断当前操作类型
    if(h.getType() == OpCode.auth) {
        LOG.info("got auth packet " + cnxn.getRemoteSocketAddress());
        AuthPacket si2 = new AuthPacket();
        ByteBufferInputStream.byteBuffer2Record(incomingBuffer, si2);
        String rh2 = si2.getScheme();
        AuthenticationProvider ap = ProviderRegistry.getProvider(rh2);
        Code authReturn = Code.AUTHFAILED;
        if(ap != null) {
            try {
                authReturn = ap.handleAuthentication(cnxn, si2.getAuth());
            } catch (RuntimeException var11) {
                LOG.warn("Caught runtime exception from AuthenticationProvider: " + rh2 + " due to " + var11);
                authReturn = Code.AUTHFAILED;
            }
        }

        ReplyHeader rh1;
        if(authReturn != Code.OK) {
            if(ap == null) {
                LOG.warn("No authentication provider for scheme: " + rh2 + " has " + ProviderRegistry.listProviders());
            } else {
                LOG.warn("Authentication failed for scheme: " + rh2);
            }

            rh1 = new ReplyHeader(h.getXid(), 0L, Code.AUTHFAILED.intValue());
            cnxn.sendResponse(rh1, (Record)null, (String)null);
            cnxn.sendBuffer(ServerCnxnFactory.closeConn);
            cnxn.disableRecv();
        } else {
            if(LOG.isDebugEnabled()) {
                LOG.debug("Authentication succeeded for scheme: " + rh2);
            }

            LOG.info("auth success " + cnxn.getRemoteSocketAddress());
            rh1 = new ReplyHeader(h.getXid(), 0L, Code.OK.intValue());
            cnxn.sendResponse(rh1, (Record)null, (String)null);
        }

    } else {
        // 如果不是授权操作，再判断是否为 sasl 操作
        if(h.getType() == OpCode.sasl) {
            // 封装请求对象
            Record si = this.processSasl(incomingBuffer, cnxn);
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0L, Code.OK.intValue());
            cnxn.sendResponse(rh, si, "response");
        } else {
            Request si1 = new Request(cnxn, cnxn.getSessionId(), h.getXid(), h.getType(), incomingBuffer, cnxn.getAuthInfo());
            si1.setOwner(ServerCnxn.me);
            // 提交请求
            this.submitRequest(si1);
        }

        cnxn.incrOutstandingRequests(h);
    }
}
```
