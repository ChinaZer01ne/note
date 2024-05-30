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


# Zookeeper源码分析

## 方法入口

入口在`QuorumPeerMain`的main方法

```java
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    main.initializeAndRun(args);
    ServiceUtils.requestSystemExit(ExitCode.EXECUTION_FINISHED.getValue());
}

protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }

    // Start and schedule the the purge task
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
        config.getDataDir(),
        config.getDataLogDir(),
        config.getSnapRetainCount(),
        config.getPurgeInterval());
    purgeMgr.start();
    // 集群模式启动
    if (args.length == 1 && config.isDistributed()) {
        runFromConfig(config);
    } else {
        // there is only server in the quorum -- run as standalone
        // 单机模式启动
        ZooKeeperServerMain.main(args);
    }
}

```

## 单机模式服务端启动

### 执行过程概述

单机模式的ZK服务端逻辑写在`ZooKeeperServerMain`类中，由里面的main函数启动，整个过程如下:

![](单机zk启动流程.png)

单机模式的委托启动类为:`ZooKeeperServerMain` 服务端启动过程 看下`ZooKeeperServerMain`里面的main函数代码:

```java
public static void main(String[] args) {
    ZooKeeperServerMain main = new ZooKeeperServerMain();
    main.initializeAndRun(args);
    ServiceUtils.requestSystemExit(ExitCode.EXECUTION_FINISHED.getValue());
}

protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    try {
        // 注册jmx    
        ManagedUtil.registerLog4jMBeans();
    } catch (JMException e) {
        LOG.warn("Unable to register log4j JMX control", e);
    }
    
    // 解析ServerConfig配置对象 
    ServerConfig config = new ServerConfig();
    // 如果参数是一个，则认为是配置文件的路径
    if (args.length == 1) {
        config.parse(args[0]);
    } else {
        //否则是多个参数
        config.parse(args);
    }

    runFromConfig(config);
}

/**
 * Run from a ServerConfig.
 * @param config ServerConfig to use.
 * @throws IOException
 * @throws AdminServerException
 */
public void runFromConfig(ServerConfig config) throws IOException, AdminServerException {
    LOG.info("Starting server");
    FileTxnSnapLog txnLog = null;
    try {
        try {
            metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
                config.getMetricsProviderClassName(),
                config.getMetricsProviderConfiguration());
        } catch (MetricsProviderLifeCycleException error) {
            throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
        }
        ServerMetrics.metricsProviderInitialized(metricsProvider);
        ProviderRegistry.initialize();
        // Note that this thread isn't going to be doing anything else,
        // so rather than spawning another thread, we will just call
        // run() in this thread.
        // create a file logger url from the command line args
        //，创建FileTxnSnapLog管理器，提供了一系列操作数据文件的接口，如事务日志文件和快照数据文件
        txnLog = new FileTxnSnapLog(config.dataLogDir, config.dataDir);
        JvmPauseMonitor jvmPauseMonitor = null;
        if (config.jvmPauseMonitorToRun) {
            jvmPauseMonitor = new JvmPauseMonitor(config);
        }
        // 初始化ZooKeeperServer对象，创建服务器统计器
        final ZooKeeperServer zkServer = new ZooKeeperServer(jvmPauseMonitor, txnLog, config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, config.listenBacklog, null, config.initialConfig);
        txnLog.setServerStats(zkServer.serverStats());

        // Registers shutdown handler which will be used to know the
        // server error or shutdown state changes.
        final CountDownLatch shutdownLatch = new CountDownLatch(1);
        // 设置zk服务钩子，用于知道服务器错误或关闭状态更改
        zkServer.registerServerShutdownHandler(new ZooKeeperServerShutdownHandler(shutdownLatch));

        // Start Admin server，创建admin服务，用于接收请求（创建jetty服务）
        adminServer = AdminServerFactory.createAdminServer();
        // 设置zookeeper服务
        adminServer.setZooKeeperServer(zkServer);
        adminServer.start();

        boolean needStartZKServer = true;
        // 启动zookeeper
        // 判断配置文件中clientPortAddress是否为null
        if (config.getClientPortAddress() != null) {
            //初始化server端IO对象，默认是NIOServerCnxnFactory ：Java原生NIO处理网络IO事件
            // ServerCnxnFactory是zookeeper中的重要的组件，负责处理客户端与服务端的连接
            cnxnFactory = ServerCnxnFactory.createFactory();
            //初始化配置信息 
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
            //启动服务：此方法除了启动ServerCnxnFactory，还会启动zookeeper
            cnxnFactory.startup(zkServer);
            // zkServer has been started. So we don't need to start it again in secureCnxnFactory.
            needStartZKServer = false;
        }
        if (config.getSecureClientPortAddress() != null) {
            secureCnxnFactory = ServerCnxnFactory.createFactory();
            secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
            secureCnxnFactory.startup(zkServer, needStartZKServer);
        }
         // 定时清除容器节点
        //container ZNodes是3.6版本之后新增的节点类型，Container类型的节点会在它没有子节点时
        // 被删除(新创建的Container节点除外)，该类就是用来周期性的进行检查清理工作
        containerManager = new ContainerManager(
            zkServer.getZKDatabase(),
            zkServer.firstProcessor,
            Integer.getInteger("znode.container.checkIntervalMs", (int) TimeUnit.MINUTES.toMillis(1)),
            Integer.getInteger("znode.container.maxPerMinute", 10000),
            Long.getLong("znode.container.maxNeverUsedIntervalMs", 0)
        );
        containerManager.start();
        ZKAuditProvider.addZKStartStopAuditLog();

        serverStarted();

        // Watch status of ZooKeeper server. It will do a graceful shutdown
        // if the server is not running or hits an internal error.
        // zookeeperShutdownHandler处理逻辑，只有在服务运行不正常的情况下，才会往下执行
        shutdownLatch.await();
        // 关闭服务
        shutdown();

        if (cnxnFactory != null) {
            cnxnFactory.join();
        }
        if (secureCnxnFactory != null) {
            secureCnxnFactory.join();
        }
        if (zkServer.canShutdown()) {
            zkServer.shutdown(true);
        }
    } catch (InterruptedException e) {
        // ...
    } finally {
        // ...
    }
}

```

