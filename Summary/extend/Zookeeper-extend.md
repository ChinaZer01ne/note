# Zookeeper环境搭建

环境搭建，略
# Java调用Zookeeper
## 原生API

Zookeeper作为⼀个分布式框架，主要⽤来解决分布式⼀致性问题，它提供了简单的分布式原语，并且对多种编程语⾔提供了API，所以接下来重点来看下Zookeeper的java客户端API使⽤⽅式。

Zookeeper API共包含五个包，分别为：

- org.apache.zookeeper
- org.apache.zookeeper.data
- org.apache.zookeeper.server
- org.apache.zookeeper.server.quorum
- org.apache.zookeeper.server.upgrade

其中org.apache.zookeeper，包含Zookeeper类，他是我们编程时最常⽤的类⽂件。这个类是Zookeeper客户端的主要类⽂件。如果要使⽤Zookeeper服务，应⽤程序⾸先必须创建⼀个Zookeeper实例，这时就需要使⽤此类。⼀旦客户端和Zookeeper服务端建⽴起了连接， Zookeeper系统将会给本次连接会话分配⼀个ID值，并且客户端将会周期性的向服务器端发送⼼跳来维持会话连接。只要连接有效，客户端就可以使⽤Zookeeper API来做相应处理了。

### 导⼊依赖

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.14</version>
</dependency>
```

### 建⽴会话

```java
public class CreateSession implements Watcher {
    //countDownLatch这个类使⼀个线程等待,主要不让main⽅法结束
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
    public static void main(String[] args) throws InterruptedException, IOException {
        /*
            客户端可以通过创建⼀个zk实例来连接zk服务器
            new Zookeeper(connectString,sesssionTimeOut,Wather)
            connectString: 连接地址： IP：端⼝
            sesssionTimeOut：会话超时时间：单位毫秒
            Wather：监听器(当特定事件触发监听时， zk会通过watcher通知到客户端)
        */
        ZooKeeper zooKeeper = new ZooKeeper("10.211.55.4:2181", 5000, new CreateSession());
        System.out.println(zooKeeper.getState());
        countDownLatch.await();
        //表示会话真正建⽴
        System.out.println("=========Client Connected to zookeeper==========");
    }
    
    // 当前类实现了Watcher接⼝，重写了process⽅法，该⽅法负责处理来⾃Zookeeper服务端的watcher通知，在收到服务端发送过来的SyncConnected事件之后，解除主程序在CountDownLatch上的等待阻塞，⾄此，会话创建完毕
    @Override
    public void process(WatchedEvent watchedEvent) {
        //当连接创建了，服务端发送给客户端SyncConnected事件
        if(watchedEvent.getState() == Event.KeeperState.SyncConnected){
            countDownLatch.countDown();
        }
    }
}
```

注意， ZooKeeper 客户端和服务端会话的建⽴是⼀个异步的过程，也就是说在程序中，构造⽅法会在处理完客户端初始化⼯作后⽴即返回，在⼤多数情况下，此时并没有真正建⽴好⼀个可⽤的会话，在会话的⽣命周期中处于“CONNECTING”的状态。 当该会话真正创建完毕后ZooKeeper服务端会向会话对应的客户端发送⼀个事件通知，以告知客户端，客户端只有在获取这个通知之后，才算真正建⽴了会话。

### 创建节点

```java
public class CreateNote implements Watcher {
    //countDownLatch这个类使⼀个线程等待,主要不让main⽅法结束
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
    private static ZooKeeper zooKeeper;

    public static void main(String[] args) throws Exception {
        zooKeeper = new ZooKeeper("10.211.55.4:2181", 5000, new CreateNote());
        countDownLatch.await();
    }

