->**zookeeper.java**

```
 public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly) throws IOException {
        LOG.info("Initiating client connection, connectString=" + connectString
                + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);
        watchManager.defaultWatcher = watcher;
        // 设置defaultWatcher 为 MyWatcher
        ConnectStringParser connectStringParser = new ConnectStringParser(
        connectString);
        // 解析-server 获取 IP以及PORT
        HostProvider hostProvider = new StaticHostProvider(
        connectStringParser.getServerAddresses());
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),hostProvider, sessionTimeout,
        this, watchManager,getClientCnxnSocket(), canBeReadOnly);
        // 创建 ClientCnxn实例
        cnxn.start(); // 启动cnxn中的SendThread and EventThread进程
    }
    
     public void start() {
        sendThread.start();
        eventThread.start();
    }
    
    private static ClientCnxnSocket getClientCnxnSocket() throws IOException {
        String clientCnxnSocketName = System
                .getProperty(ZOOKEEPER_CLIENT_CNXN_SOCKET);
        if (clientCnxnSocketName == null) {
            clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
        }
        try {
            return (ClientCnxnSocket) Class.forName(clientCnxnSocketName)
                    .newInstance();
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate "
                    + clientCnxnSocketName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```

ClientCnxn内部类SendThread->**startConnect()**

```
private void startConnect() throws IOException {
            state = States.CONNECTING;//state 状态设置为连接
            InetSocketAddress addr;
            if (rwServerAddress != null) {
                addr = rwServerAddress;
                rwServerAddress = null;
            } else {
                addr = hostProvider.next(1000);
            }
                setName(getName().replaceAll("\\(.*\\)",
                    "(" + addr.getHostName() + ":" + addr.getPort() + ")"));
            if (ZooKeeperSaslClient.isEnabled()) {
                try {
                    String principalUserName = System.getProperty(
                            ZK_SASL_CLIENT_USERNAME, "zookeeper");
                    zooKeeperSaslClient =
                        new ZooKeeperSaslClient(
                                principalUserName+"/"+addr.getHostName());
                } catch (LoginException e) {                    
                   }
            }
            logStartConnect(addr);//写连接日志
            clientCnxnSocket.connect(addr); //连接Socket
        }
```
ClientCnxnSocketNIO->**connect(InetSocketAddress addr)**

```
 void connect(InetSocketAddress addr) throws IOException {
        SocketChannel sock = createSock();// 创建一个非阻塞空SocketChannel
        try {
           registerAndConnect(sock, addr);//注册并连接socket到addr
        } catch (IOException e) {
            LOG.error("Unable to open socket to " + addr);
            sock.close();
            throw e;
        }
        initialized = false;
        lenBuffer.clear();
        incomingBuffer = lenBuffer;
    }
```
**registerAndConnect(sock, addr)**
```
 void registerAndConnect(SocketChannel sock, InetSocketAddress addr) 
    throws IOException {
        sockKey = sock.register(selector, SelectionKey.OP_CONNECT); //将socket注册到selector中
        boolean immediateConnect = sock.connect(addr); //socket连接服务器
        if (immediateConnect) {
            sendThread.primeConnection(); //初始化连接事件
        }
    }
```

**primeConnection()**

```
void primeConnection() throws IOException {
            isFirstConnect = false;// 设置为非首次连接
            long sessId = (seenRwServerBefore) ? sessionId : 0;// 客户端默认sessionid为0
            ConnectRequest conReq = new ConnectRequest(0, lastZxid,
                    sessionTimeout, sessId, sessionPasswd);
                     // 创建连接request lastZxid 代表最新一次的节点ZXID
			// 线程安全占用outgoing 
            synchronized (outgoingQueue) {     
            //组合成通讯层的Packet对象，添加到发送队列，对于ConnectRequest其requestHeader为null  
                            Packet packet = new Packet(h, new ReplyHeader(), sw, null, null);
                            outgoingQueue.addFirst(packet);
                        }
                    }
               }
            }
            //确保读写事件都监听，也就是设置成可读可写
            clientCnxnSocket.enableReadWriteOnly();            
        }
```
run()->**clientCnxnSocket.doTransport(to, pendingQueue, outgoingQueue, ClientCnxn.this)**

```
void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue,
                     ClientCnxn cnxn)
            throws IOException, InterruptedException {
        selector.select(waitTimeOut);
        Set<SelectionKey> selected;
        synchronized (this) {
            selected = selector.selectedKeys();
        }
        updateNow();
        for (SelectionKey k : selected) {
            SocketChannel sc = ((SocketChannel) k.channel());
            //如果之前连接没有立马连上，则在这里处理OP_CONNECT事件
            if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
                if (sc.finishConnect()) {
                    updateLastSendAndHeard();
                    sendThread.primeConnection();
                }
                 //如果读写就位，则处理之
            } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                doIO(pendingQueue, outgoingQueue, cnxn);
            }
        }
        if (sendThread.getZkState().isConnected()) {
            synchronized(outgoingQueue) {
             	//找到连接Packet并且将他放到队列头
                if (findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                     //  将要Channecl设置为可写
                    enableWrite();
                }
            }
        }
        selected.clear();
    }
```

**findSendablePacket(outgoingQueue,cnxn.sendThread.clientTunneledAuthenticationInProgress())**

