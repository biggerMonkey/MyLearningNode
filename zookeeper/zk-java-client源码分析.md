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

