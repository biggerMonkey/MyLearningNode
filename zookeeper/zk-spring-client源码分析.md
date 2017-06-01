 ```
 public ZKClient(String zkServers, String namespace, int connectionTimeoutMs,
            int sessionTimeoutMs) {
        this.client =CuratorFrameworkFactory.builder().connectString(zkServers)
        .namespace(namespace).retryPolicy(retryPolicy)
        .connectionTimeoutMs(connectionTimeoutMs)
        .sessionTimeoutMs(sessionTimeoutMs).build();
        this.client.start();
    }
 ```

***build();***

CuratorFrameworkFactory->***build();***

***ZookeeperFactory localZookeeperFactory = makeZookeeperFactory(builder.getZookeeperFactory());***

->

***ZooKeeper zooKeeper = actualZookeeperFactory.newZooKeeper(connectString, sessionTimeout, watcher, canBeReadOnly);***

DefaultZookeeperFactory->

***new ZooKeeper(connectString, sessionTimeout, watcher, canBeReadOnly);***

->zookeeper构造器