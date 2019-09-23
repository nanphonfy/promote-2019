### 数据存储
>事务日志  
快照日志  
运行时日志：bin/zookeeper.out

### java使用zookeeper
#### 建立连接与增删改查
```java 
public class JavaClient {
    public static void main(String[] args) throws KeeperException {
        String connectString = "192.168.25.154:2181";
        String createPath = "/zk-persis-np";
        try {
            final CountDownLatch countDownLatch = new CountDownLatch(1);
            ZooKeeper zooKeeper = new ZooKeeper(connectString, 4000, new Watcher() {
                @Override public void process(WatchedEvent event) {
                    if (Event.KeeperState.SyncConnected == event.getState()) {
                        //若收到了服务端的响应事件，连接成功
                        countDownLatch.countDown();
                    }
                }
            });
            countDownLatch.await();
            // CONNECTING
            System.out.println(zooKeeper.getState());

            // 数据增删改查
            Stat stat = new Stat();
            // 添加节点
            create(zooKeeper, createPath, "11");
            // 得到当前节点值
            search(zooKeeper,createPath,stat);
            // 修改节点值
            update(zooKeeper,createPath,stat,"069");
            search(zooKeeper, createPath, stat);
            //delete(zooKeeper, createPath, stat);

            zooKeeper.close();
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static void create(ZooKeeper zooKeeper, String path, String value) throws InterruptedException {
        try {
            zooKeeper.create(path, value.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    private static void search(ZooKeeper zooKeeper, String path, Stat stat)
            throws KeeperException, InterruptedException {
        Thread.sleep(1000);
        // 得到当前节点的值
        byte[] bytes = new byte[0];
        bytes = zooKeeper.getData(path, null, stat);
        System.out.println(new String(bytes));
    }

    private static void update(ZooKeeper zooKeeper, String path, Stat stat, String newValue)
            throws KeeperException, InterruptedException {
        zooKeeper.setData(path, newValue.getBytes(), stat.getVersion());
    }

    private static void delete(ZooKeeper zooKeeper, String path, Stat stat)
            throws KeeperException, InterruptedException {
        zooKeeper.delete(path, stat.getVersion());
    }
}
```