### 启动流程小结

zk单机模式启动主要流程:
1. 注册jmx
2. 解析ServerConfig配置对象
3. 根据配置对象，运行单机zk服务
4. 创建管理事务日志和快照FileTxnSnapLog对象，zookeeperServer对象，并设置zkServer的统计对象
5. 设置zk服务钩子，原理是通过设置CountDownLatch，调用ZooKeeperServerShutdownHandler的 handle方法，可以将触发shutdownLatch.await方法继续执行，即调用shutdown关闭单机服务
6. 基于jetty创建zk的admin服务
7. 创建连接对象cnxnFactory和secureCnxnFactory(安全连接才创建该对象)，用于处理客户端的请求
8. 创建定时清除容器节点管理器，用于处理容器节点下不存在子节点的清理容器节点工作等可以看到关键点在于解析配置跟启动两个方法，先来看下解析配置逻辑，对应上面的configure方法:

```java
public void configure(InetSocketAddress addr, int maxcc, int backlog, boolean secure) throws IOException {
    if (secure) {
        throw new UnsupportedOperationException("SSL isn't supported in NIOServerCnxn");
    }
    configureSaslLogin();

    //会话超时时间
    maxClientCnxns = maxcc;
    
    initMaxCnxns();
    sessionlessCnxnTimeout = Integer.getInteger(ZOOKEEPER_NIO_SESSIONLESS_CNXN_TIMEOUT, 10000);
    // We also use the sessionlessCnxnTimeout as expiring interval for
    // cnxnExpiryQueue. These don't need to be the same, but the expiring
    // interval passed into the ExpiryQueue() constructor below should be
    // less than or equal to the timeout.
    //过期队列
    cnxnExpiryQueue = new ExpiryQueue<NIOServerCnxn>(sessionlessCnxnTimeout);
    //过期线程，从cnxnExpiryQueue中读取数据，如果已经过期则关闭
    expirerThread = new ConnectionExpirerThread();

    int numCores = Runtime.getRuntime().availableProcessors();
    // 32 cores sweet spot seems to be 4 selector threads
    numSelectorThreads = Integer.getInteger(
        ZOOKEEPER_NIO_NUM_SELECTOR_THREADS,
        Math.max((int) Math.sqrt((float) numCores / 2), 1));
    if (numSelectorThreads < 1) {
        throw new IOException("numSelectorThreads must be at least 1");
    }

    //计算woker线程的数量
    numWorkerThreads = Integer.getInteger(ZOOKEEPER_NIO_NUM_WORKER_THREADS, 2 * numCores);
    //worker线程关闭时间
    workerShutdownTimeoutMS = Long.getLong(ZOOKEEPER_NIO_SHUTDOWN_TIMEOUT, 5000);

    for (int i = 0; i < numSelectorThreads; ++i) {
        selectorThreads.add(new SelectorThread(i));
    }

    listenBacklog = backlog;
    //初始化selector线程
    this.ss = ServerSocketChannel.open();
    ss.socket().setReuseAddress(true);
    LOG.info("binding to port {}", addr);
    if (listenBacklog == -1) {
        ss.socket().bind(addr);
    } else {
        ss.socket().bind(addr, listenBacklog);
    }
    ss.configureBlocking(false);
    //初始化accept线程，这里看出accept线程只有一个，里面会注册监听ACCEPT事件
    acceptThread = new AcceptThread(ss, addr, selectorThreads);
}

```

再来看下启动逻辑，在`NIOServerCnxnFactory`里:

```java
public void startup(ZooKeeperServer zkServer) throws IOException, InterruptedException {
    startup(zkServer, true);
}

public void startup(ZooKeeperServer zks, boolean startServer) throws IOException, InterruptedException {
    // 启动相关线程
    start();
    setZooKeeperServer(zks);
    if (startServer) {
        //恢复本地数据
        zks.startdata();
        // 启动定时清除session的管理器，注册jmx，添加请求处理器
        zks.startup();
    }
}

//首先是start方法
public void start() {
    stopped = false; 
    //初始化worker线程池
    if (workerPool == null) {
        workerPool = new WorkerService("NIOWorker", numWorkerThreads, false);
    }
    //挨个启动selector线程
    for (SelectorThread thread : selectorThreads) {
        if (thread.getState() == Thread.State.NEW) {
            thread.start();
        }
    }
    //启动acceptThread线程
    if (acceptThread.getState() == Thread.State.NEW) {
        acceptThread.start();
    }
    //启动expirerThread线程，处理过期连接
    if (expirerThread.getState() == Thread.State.NEW) {
        expirerThread.start();
    }
}

//初始化数据结构
public void startdata() throws IOException, InterruptedException {
    //初始化ZKDatabase，该数据结构用来保存ZK上面存储的所有数据
    if (zkDb == null) {
        //初始化数据数据，这里会加入一些原始节点，例如/zookeeper
        zkDb = new ZKDatabase(this.txnLogFactory);
    }
    //加载磁盘上已经存储的数据，如果有的话
    if (!zkDb.isInitialized()) {
        loadData();
    }
}

private void startupWithServerState(State state) {
    //初始化session追踪器
    if (sessionTracker == null) {
        createSessionTracker();
    }
    //启动session追踪器 
    startSessionTracker();
    //建立请求处理链路 
    setupRequestProcessors();

    startRequestThrottler();
    // 注册jmx
    // jmx的全程Java Management Extensions，是管理Java的一种扩展
    // 这种机制可以方便的管理、监控正在运行中的程序。常用于管理线程，内存，日志level，服务重启，系统环境等
    registerJMX();

    startJvmPauseMonitor();

    registerMetrics();

    setState(state);

    requestPathMetricsCollector.start();

    localSessionEnabled = sessionTracker.isLocalSessionsEnabled();

    notifyAll();
}

//这里可以看出，单机模式下请求的处理链路为:
//PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor syncProcessor = new SyncRequestProcessor(this, finalProcessor);
    ((SyncRequestProcessor) syncProcessor).start();
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor) firstProcessor).start();
}
```

## Leader选举

### Election

分析Zookeeper中一个核心的模块，Leader选举。主要是`Election``实现类FastLeaderElection`，`AuthFastLeaderElection`，`LeaderElection`其在3.4.0之后的版本中已经不建议使用。

```java
public interface Election {
    public Vote lookForLeader() throws InterruptedException;
    public void shutdown();
}
```

说明: 选举的父接口为`Election`，其定义了`lookForLeader`和`shutdown`两个方法，`lookForLeader`表示寻找Leader，`shutdown`则表示关闭，如关闭服务端之间的连接。

### FastLeaderElection

刚刚介绍了Leader选举的总体框架，接着来学习Zookeeper中默认的选举策略，FastLeaderElection。 FastLeaderElection源码分析类的继承关系

```java
public class FastLeaderElection implements Election {}
```

说明:FastLeaderElection实现了Election接口，重写了接口中定义的lookForLeader方法和shutdown方法

在源码分析之前，我们首先介绍几个概念:

- 外部投票：特指其他服务器发来的投票。
- 内部投票：服务器自身当前的投票。
- 选举轮次：ZooKeeper服务器Leader选举的轮次，即logical clock(逻辑时钟)。
- PK：指对内部投票和外部投票进行一个对比来确定是否需要变更内部投票。选票管理
- sendqueue：选票发送队列，用于保存待发送的选票。
- recvqueue：选票接收队列，用于保存接收到的外部投票。

![](FastLeaderElection类结构.jpg)

- `Notification`：FastLeaderElection的内部类。他表示收到的选举投票信息（其他服务器发来的选举投票信息），其中包含了被选举者的id、zxid、选举周期等信息。
- `ToSend`：表示发送给其他服务器的选举投票信息，也包含了被选举者的id、zxid、选举周期等信息
- `Messager`：消息处理的多线程实现，其中有两个内部类，WorkReceiver和WordSender，一个是接收选票信息，另一个是发送选票信息

### lookForLeader函数

当 ZooKeeper 服务器检测到当前服务器状态变成 LOOKING 时，就会触发 Leader选举，即调用 lookForLeader方法来进行Leader选举。

![](zk选举流程图.png)

