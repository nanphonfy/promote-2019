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
### 事件机制
>Watcher监听机制是很重要的特性，基于ZK创建的节点，可对这些节点绑定监听事件，eg.可监听节点数据变更、节点删除、子节点状态变更等事件，可基于ZK实现分布式锁、集群管理等功能。  
- watcher特性
>当数据变化时，ZK会产生一个watcher事件，发送到客户端，但客户端只会收到一次通知。若后续该节点再次变化，那之前设置watcher的客户端不会再收到消息（watcher是一次性操作）。可通过循环监听达到永久监听效果。

```java 
public static void main(String[] args) throws KeeperException {
    String connectString = "192.168.25.154:2181";
    String createPath = "/zk-persis-np";
    try {
        final CountDownLatch countDownLatch = new CountDownLatch(1);
        final ZooKeeper zooKeeper = new ZooKeeper(connectString, 4000, new Watcher() {
            @Override public void process(WatchedEvent event) {
                System.out.println("全局默认事件:"+event.getType());
                if (Event.KeeperState.SyncConnected == event.getState()) {
                    //若收到了服务端的响应事件，连接成功
                    countDownLatch.countDown();
                }
            }
        });
        countDownLatch.await();
        // CONNECTING
        System.out.println("zk state:"+zooKeeper.getState());
        // test1(zooKeeper,createPath);
        // 通过exists绑定事件
        Stat stat2 = zooKeeper.exists(createPath, new Watcher() {
            @Override public void process(WatchedEvent watchedEvent) {
                System.out.println(watchedEvent.getType()+"->"+watchedEvent.getPath());
                try {
                    // 再次绑定事件(第二次)
                    zooKeeper.exists(watchedEvent.getPath(),true);
                } catch (KeeperException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        // 通过修改事务类型操作，触发监听事件
        stat2 = zooKeeper.setData(createPath, "4".getBytes(), stat2.getVersion());
        stat2 = zooKeeper.setData(createPath, "3".getBytes(), stat2.getVersion());
        // watch只能响应一次，可通过循环监听达到永久监听效果
        //delete(zooKeeper, createPath, stat2);
        zooKeeper.close();
        System.in.read();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```
- watcher事件类型

类型 | 备注
---|---
None(-1) | 客户端链接状态发生变化时，会收到 none事件
NodeCreated(1) | 创建节点事件
NodeDeleted(2) | 删除节点的事件
NodeDataChanged(3) | 节点数据发生变更
NodeChildrenChanged(4) | 子节点被创建、删除、会发生事件触发

- 操作->事件类型

操作 | 事件类型 | 事件类型 
---|---|---
create(/zk-persis-np) | NodeCreated(exists/getData) | 无
delete(/zk-persis-np) | NodeDeleted(exists/getData) | 无
setData(/zk-persis-np) | NodeDataChanged(exists/getData) | 无
create(/zk-persis-np/children) | NodeChildrenChanged(getChild) | NodedCreated
delete(/zk-persis-np/children) | NodeChildrenChanged(getChild) | NodedDeleted
setData(/zk-persis-np/children)  | - | NodedDataChanged