```
private Packet findSendablePacket(LinkedList<Packet> outgoingQueue,
                                      boolean clientTunneledAuthenticationInProgress) {
        synchronized (outgoingQueue) {
            //因为Conn Packet需要发送到SASL authentication进行处理，其他Packet都需要等待直到该处理完成，
            //Conn Packet必须第一个处理，所以找出它并且把它放到OutgoingQueue头,也就是requestheader=null的辣个
            ListIterator<Packet> iter = outgoingQueue.listIterator();
            while (iter.hasNext()) {
                Packet p = iter.next();
                if (p.requestHeader == null) {
                    // We've found the priming-packet. Move it to the beginning of the queue.
                    iter.remove();  
                    outgoingQueue.add(0, p); // 将连接放到outgogingQueue第一个
                    return p;
                } else {
                    // Non-priming packet: defer it until later, leaving it in the queue
                    // until authentication completes.
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
```

**IO write**->**doIO(pendingQueue, outgoingQueue, cnxn);**

```
if (sockKey.isWritable()) {
            synchronized(outgoingQueue) {
                // 获得packet
                Packet p = findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress());
                if (p != null) {
                    updateLastSend();
                    // If we already started writing p, p.bb will already exist
                    if (p.bb == null) {
                        if ((p.requestHeader != null) &&
                                (p.requestHeader.getType() != OpCode.ping) &&
                                (p.requestHeader.getType() != OpCode.auth)) {
                            //如果不是 连接事件，不是ping 事件，不是 认证时间 
                            p.requestHeader.setXid(cnxn.getXid());
                        }
                            // 序列化
                        p.createBB();
                    }
                   //将数据写入Channel
                    sock.write(p.bb);
                    // p.bb中如果没有内容 则表示发送成功
                    if (!p.bb.hasRemaining()) {
                       //发送数+1
                        sentCount++;
                        //将该P从队列中移除
                        outgoingQueue.removeFirstOccurrence(p);
                        //如果该事件不是连接事件，不是ping事件，不是认证事件， 则将他加入pending队列中
                        if (p.requestHeader != null
                                && p.requestHeader.getType() != OpCode.ping
                                && p.requestHeader.getType() != OpCode.auth) {
                            synchronized (pendingQueue) {
                                pendingQueue.add(p);
                            }
                        }
                    }
                }
                if (outgoingQueue.isEmpty()) {
                    // No more packets to 
                    disableWrite();
                } else if (!initialized && p != null && !p.bb.hasRemaining()) {
                    disableWrite();
                } else {
                    enableWrite();
                }
            }
        }
```

**p.createBB();->createBB()**

```
public void createBB() {
            try {
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                boa.writeInt(-1, "len"); // We'll fill this in later
                // 如果不是连接事件则设置协议头
                if (requestHeader != null) {
                    requestHeader.serialize(boa, "header");
                }
                //设置协议体
                if (request instanceof ConnectRequest) {
                    request.serialize(boa, "connect");
                    // append "am-I-allowed-to-be-readonly" flag
                    boa.writeBool(readOnly, "readOnly");
                } else if (request != null) {
                    request.serialize(boa, "request");
                }
                baos.close();
                //生成ByteBuffer
                this.bb = ByteBuffer.wrap(baos.toByteArray());
              //将bytebuffer的前4个字节修改成真正的长度，总长度减去一个int的长度头 
                this.bb.putInt(this.bb.capacity() - 4);
              //准备给后续读    让buffer position = 0
                this.bb.rewind();
            } catch (IOException e) {
                LOG.warn("Ignoring unexpected exception", e);
            }
        }
```

**IO read**

```
if (sockKey.isReadable()) {
            //先从Channel读4个字节，代表头  
            int rc = sock.read(incomingBuffer);
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from server sessionid 0x"
                                + Long.toHexString(sessionId)
                                + ", likely server has closed socket");
            }
            if (!incomingBuffer.hasRemaining()) {
                incomingBuffer.flip();
                if (incomingBuffer == lenBuffer) {
                    recvCount++;
                    readLength();
                } 
                //初始化
                else if (!initialized) {
                    readConnectResult();  // 读取连接结果
                    enableRead(); // Channel 可读
                    if (findSendablePacket(outgoingQueue,
                            cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                        // Since SASL authentication has completed (if client is configured to do so),
                        // outgoing packets waiting in the outgoingQueue can now be sent.
                        enableWrite();
                    }
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                    initialized = true;
                } else {    
                    // 处理其他请求
                    sendThread.readResponse(incomingBuffer);
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                }
            }
        }
```

**sendThread.readResponse(incomingBuffer);->readResponse**

readResponse函数，用来消费PendingQueue,处理的消息分为三类 
 ping 消息 XID=-2 
 auth认证消息 XID=-4 
 订阅的消息，即各种变化的通知，比如子节点变化、节点内容变化，由服务器推过来的消息 ，获取到这类消息或通过eventThread.queueEvent将消息推入事件队列 
XID=-1



结束了IO之后就是对于事件的消费，也就是一开始图示的右半部分也是接近最后部分

```
public void run() {
           try {
              isRunning = true;
              while (true) {
                  // 获取事件
                 Object event = waitingEvents.take();
                 if (event == eventOfDeath) {
                    wasKilled = true;
                 } else {
                     //处理事件
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

            LOG.info("EventThread shut down");
        }
        }
}
```