```java
/**
 * 开始新一轮的leader选举。 每当我们的 QuorumPeer 状态更改为 LOOKING 时，就会调用此方法，并向所有其他对等方发送通知。
 * Starts a new round of leader election. Whenever our QuorumPeer
 * changes its state to LOOKING, this method is invoked, and it
 * sends notifications to all other peers.
 */
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        self.jmxLeaderElectionBean = null;
    }

    self.start_fle = Time.currentElapsedTime();
    try {
        /*
         * 当前领导人选举的选票存储在 recvset 中。
         * 换句话说，如果 v.electionEpoch == logicalclock，则投票 v 在 recvset 中。
         * 当前参与者使用 recvset 来推断是否大多数参与者都投票支持它。
         * The votes from the current leader election are stored in recvset. In other words, a vote v is in recvset
         * if v.electionEpoch == logicalclock. The current participant uses recvset to deduce on whether a majority
         * of participants has voted for it.
         */
        Map<Long, Vote> recvset = new HashMap<Long, Vote>();

        /*
         * 之前领导人选举的选票，以及当前领导人选举的选票都存储在 outofelection 中。
         * 请注意，处于 LOOKING 状态的通知不会存储在 outofelection 中。
         * 只有 FOLLOWING 或 LEADING 通知存储在 outofelection 中。
         * 当前参与者可以使用 outofelection 来了解哪个参与者是领导者，如果它在领导选举中迟到（即，逻辑时钟高于收到通知的选举纪元）。
         * The votes from previous leader elections, as well as the votes from the current leader election are
         * stored in outofelection. Note that notifications in a LOOKING state are not stored in outofelection.
         * Only FOLLOWING or LEADING notifications are stored in outofelection. The current participant could use
         * outofelection to learn which participant is the leader if it arrives late (i.e., higher logicalclock than
         * the electionEpoch of the received notifications) in a leader election.
        */
        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = minNotificationInterval;

        synchronized (this) {
            // 1. ⾸先会将逻辑时钟⾃增，每进⾏⼀轮新的leader选举，都需要更新逻辑时钟
            logicalclock.incrementAndGet();
            // 2. 初始化选票（更新选票）
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        
        // 3. 向其他服务器发送⾃⼰的选票（已更新的选票）
        sendNotifications();

        SyncedLearnerTracker voteSet;

        /*
         * 我们交换选票信息直到找到领导者的循环
         * Loop in which we exchange notifications until we find a leader
         */
        // 选举开始，当前机器选举状态是LOOKING，选举还未结果
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time
             */
            // 4. 接收外部投票。从recvqueue接收队列中取出其他机器的投票（外部投票）
            Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            // 没有获得选票
            if (n == null) {
                // 判断自己是否和集群断开了连接
                // manager已经发送了所有选票消息（表示有连接）
                if (manager.haveDelivered()) {
                    // 向所有其他服务器发送消息
                    sendNotifications();
                } else { // 还未发送所有消息（表示⽆连接）
                    // 重新连接其他每个服务器
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                int tmpTimeOut = notTimeout * 2;
                notTimeout = Math.min(tmpTimeOut, maxNotificationInterval);
            // 5. 处理外部投票（判断选举轮次）
            } else if (validVoter(n.sid) && validVoter(n.leader)) {
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view for a replica in the current or next voting view.
                 */
                switch (n.state) {
                case LOOKING:
                    if (getInitLastLoggedZxid() == -1) {
                        LOG.debug("Ignoring notification as our zxid is -1");
                        break;
                    }
                    if (n.zxid == -1) {
                        LOG.debug("Ignoring notification from member with -1 zxid {}", n.sid);
                        break;
                    }
                    // If notification > current, replace and send messages out
                    // 外部投票的选举轮次大于内部投票
                    if (n.electionEpoch > logicalclock.get()) {
                        // 更新选举轮次
                        logicalclock.set(n.electionEpoch);
                        // 清空所有接收到的投票
                        recvset.clear();
                        // 进⾏PK，选出较优的服务器
                        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            // 更新选票，外部选票为主
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else { // ⽆法选出较优的服务器
                            // 更新选票，自己的选票优先
                            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                        }
                        // 再次发送本服务器的内部选票消息
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) {
                        // 如果外部投票的选举轮次小于内部投票，不做处理，直接忽略
                        break;
                    // 外部投票的选举轮次和内部投票一致，也是绝大多数情况
                    // 6. 进行投票PK
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) { // PK，选出较优的服务器
                        // 更新自己本身的轮次
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        //7。 变更选票，重新发送选票信息
                        sendNotifications();
                    }
        
                    //8.选票归档： 将收到的外部投票放进选票集合recvset中
                    // recvset⽤于记录当前服务器在本轮次的Leader选举中收到的所有外部投票
                    // don't care about the version if it's in LOOKING state
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));
                    // 9. 判断当前节点收到的票数是否可以结束选举
                    // 若能选出leader  
                    if (voteSet.hasAllQuorums()) {
                        // 遍历已经接收的投票集合
                        // Verify if there is any change in the proposed leader
                        while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                            // 选票有变更，⽐之前提议的Leader有更好的选票加⼊
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                // 将更优的选票放在recvset中
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                         // 10. 更新服务器状态
                         // 表示之前提议的Leader已经是最优的
                        if (n == null) {
                            // 设置服务器状态
                            setPeerState(proposedLeader, voteSet);
                            // 最终的选票
                            Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                            // 清空recvqueue队列的选票
                            leaveInstance(endVote);
                            // 返回选票
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: {}", n.sid);
                    break;
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if (n.electionEpoch == logicalclock.get()) {
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        if (voteSet.hasAllQuorums() && checkLeader(recvset, n.leader, n.electionEpoch)) {
                            setPeerState(n.leader, voteSet);
                            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify that
                     * a majority are following the same leader.
                     *
                     * Note that the outofelection map also stores votes from the current leader election.
                     * See ZOOKEEPER-1732 for more information.
                     */
                    outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

                    if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                        synchronized (this) {
                            logicalclock.set(n.electionEpoch);
                            setPeerState(n.leader, voteSet);
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecognized: {} (n.state), {}(n.sid)", n.state, n.sid);
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if (self.jmxLeaderElectionBean != null) {
                MBeanRegistry.getInstance().unregister(self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
    }
}
```

### 执行顺序