    public void process(WatchedEvent watchedEvent) {
        //当连接创建了，服务端发送给客户端SyncConnected事件
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            countDownLatch.countDown();
        }
        //调⽤创建节点⽅法
        try {
            createNodeSync();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void createNodeSync() throws Exception {
        /**
         * path ：节点创建的路径
         * data[] ：节点创建要保存的数据，是个byte类型的
         * acl ：节点创建的权限信息(4种类型)
         * ANYONE_ID_UNSAFE : 表示任何⼈
         * AUTH_IDS ：此ID仅可⽤于设置ACL。它将被客户机验证的ID替换。
         * OPEN_ACL_UNSAFE ：这是⼀个完全开放的ACL(常⽤)-->world:anyone
         * CREATOR_ALL_ACL ：此ACL授予创建者身份验证ID的所有权限
         * createMode ：创建节点的类型(4种类型)
         * PERSISTENT：持久节点
         * PERSISTENT_SEQUENTIAL：持久顺序节点
         * EPHEMERAL：临时节点
         * EPHEMERAL_SEQUENTIAL：临时顺序节点
         String node = zookeeper.create(path,data,acl,createMode);
         */
        String node_PERSISTENT = zooKeeper.create("/persistent", "持久节点内容".getBytes(" utf-8"), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        String node_PERSISTENT_SEQUENTIAL = zooKeeper.create("/persistent_sequential", "持久节点内容".getBytes("utf-8"), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);
        String node_EPERSISTENT = zooKeeper.create("/ephemeral", "临时节点内容".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        System.out.println("创建的持久节点是:" + node_PERSISTENT);
        System.out.println("创建的持久顺序节点是:" + node_PERSISTENT_SEQUENTIAL);
        System.out.println("创建的临时节点是:" + node_EPERSISTENT);
    }
}
```

### 获取节点数据

```java
public class GetNoteData implements Watcher {
    //countDownLatch这个类使⼀个线程等待,主要不让main⽅法结束
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
    private static ZooKeeper zooKeeper;

    public static void main(String[] args) throws Exception {
        zooKeeper = new ZooKeeper("10.211.55.4:2181", 10000, new GetNoteDate());
        Thread.sleep(Integer.MAX_VALUE);
    }

    public void process(WatchedEvent watchedEvent) {
        //⼦节点列表发⽣变化时，服务器会发出NodeChildrenChanged通知，但不会把变化情况告诉给客户端
        // 需要客户端⾃⾏获取，且通知是⼀次性的，需反复注册监听
        if (watchedEvent.getType() == Event.EventType.NodeChildrenChanged) {
            //再次获取节点数据
            try {
                List<String> children = zooKeeper.getChildren(watchedEvent.getPath(), true);
                System.out.println(children);
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //当连接创建了，服务端发送给客户端SyncConnected事件
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            try {
                //调⽤获取单个节点数据⽅法
                getNoteDate();
                getChildrens();
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private static void getNoteData() throws Exception {
        /**
         * path : 获取数据的路径
         * watch : 是否开启监听
         * stat : 节点状态信息, null表示获取最新版本的数据
         * zk.getData(path, watch, stat);
         */
        byte[] data = zooKeeper.getData("/persistent/children", true, null);
        System.out.println(new String(data, "utf-8"));
    }

    private static void getChildrens() throws KeeperException, InterruptedException {
        /*
        path:路径
        watch:是否要启动监听，当⼦节点列表发⽣变化，会触发监听
        zooKeeper.getChildren(path, watch);
        */
        List<String> children = zooKeeper.getChildren("/persistent", true);
        System.out.println(children);
    }
}
```

### 修改节点数据

```java
public class updateNote implements Watcher {
    private static ZooKeeper zooKeeper;

    public static void main(String[] args) throws Exception {
        zooKeeper = new ZooKeeper("10.211.55.4:2181", 5000, new updateNote());
        Thread.sleep(Integer.MAX_VALUE);
    }

    public void process(WatchedEvent watchedEvent) {
        //当连接创建了，服务端发送给客户端SyncConnected事件
        try {
            updateNodeSync();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void updateNodeSync() throws Exception {
        /*
        path:路径
        data:要修改的内容 byte[]
        version:为-1，表示对最新版本的数据进⾏修改
        zooKeeper.setData(path, data,version);
        */
        byte[] data = zooKeeper.getData("/persistent", false, null);
        System.out.println("修改前的值:" + new String(data));
        //修改 stat:状态信息对象 -1:最新版本
        Stat stat = zooKeeper.setData("/persistent", "客户端修改内容".getBytes(), -1);
        byte[] data2 = zooKeeper.getData("/persistent", false, null);
        System.out.println("修改后的值:" + new String(data2));
    }
}
```

### 删除节点

```java
public class DeleteNote implements Watcher {
    private static ZooKeeper zooKeeper;
    public static void main(String[] args) throws Exception {
        zooKeeper = new ZooKeeper("10.211.55.4:2181", 5000, new DeleteNote());
        Thread.sleep(Integer.MAX_VALUE);
    }
    public void process(WatchedEvent watchedEvent) {
        //当连接创建了，服务端发送给客户端SyncConnected事件
        try {
            deleteNodeSync();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    private void deleteNodeSync() throws KeeperException, InterruptedException
    {
        /*
        zooKeeper.exists(path,watch) :判断节点是否存在
        zookeeper.delete(path,version) : 删除节点
        */
        Stat exists = zooKeeper.exists("/persistent/children", false);
        System.out.println(exists == null ? "该节点不存在":"该节点存在");
        zooKeeper.delete("/persistent/children",-1);
        Stat exists2 = zooKeeper.exists("/persistent/children", false);
        System.out.println(exists2 == null ? "该节点不存在":"该节点存在");
    }
}
```

## ZkClient

ZkClient是Github上⼀个开源的zookeeper客户端，在Zookeeper原⽣API接⼝之上进⾏了包装，是⼀个更易⽤的Zookeeper客户端，同时， zkClient在内部还实现了诸如Session超时重连、 Watcher反复注册等功能。

接下来，还是从创建会话、创建节点、读取数据、更新数据、删除节点等⽅⾯来介绍如何使⽤zkClient这个zookeeper客户端。

### 添加依赖

在pom.xml⽂件中添加如下内容

```xml
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.2</version>
</dependency>
```

### 创建会话

使⽤ZkClient可以轻松的创建会话，连接到服务端。

```java
public class CreateSession {
    /*
    创建⼀个zkClient实例来进⾏连接
    注意： zkClient通过对zookeeperAPI内部包装，将这个异步的会话创建过程同步化了
    */
    public static void main(String[] args) {
        ZkClient zkClient = new ZkClient("127.0.0.1:2181");
        System.out.println("ZooKeeper session established.");
    }
}
```

运⾏结果： ZooKeeper session established.

结果表明已经成功创建会话。

### 创建节点

ZkClient提供了递归创建节点的接⼝，即其帮助开发者先完成⽗节点的创建，再创建⼦节点

```java
public class Create_Node_Sample {
    public static void main(String[] args) {
        ZkClient zkClient = new ZkClient("127.0.0.1:2181");
        System.out.println("ZooKeeper session established.");
        //createParents的值设置为true，可以递归创建节点
        zkClient.createPersistent("/zkClient/c1", true);
        System.out.println("success create znode.");
    }
}
```

运⾏结果： success create znode.

结果表明已经成功创建了节点，值得注意的是，在原⽣态接⼝中是⽆法创建成功的（⽗节点不存在），

但是通过ZkClient通过设置createParents参数为true可以递归的先创建⽗节点，再创建⼦节点

### 删除节点

ZkClient提供了递归删除节点的接⼝，即其帮助开发者先删除所有⼦节点（存在），再删除⽗节点。

```java
public class Del_Data_Sample {
    public static void main(String[] args) throws Exception {
        String path = "/zkClient/c1";
        ZkClient zkClient = new ZkClient("127.0.0.1:2181", 5000);
        zkClient.deleteRecursive(path);
        System.out.println("success delete znode.");
    }
}
```

运⾏结果: success delete znode.

结果表明ZkClient可直接删除带⼦节点的⽗节点，因为其底层先删除其所有⼦节点，然后再删除⽗节点

### 获取⼦节点

```java
public class GetChildrenSample {
    public static void main(String[] args) throws Exception {
        ZkClient zkClient = new ZkClient("127.0.0.1:2181", 5000);
        List<String> children = zkClient.getChildren("/zkClient");
        System.out.println(children);
        //注册监听事件
        zkClient.subscribeChildChanges(path, new IZkChildListener() {
            public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception {
                System.out.println(parentPath + " 's child changed, currentChilds:" + currentChilds);
            }
        });
        zkClient.createPersistent("/zkClient");
        Thread.sleep(1000);
        zkClient.createPersistent("/zkClient/c1");
        Thread.sleep(1000);
        zkClient.delete("/zkClient/c1");
        Thread.sleep(1000);
        zkClient.delete(path);
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

运⾏结果：

```text
/zk-book 's child changed, currentChilds:[]
/zk-book 's child changed, currentChilds:[c1]
/zk-book 's child changed, currentChilds:[]
/zk-book 's child changed, currentChilds:null
```

结果表明：

客户端可以对⼀个不存在的节点进⾏⼦节点变更的监听。

⼀旦客户端对⼀个节点注册了⼦节点列表变更监听之后，那么当该节点的⼦节点列表发⽣变更时，服务端都会通知客户端，并将最新的⼦节点列表发送给客户端。

该节点本身的创建或删除也会通知到客户端。

### 获取数据（节点是否存在、更新、删除）

```java
public class GetDataSample {
    public static void main(String[] args) throws InterruptedException {
        String path = "/zkClient-Ep";
        ZkClient zkClient = new ZkClient("127.0.0.1:2181");
        //判断节点是否存在
        boolean exists = zkClient.exists(path);
        if (!exists) {
            zkClient.createEphemeral(path, "123");
        }
        //注册监听
        zkClient.subscribeDataChanges(path, new IZkDataListener() {
            public void handleDataChange(String path, Object data) throws Exception {
                System.out.println(path + "该节点内容被更新，更新后的内容" + data);
            }

            public void handleDataDeleted(String s) throws Exception {
                System.out.println(s + " 该节点被删除");
            }
        });
        //获取节点内容
        Object o = zkClient.readData(path);
        System.out.println(o);
        //更新
        zkClient.writeData(path, "4567");
        Thread.sleep(1000);
        //删除
        zkClient.delete(path);
        Thread.sleep(1000);
    }
}
```

运行结果：

```text
123
/zkClient-Ep该节点内容被更新，更新后的内容4567
/zkClient-Ep 该节点被删除
```

结果表明可以成功监听节点数据变化或删除事件。

## Curator

curator是Netflix公司开源的⼀套Zookeeper客户端框架，和ZKClient⼀样， Curator解决了很多Zookeeper客户端⾮常底层的细节开发⼯作，包括连接重连，反复注册Watcher和NodeExistsException异常等，是最流⾏的Zookeeper客户端之⼀。从编码⻛格上来讲，它提供了基于Fluent的编程⻛格⽀持。

### 添加依赖

在pom.xml⽂件中添加如下内容：

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.12.0</version>
</dependency>
```

### 创建会话

Curator的创建会话⽅式与原⽣的API和ZkClient的创建⽅式区别很⼤。 Curator创建客户端是通过`CuratorFrameworkFactory`⼯⼚类来实现的。具体如下：

1. 使⽤CuratorFramework这个⼯⼚类的两个静态⽅法来创建⼀个客户端

```java
public static CuratorFramework newClient(String connectString, RetryPolicy retryPolicy)
public static CuratorFramework newClient(String connectString, int sessionTimeoutMs, int connectionTimeoutMs, RetryPolicy retryPolicy)
```

其中参数`RetryPolicy`提供重试策略的接⼝，可以让⽤户实现⾃定义的重试策略，默认提供了以下实现，分别为`ExponentialBackoffRetry`（基于backoff的重连策略）、 `RetryNTimes`（重连N次策略）、`RetryForever`（永远重试策略）

2. 通过调⽤`CuratorFramework`中的`start()`⽅法来启动会话

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000,3);
CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181",retryPolicy);
client.start();

```

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000,3);
CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000,1000,retryPolicy);
client.start();
```

其实进⼀步查看源代码可以得知，其实这两种⽅法内部实现⼀样，只是对外包装成不同的⽅法。它们的底层都是通过第三个⽅法builder来实现的。

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000,3);
private static CuratorFramework Client = CuratorFrameworkFactory.builder()
            .connectString("server1:2181,server2:2181,server3:2181")
            .sessionTimeoutMs(50000)
            .connectionTimeoutMs(30000)
            .retryPolicy(retryPolicy)
            .build();
client.start();
```

参数：  

- `connectString`： zk的server地址，多个server之间使⽤英⽂逗号分隔开  
- `connectionTimeoutMs`：连接超时时间，如上是30s，默认是15s  
- `sessionTimeoutMs`：会话超时时间，如上是50s，默认是60s  
- `retryPolicy`：失败重试策略  
    - `ExponentialBackoffRetry`：构造器含有三个参数 `ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries, int maxSleepMs)  `
        - `baseSleepTimeMs`：初始的sleep时间，⽤于计算之后的每次重试的sleep时间，  
            - 计算公式：当前sleep时间 = baseSleepTimeMs*Math.max(1, random.nextInt(1<<(retryCount+1)))
        - `maxRetries`：最⼤重试次数  
        - `maxSleepMs`：最⼤sleep时间，如果上述的当前sleep计算出来⽐这个⼤，那么sleep⽤这个时间，默认的最⼤时间是Integer.MAX_VALUE毫秒。  
    - 其他，查看`org.apache.curator.RetryPolicy`接⼝的实现类  

`start()`：完成会话的创建

```java
public class CreateSessionSample {
    public static void main(String[] args) throws Exception {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", 5000, 3000, retryPolicy);
        client.start();
        System.out.println("Zookeeper session1 established. ");
        CuratorFramework client1 = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181") //server地址
                .sessionTimeoutMs(5000) // 会话超时时间
                .connectionTimeoutMs(3000) // 连接超时时间
                .retryPolicy(retryPolicy) // 重试策略
                .namespace("base") // 独⽴命名空间/base
                .build(); //
        client1.start();
        System.out.println("Zookeeper session2 established. ");
    }
}
```

运⾏结果： Zookeeper session1 established. Zookeeper session2 established

需要注意的是session2会话含有隔离命名空间，即客户端对Zookeeper上数据节点的任何操作都是相对/base⽬录进⾏的，这有利于实现不同的Zookeeper的业务之间的隔离

### 创建节点

curator提供了⼀系列Fluent⻛格的接⼝，通过使⽤Fluent编程⻛格的接⼝，开发⼈员可以进⾏⾃由组合来完成各种类型节点的创建。

下⾯简单介绍⼀下常⽤的⼏个节点创建场景。

- 创建⼀个初始内容为空的节点

```java
client.create().forPath(path);
```

```
Curator默认创建的是持久节点，内容为空。  
```

- 创建⼀个包含内容的节点

```java
client.create().forPath(path,"我是内容".getBytes());
```

Curator和ZkClient不同的是依旧采⽤Zookeeper原⽣API的⻛格，内容使⽤byte[]作为⽅法参数。  

- 递归创建⽗节点,并选择节点类型

```java
client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path);
```

creatingParentsIfNeeded这个接⼝⾮常有⽤，在使⽤ZooKeeper 的过程中，开发⼈员经常会碰到NoNodeException 异常，其中⼀个可能的原因就是试图对⼀个不存在的⽗节点创建⼦节点。因此，开发⼈员不得不在每次创建节点之前，都判断⼀下该⽗节点是否存在——这个处理通常⽐较麻烦。在使⽤Curator 之后，通过调⽤creatingParentsIfNeeded 接⼝， Curator 就能够⾃动地递归创建所有需要的⽗节点。

下⾯通过⼀个实际例⼦来演示如何在代码中使⽤这些API。

```java
public static void main(String[] args) throws Exception {
    CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("127.0.0.1:2181") //server地址
            .sessionTimeoutMs(5000) // 会话超时时间
            .connectionTimeoutMs(3000) // 连接超时时间
            .retryPolicy(new ExponentialBackoffRetry(1000, 5)) // 重试策略
            .build(); //
    client.start();
    System.out.println("Zookeeper session established. ");
    //添加节点
    String path = "/curator/c1";
    client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path, "init".getBytes());
    Thread.sleep(1000);
    System.out.println("success create znode" + path);
}
```

运⾏结果： Zookeeper session established. success create znode/curator/c1

其中，也创建了curator/c1的⽗节点curator节点。

### 删除节点

删除节点的⽅法也是基于Fluent⽅式来进⾏操作，不同类型的操作调⽤ 新增不同的⽅法调⽤即可。

* 删除⼀个⼦节点

```java
client.delete().forPath(path);
```

* 删除节点并递归删除其⼦节点

```java
client.delete().deletingChildrenIfNeeded().forPath(path);
```

* 指定版本进⾏删除

```java
client.delete().withVersion(1).forPath(path);
```

如果此版本已经不存在，则删除异常，异常信息如下。

```java
org.apache.zookeeper.KeeperException$BadVersionException: KeeperErrorCode =
BadVersion for
```

* 强制保证删除⼀个节点

```java
client.delete().guaranteed().forPath(path);
```

只要客户端会话有效，那么Curator会在后台持续进⾏删除操作，直到节点删除成功。⽐如遇到⼀些⽹络异常的情况，此guaranteed的强制删除就会很有效果。

演示实例：

```java
public static void main(String[] args) throws Exception {
    CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("127.0.0.1:2181") //server地址
            .sessionTimeoutMs(5000) // 会话超时时间
            .connectionTimeoutMs(3000) // 连接超时时间
            .retryPolicy(new ExponentialBackoffRetry(1000,5)) // 重试策略
            .build(); //
    client.start();
    System.out.println("Zookeeper session established. ");
    //删除节点
    String path = "/curator";
    client.delete().deletingChildrenIfNeeded().withVersion(-1).forPath(path);
    System.out.println("success create znode"+path);
}
```

运⾏结果： Zookeeper session established. success create znode/curator

结果表明成功删除/curator节点

### 获取数据

获取节点数据内容API相当简单，同时Curator提供了传⼊⼀个Stat变量的⽅式来存储服务器端返回的最新的节点状态信息

```java
// 普通查询
client.getData().forPath(path);
// 包含状态查询
Stat stat = new Stat();
client.getData().storingStatIn(stat).forPath(path);
```

演示

```java
public class GetNodeSample {
    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181") //server地址
                .sessionTimeoutMs(5000) // 会话超时时间运⾏结果： Zookeeper session established. success create znode/curator/c1 init
                .connectionTimeoutMs(3000) // 连接超时时间
                .retryPolicy(new ExponentialBackoffRetry(1000, 5)) // 重试策略
                .build(); //
        client.start();
        System.out.println("Zookeeper session established. ");
        //添加节点
        String path = "/curator/c1";
        client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path, "init".getBytes());
        System.out.println("success create znode" + path);
        //获取节点数据
        Stat stat = new Stat();
        byte[] bytes = client.getData().storingStatIn(stat).forPath(path);
        System.out.println(new String(bytes));
    }
}
```

运⾏结果： Zookeeper session established. success create znode/curator/c1 init

结果表明成功获取了节点的数据

### 更新数据

更新数据，如果未传⼊version参数，那么更新当前最新版本，如果传⼊version则更新指定version，如果version已经变更，则抛出异常。

```java
// 普通更新
client.setData().forPath(path,"新内容".getBytes());
// 指定版本更新
client.setData().withVersion(1).forPath(path);  
```

版本不⼀致异常信息：

```java
org.apache.zookeeper.KeeperException$BadVersionException: KeeperErrorCode =
BadVersion for
```

案例演示：

```java
public class SetNodeSample {
    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181") //server地址
                .sessionTimeoutMs(5000) // 会话超时时间
                .connectionTimeoutMs(3000) // 连接超时时间
                .retryPolicy(new ExponentialBackoffRetry(1000, 5)) // 重试策略
                .build(); //
        client.start();
        System.out.println("Zookeeper session established. ");
        String path = "/curator/c1";
        //获取节点数据
        Stat stat = new Stat();
        byte[] bytes = client.getData().storingStatIn(stat).forPath(path);
        System.out.println(new String(bytes));
        //更新节点数据
        int version = client.setData().withVersion(stat.getVersion()).forPath(path).getVersion();
        System.out.println("Success set node for : " + path + ", new version:"+version);
        client.setData().withVersion(stat.getVersion()).forPath(path).getVersion();
    }
}
```

运⾏结果：

```java
Zookeeper session established.
init
Success set node for : /curator/c1, new version: 1
Exception in thread "main"
org.apache.zookeeper.KeeperException$BadVersionException: KeeperErrorCode =
BadVersion for /curator/c1
```

结果表明当携带数据版本不⼀致时，⽆法完成更新操作。