1. 自增选举轮次。 在 FastLeaderElection 实现中，有一个 logicalclock 属性，用于标识当前Leader的 选举轮次，ZooKeeper规定了所有有效的投票都必须在同一轮次中。ZooKeeper在开始新一轮的投票 时，会首先对logicalclock进行自增操作。
2. 初始化选票。 在开始进行新一轮的投票之前，每个服务器都会首先初始化自己的选票。在图7-33中我 们已经讲解了 Vote 数据结构，初始化选票也就是对 Vote 属性的初始化。在初始化阶段，每台服务器都 会将自己推举为Leader，表7-10展示了一个初始化的选票。
3. 发送初始化选票。 在完成选票的初始化后，服务器就会发起第一次投票。ZooKeeper 会将刚刚初始 化好的选票放入sendqueue队列中，由发送器WorkerSender负责

```java
/**
 * Send notifications to all peers upon a change in our vote
 */
private void sendNotifications() {
    for (long sid : self.getCurrentAndNextConfigVoters()) {
        QuorumVerifier qv = self.getQuorumVerifier();
        // 将选票信息封装成toSend
        ToSend notmsg = new ToSend(
            ToSend.mType.notification,
            proposedLeader,
            proposedZxid,
            logicalclock.get(),
            QuorumPeer.ServerState.LOOKING,
            sid,
            proposedEpoch,
            qv.toString().getBytes(UTF_8));
        // 放入队列中，由发送器workerSender发送出去
        sendqueue.offer(notmsg);
    }
}
```

4. 接收外部投票。 每台服务器都会不断地从 recvqueue 队列中获取外部投票。如果服务器发现无法获 取到任何的外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效连接。如果发现没有 建立连接，那么就会⻢上建立连接。如果已经建立了连接，那么就再次发送自己当前的内部投票。

```java
/*
 * Loop in which we exchange notifications until we find a leader
 */
  
while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
    // 从recvqueue接收队列中取出投票
    Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

    /*
     * Sends more notifications if haven't received enough.
     * Otherwise processes new notification.
     */
    if(n == null){ // 无法获取选票
        if(manager.haveDelivered()){ 
            // manager已经发送了所有选票消息
            // 向所有其他服务器发送消息
            sendNotifications();
        } else { 
            // 还未发送所有消息(表示无连接)
            // 连接其他每个服务器
            manager.connectAll();
        }
        /*
         * Exponential backoff
         */
        int tmpTimeOut = notTimeout*2;
        notTimeout = (tmpTimeOut < maxNotificationInterva l? tmpTimeOut : maxNotificationInterval);
    }
```

5. 判断选举轮次。 当发送完初始化选票之后，接下来就要开始处理外部投票了。在处理外部投票的时 候，会根据选举轮次来进行不同的处理。
    - 外部投票的选举轮次大于内部投票。如果服务器发现自己的 选举轮次已经落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次 (logicalclock)，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内 部投票(关于PK的逻辑会在步骤6中统一讲解)，最终再将内部投票发送出去。
    - 外部投票的选举轮次 小于内部投票。 如果接收到的选票的选举轮次落后于服务器自身的，那么ZooKeeper就会直接忽略该外 部投票，不做任何处理，并返回步骤4。
    - 外部投票的选举轮次和内部投票一致。 这也是绝大多数投票的场景，如外部投票的选举轮次和内部投 票一致的话，那么就开始进行选票PK。 总的来说，只有在同一个选举轮次的投票才是有效的投票。
6. 选票PK。 在步骤5中提到，在收到来自其他服务器有效的外部投票后，就要进行选票PK了——也就是 FastLeaderElection.totalOrderPredicate方法的核心逻辑。选票PK的目的是为了确定当前服务器是否 需要变更投票，主要从选举轮次、ZXID和 SID 三个因素来考虑，具体条件如下:在选票 PK 的时候依次 判断，符合任意一个条件就需要进行投票变更。 · 如果外部投票中被推举的Leader服务器的选举轮次大 于内部投票，那么就需要进行投票变更。 · 如果选举轮次一致的话，那么就对比两者的ZXID。如果外部 投票的ZXID大于内部投票，那么就需要进行投票变更。 · 如果两者的 ZXID 一致，那么就对比两者的 SID。如果外部投票的 SID 大于内部投票，那么就需要进行投票变更。

```java
/**
 * 如何PK出leader？
 */
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    if (self.getQuorumVerifier().getWeight(newId) == 0) {
        return false;
    }

    /*
     * 如果以下三种情况之一成立，我们将返回 true： 
     * 1- 新纪元更高 （选举轮次高的优先）
     * 2- 新纪元与当前纪元相同，但新 zxid 更高 （选举轮次相同，zxid高的优先）
     * 3- 新纪元与当前纪元相同，新 zxid 相同，但服务器 id 更高。（选举轮次相同，zxid也相同，服务器id高的优先）
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */

    return ((newEpoch > curEpoch)
            || ((newEpoch == curEpoch)
                && ((newZxid > curZxid)
                    || ((newZxid == curZxid)
                        && (newId > curId)))));
}
```

7. 变更投票。 通过选票PK后，如 果确定了外部投票优于内部投票(所谓的“优于”，是指外部投票所推举的服务器更适合成为Leader)， 那么就进行投票变更——使用外部投票的选票信息来覆盖内部投票。变更完成后，再次将这个变更后的 内部投票发送出去。
8. 选票归档。 无论是否进行了投票变更，都会将刚刚收到的那份外部投票放入“选票集合”recvset中进行 归档。recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票——按照服务器对应 的SID来区分，例如，{(1，vote1)，(2，vote2)，...}。

```java
voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));

if (voteSet.hasAllQuorums()) {
// Verify if there is any change in the proposed leader
    while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
            recvqueue.put(n);
            break;
        }
    }

    /*
     * This predicate is true once we don't read any new
     * relevant message from the reception queue
     */
    if (n == null) {
        setPeerState(proposedLeader, voteSet);
        Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
        leaveInstance(endVote);
        return endVote;
    }
}

```

9. 统计投票。 完成了选票归档之后，就可 以开始统计投票了。统计投票的过程就是为了统计集群中是否已经有过半的服务器认可了当前的内部投 票。如果确定已经有过半的服务器认可了该内部投票，则终止投票。否则返回步骤4。
10. 更新服务器状态。 统计投票后，如果已经确定可以终止投票，那么就开始更新服务器状态。服务器会首先判断当前被 过半服务器认可的投票所对应的Leader服务器是否是自己，如果是自己的话，那么就会将自己的服务器 状态更新为 LEADING。如果自己不是被选举产生的 Leader 的话，那么就会根据具体情况来确定自己是 FOLLOWING或是OBSERVING。

以上 10 个步骤，就是 FastLeaderElection 选举算法的核心步骤，其中步骤 4~9 会经过几轮循环，直到Leader选举产生。另外还有一个细节需要注意，就是在完成步骤9 之后，如果统计投票发现已经有过半的服务器认可了当前的选票，这个时候，ZooKeeper 并不会立即进 入步骤 10 来更新服务器状态，而是会等待一段时间(默认是 200 毫秒)来确定是否有新的更优的投票

## 集群模式服务端

执行流程图

![](集群zk启动流程.png)
### 源码分析

集群模式下启动所有的ZK节点启动入口都是QuorumPeerMain类的main方法。 main方法加载配置文件以后，最终会调用到QuorumPeer的start方法，来看下:

```java
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
    try {
        ManagedUtil.registerLog4jMBeans();
    } catch (JMException e) {
        LOG.warn("Unable to register log4j JMX control", e);
    }

    LOG.info("Starting quorum peer, myid=" + config.getServerId());
    final MetricsProvider metricsProvider;
    try {
        metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
            config.getMetricsProviderClassName(),
            config.getMetricsProviderConfiguration());
    } catch (MetricsProviderLifeCycleException error) {
        throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
    }
    try {
        ServerMetrics.metricsProviderInitialized(metricsProvider);
        ProviderRegistry.initialize();
        ServerCnxnFactory cnxnFactory = null;
        ServerCnxnFactory secureCnxnFactory = null;

        if (config.getClientPortAddress() != null) {
            cnxnFactory = ServerCnxnFactory.createFactory();
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
        }

        if (config.getSecureClientPortAddress() != null) {
            secureCnxnFactory = ServerCnxnFactory.createFactory();
            secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
        }

        quorumPeer = getQuorumPeer();
        quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));
        quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
        quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());
        //quorumPeer.setQuorumPeers(config.getAllMembers());
        quorumPeer.setElectionType(config.getElectionAlg());
        quorumPeer.setMyid(config.getServerId());
        quorumPeer.setTickTime(config.getTickTime());
        quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
        quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
        quorumPeer.setInitLimit(config.getInitLimit());
        quorumPeer.setSyncLimit(config.getSyncLimit());
        quorumPeer.setConnectToLearnerMasterLimit(config.getConnectToLearnerMasterLimit());
        quorumPeer.setObserverMasterPort(config.getObserverMasterPort());
        quorumPeer.setConfigFileName(config.getConfigFilename());
        quorumPeer.setClientPortListenBacklog(config.getClientPortListenBacklog());
        quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
        quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
        if (config.getLastSeenQuorumVerifier() != null) {
            quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
        }
        quorumPeer.initConfigInZKDatabase();
        quorumPeer.setCnxnFactory(cnxnFactory);
        quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
        quorumPeer.setSslQuorum(config.isSslQuorum());
        quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
        quorumPeer.setLearnerType(config.getPeerType());
        quorumPeer.setSyncEnabled(config.getSyncEnabled());
        quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
        if (config.sslQuorumReloadCertFiles) {
            quorumPeer.getX509Util().enableCertFileReloading();
        }
        quorumPeer.setMultiAddressEnabled(config.isMultiAddressEnabled());
        quorumPeer.setMultiAddressReachabilityCheckEnabled(config.isMultiAddressReachabilityCheckEnabled());
        quorumPeer.setMultiAddressReachabilityCheckTimeoutMs(config.getMultiAddressReachabilityCheckTimeoutMs());

        // sets quorum sasl authentication configurations
        quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
        if (quorumPeer.isQuorumSaslAuthEnabled()) {
            quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
            quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
            quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
            quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
            quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
        }
        quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
        quorumPeer.initialize();

        if (config.jvmPauseMonitorToRun) {
            quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
        }

        quorumPeer.start();
        ZKAuditProvider.addZKStartStopAuditLog();
        quorumPeer.join();
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Quorum Peer interrupted", e);
    } finally {
        try {
            metricsProvider.stop();
        } catch (Throwable error) {
            LOG.warn("Error while stopping metrics", error);
        }
    }
}
```

```java
public synchronized void start() { //校验ServerId是否合法
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    //载入之前持久化的一些信息 
    loadDataBase(); 
    //启动线程监听 
    startServerCnxnFactory(); 
    try {
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
    }
    //初始化选举投票以及算法 
    startLeaderElection(); 
    startJvmPauseMonitor();
    //当前也是一个线程，注意run方法 
    super.start();
}
```

我们已经知道了当一个节点启动时需要先发起选举寻找Leader节点，然后再根据Leader节点的事务信息进行同步，最后开始对外提供服务，这里我们先来看下初始化选举的逻辑，即上面的 startLeaderElection方法:

```java
synchronized public void startLeaderElection() {
    try {
        //所有节点启动的初始状态都是LOOKING，因此这里都会是创建一张投自己为Leader的票 
        if (getPeerState() == ServerState.LOOKING) {
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch(IOException e) {
        //异常处理 
    }
    //初始化选举算法，electionType默认为3
    this.electionAlg = createElectionAlgorithm(electionType);
}

protected Election createElectionAlgorithm(int electionAlgorithm) {
    Election le = null;
    switch (electionAlgorithm) {
        case 1:
        //忽略 
        case 2:
        //忽略 
        case 3:
            //electionAlgorithm默认是3，直接走到这里
            qcm = createCnxnManager();
            //监听选举事件的listener
            QuorumCnxManager.Listener listener = qcm.listener; 
            if(listener != null){
                //开启监听器
                listener.start();
                //初始化选举算法
                FastLeaderElection fle = new FastLeaderElection(this, qcm); 
                //发起选举
                fle.start();
                le = fle;
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
      default:
      //忽略 
    }
    return le; 
}

```

接下来，回到QuorumPeer类中start方法的最后一行super.start()，QuorumPeer本身也是一个线程类，一起来看下它的run方法:

```java
public void run() {
  try {
      while (running) {
          //根据当前节点的状态执行不同流程
          switch (getPeerState()) {
              case LOOKING:
                  try {
                      //寻找Leader节点
                      setCurrentVote(makeLEStrategy().lookForLeader());
                  } catch (Exception e) {
                      setPeerState(ServerState.LOOKING);
                  }
                  break;
              case OBSERVING:
                  try {
                      //当前节点启动模式为Observer
                      setObserver(makeObserver(logFactory));
                      //与Leader节点进行数据同步
                      observer.observeLeader();
                  } catch (Exception e) {
                  } finally {
                  }
                  break;
              case FOLLOWING:
                  try {
                      //当前节点启动模式为Follower
                      setFollower(makeFollower(logFactory));
                      //与Leader节点进行数据同步
                      follower.followLeader();
                  } catch (Exception e) {
                  } finally {
                  }
                  break;
              case LEADING:
                  try {
                      //当前节点启动模式为Leader
                      setLeader(makeLeader(logFactory));
                      //发送自己成为Leader的通知
                      leader.lead();
                      setLeader(null);
                  } catch (Exception e) {
                  } finally {
                  }
                  break;
          }
      }
  }
}
```

节点初始化的状态为LOOKING，因此启动时直接会调用lookForLeader方法发起Leader选举，一起看下:

```java
/**
 * 开始新一轮的leader选举。 每当我们的 QuorumPeer 状态更改为 LOOKING 时，就会调用此方法，并向所有其他对等方发送通知。
 * Starts a new round of leader election. Whenever our QuorumPeer
 * changes its state to LOOKING, this method is invoked, and it
 * sends notifications to all other peers.
 */
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        self.jmxLeaderElectionBean = null;
    }

    self.start_fle = Time.currentElapsedTime();
    try {
        /*
         * 当前领导人选举的选票存储在 recvset 中。
         * 换句话说，如果 v.electionEpoch == logicalclock，则投票 v 在 recvset 中。
         * 当前参与者使用 recvset 来推断是否大多数参与者都投票支持它。
         * The votes from the current leader election are stored in recvset. In other words, a vote v is in recvset
         * if v.electionEpoch == logicalclock. The current participant uses recvset to deduce on whether a majority
         * of participants has voted for it.
         */
        Map<Long, Vote> recvset = new HashMap<Long, Vote>();

        /*
         * 之前领导人选举的选票，以及当前领导人选举的选票都存储在 outofelection 中。
         * 请注意，处于 LOOKING 状态的通知不会存储在 outofelection 中。
         * 只有 FOLLOWING 或 LEADING 通知存储在 outofelection 中。
         * 当前参与者可以使用 outofelection 来了解哪个参与者是领导者，如果它在领导选举中迟到（即，逻辑时钟高于收到通知的选举纪元）。
         * The votes from previous leader elections, as well as the votes from the current leader election are
         * stored in outofelection. Note that notifications in a LOOKING state are not stored in outofelection.
         * Only FOLLOWING or LEADING notifications are stored in outofelection. The current participant could use
         * outofelection to learn which participant is the leader if it arrives late (i.e., higher logicalclock than
         * the electionEpoch of the received notifications) in a leader election.
         */
        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = minNotificationInterval;

        synchronized (this) {
            // ⾸先会将逻辑时钟⾃增，每进⾏⼀轮新的leader选举，都需要更新逻辑时钟
            logicalclock.incrementAndGet();
            // 更新选票（初始化选票）
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info("New election. My id = {}, proposed zxid=0x{}", self.getId(), Long.toHexString(proposedZxid));
        
        // 向其他服务器发送⾃⼰的选票（已更新的选票）
        sendNotifications();

        SyncedLearnerTracker voteSet;

        /*
         * 我们交换选票信息直到找到领导者的循环
         * Loop in which we exchange notifications until we find a leader
         */
        // 选举开始，当前机器选举状态是LOOKING，选举还未结果
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time
             */
            // 从recvqueue接收队列中取出其他机器的投票
            Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            // 没有获得选票
            if (n == null) {
                // manager已经发送了所有选票消息（表示有连接）
                if (manager.haveDelivered()) {
                    // 向所有其他服务器发送消息
                    sendNotifications();
                } else { // 还未发送所有消息（表示⽆连接）
                    // 连接其他每个服务器
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                int tmpTimeOut = notTimeout * 2;
                notTimeout = Math.min(tmpTimeOut, maxNotificationInterval);
                LOG.info("Notification time out: {}", notTimeout);
            } else if (validVoter(n.sid) && validVoter(n.leader)) {
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view for a replica in the current or next voting view.
                 */
                switch (n.state) {
                case LOOKING:
                    if (getInitLastLoggedZxid() == -1) {
                        LOG.debug("Ignoring notification as our zxid is -1");
                        break;
                    }
                    if (n.zxid == -1) {
                        LOG.debug("Ignoring notification from member with -1 zxid {}", n.sid);
                        break;
                    }
                    // If notification > current, replace and send messages out
                    // 其选举周期⼤于逻辑时钟
                    if (n.electionEpoch > logicalclock.get()) {
                        // 重新赋值逻辑时钟
                        logicalclock.set(n.electionEpoch);
                        // 清空所有接收到的所有选票
                        recvset.clear();
                        // 进⾏PK，选出较优的服务器
                        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            // 更新选票
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else { // ⽆法选出较优的服务器
                            // 更新选票
                            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                        }
                        // 发送本服务器的内部选票消息
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) {
                        // 选举周期⼩于逻辑时钟，不做处理，直接忽略
                            LOG.debug(
                                "Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x{}, logicalclock=0x{}",
                                Long.toHexString(n.electionEpoch),
                                Long.toHexString(logicalclock.get()));
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) { // PK，选出较优的服务器
                        // 更新选票
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        // 本服务器的内部选票消息
                        sendNotifications();
                    }

                    LOG.debug(
                        "Adding vote: from={}, proposed leader={}, proposed zxid=0x{}, proposed election epoch=0x{}",
                        n.sid,
                        n.leader,
                        Long.toHexString(n.zxid),
                        Long.toHexString(n.electionEpoch));

                    // recvset⽤于记录当前服务器在本轮次的Leader选举中收到的所有外部投票
                    // don't care about the version if it's in LOOKING state
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));

                    // 若能选出leader  
                    if (voteSet.hasAllQuorums()) {
                        // 遍历已经接收的投票集合
                        // Verify if there is any change in the proposed leader
                        while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                            // 选票有变更，⽐之前提议的Leader有更好的选票加⼊
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                // 将更优的选票放在recvset中
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                         // 表示之前提议的Leader已经是最优的
                        if (n == null) {
                            // 设置服务器状态
                            setPeerState(proposedLeader, voteSet);
                            // 最终的选票
                            Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                            // 清空recvqueue队列的选票
                            leaveInstance(endVote);
                            // 返回选票
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: {}", n.sid);
                    break;
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if (n.electionEpoch == logicalclock.get()) {
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        if (voteSet.hasAllQuorums() && checkLeader(recvset, n.leader, n.electionEpoch)) {
                            setPeerState(n.leader, voteSet);
                            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify that
                     * a majority are following the same leader.
                     *
                     * Note that the outofelection map also stores votes from the current leader election.
                     * See ZOOKEEPER-1732 for more information.
                     */
                    outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

                    if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                        synchronized (this) {
                            logicalclock.set(n.electionEpoch);
                            setPeerState(n.leader, voteSet);
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecognized: {} (n.state), {}(n.sid)", n.state, n.sid);
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if (self.jmxLeaderElectionBean != null) {
                MBeanRegistry.getInstance().unregister(self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
        LOG.debug("Number of connection processing threads: {}", manager.getConnectionThrea
```

经过上面的发起投票，统计投票信息最终每个节点都会确认自己的身份，节点根据类型的不同会执行以下逻辑:

1. 如果是Leader节点，首先会想其他节点发送一条NEWLEADER信息，确认自己的身份，等到各个节 点的ACK消息以后开始正式对外提供服务，同时开启新的监听器，处理新节点加入的逻辑。
2. 如果是Follower节点，首先向Leader节点发送一条FOLLOWERINFO信息，告诉Leader节点自己已 处理的事务的最大Zxid，然后Leader节点会根据自己的最大Zxid与Follower节点进行同步，如果 Follower节点落后的不多则会收到Leader的DIFF信息通过内存同步，如果Follower节点落后的很 多则会收到SNAP通过快照同步，如果Follower节点的Zxid大于Leader节点则会收到TRUNC信息忽 略多余的事务。
3. 如果是Observer节点，则与Follower节点相同
