
# 注册中心

服务注册中⼼本质上是为了解耦服务提供者和服务消费者。为了⽀持弹性扩缩容特性，⼀个微服务的提供者的数量和分布往往是动态变化的，也是⽆法预先确定的。因此需要引⼊服务注册中⼼。

> 为了⽀持弹性扩缩容特性，需要注册中心来解决
## 原理

- 服务提供者启动，将相关服务信息主动注册到注册中⼼
- 服务消费者获取服务注册信息：
    - poll模式：服务消费者可以主动拉取可⽤的服务提供者清单
    - push模式：服务消费者订阅服务（当服务提供者有变化时，注册中⼼也会主动推送更新后的服务清单给消费者
- 服务消费者直接调⽤服务提供者

另外，注册中⼼也需要完成服务提供者的健康监控，当发现服务提供者失效时需要及时剔除；

> 服务提供者上报注册信息
> 服务消费者获取注册信息 

## 主流服务中⼼对⽐

### Zookeeper

Zookeeper本质 = 存储 + 监听通知。

Zookeeper ⽤来做服务注册中⼼，主要是因为它具有**节点变更通知功能，只要客户端监听相关服务节点，服务节点的所有变更，都能及时的通知到监听客户端**，这样作为调⽤⽅只要使⽤ Zookeeper 的客户端就能实现服务节点的订阅和变更通知功能了。 Zookeeper 可⽤性也可以，只要半数以上的选举节点存活，整个集群就是可⽤的。 

> Leader选举期间短暂不可用
### Eureka

[[Spring Cloud#Eureka]]

> AP架构，高可用

### Consul

Consul是由HashiCorp基于Go语⾔开发的⽀持多数据中⼼分布式⾼可⽤的服务发布和注册服务软件， 采⽤Raft算法保证服务的⼀致性，且⽀持健康检查。

### Nacos

Nacos是⼀个更易于构建云原⽣应⽤的动态服务发现、配置管理和服务管理平台。简单来说 Nacos 就是 注册中⼼ + 配置中⼼的组合，帮助我们解决微服务开发必会涉及到的服务注册 与发现，服务配置，服务管理等问题。 Nacos 是Spring Cloud Alibaba 核⼼组件之⼀，负责服务注册与发现，还有配置。
### 对比

|组件名|语⾔|CAP|对外暴露接⼝|
|---|---|---|---|
|Eureka|Java|AP（⾃我保护机制，保证可⽤）|HTTP|
|Consul|Go|CP|HTTP/DNS|
|Zookeeper|Java|CP|客户端|
|Nacos|Java|⽀持AP/CP切换|HTTP|

# 分布式配置中⼼
## 应⽤场景

集中式管理分布式微服务的配置信息，运行期间动态调整配置，⽐如：根据各个微服务的负载情况，动态调整数据源连接池⼤⼩。

场景总结如下：

- 集中配置管理，⼀个微服务架构中可能有成百上千个微服务（⼀次修改、到处⽣效）
- 不同环境不同配置，⽐如数据源配置在不同环境（dev,test,prod）中是不同的
- 运⾏期间可动态调整。如配置内容发⽣变化，微服务可以⾃动更新配置

## 原理
TODO

# 分布式事务
- XA 方案
- TCC 方案
- SAGA 方案
- 本地消息表
- 可靠消息最终一致性方案
- 最大努力通知方案

## 两阶段提交方案/XA 方案

所谓的 XA 方案，即两阶段提交，有一个**事务管理器**的概念，负责协调多个数据库（**资源管理器**）的事务，事务管理器先问问各个数据库你准备好了吗？如果每个数据库都回复 ok，那么就正式提交事务，在各个数据库上执行操作；如果任何其中一个数据库回答不 ok，那么就回滚事务。

这种分布式事务方案，比较适合单块应用里，跨多个库的分布式事务，而且因为严重依赖于数据库层面来搞定复杂的事务，效率很低，**绝对不适合高并发的场景**。参考 `Spring + JTA` 就可以搞定。

> 一般来说某个系统内部如果出现跨多个库的这么一个操作，是不合规的。正常规范要求**每个服务只能操作自己对应的一个数据库**。操作别的服务需要通过接口。

[![distributed-transacion-XA](https://github.com/doocs/advanced-java/raw/main/docs/distributed-system/images/distributed-transaction-XA.png)](https://github.com/doocs/advanced-java/blob/main/docs/distributed-system/images/distributed-transaction-XA.png)

## TCC 方案

TCC 的全称是： `Try` 、 `Confirm` 、 `Cancel` 。

- Try 阶段：这个阶段说的是对各个服务的资源做检测以及对资源进行**锁定或者预留**。
- Confirm 阶段：这个阶段说的是在各个服务中**执行实际的操作**。
- Cancel 阶段：如果任何一个服务的业务方法执行出错，那么这里就需要**进行补偿**，就是执行已经执行成功的业务逻辑的回滚操作。

一般来说**支付**、**交易**相关的场景，我们会用 TCC，严格保证资金的正确性。

> 业务侵入性大，事务回滚依赖于开发者来回滚和补偿，代码量大，复杂度高。


[![distributed-transacion-TCC](distributed-transaction-TCC.png)
## Saga 方案

金融核心等业务可能会选择 TCC 方案，以追求强一致性和更高的并发量，而对于更多的金融核心以上的业务系统往往会选择补偿事务，补偿事务处理在 30 多年前就提出了 Saga 理论。目前业界比较公认的是采用 Saga 作为长事务的解决方案。

#### 基本原理

业务流程中每个参与者都提交本地事务，若某一个参与者失败，则补偿前面已经成功的参与者。下图左侧是正常的事务流程，当执行到 T3 时发生了错误，则开始执行右边的事务补偿流程，反向执行 T3、T2、T1 的补偿服务 C3、C2、C1，将 T3、T2、T1 已经修改的数据补偿掉。

[![distributed-transacion-TCC](distributed-transaction-saga.png)]

#### 使用场景

对于一致性要求高、短流程、并发高的场景，如：金融核心系统，会优先考虑 TCC 方案。而在另外一些场景下，我们并不需要这么强的一致性，只需要保证最终一致性即可。

比如很多金融核心以上的业务（渠道层、产品层、系统集成层），这些系统的特点是最终一致即可、流程多、流程长、还可能要调用其它公司的服务。这种情况如果选择 TCC 方案开发的话，一来成本高，二来无法要求其它公司的服务也遵循 TCC 模式。同时流程长，事务边界太长，加锁时间长，也会影响并发性能。

所以 Saga 模式的适用场景是：

- 业务流程长、业务流程多；
- 参与者包含其它公司或遗留系统服务，无法提供 TCC 模式要求的三个接口。

#### 优势
- 一阶段提交本地事务，无锁，高性能；
- 参与者可异步执行，高吞吐；
- 补偿服务易于实现，因为一个更新操作的反向操作是比较容易理解的。

#### 缺点

- 不保证事务的隔离性。

### 本地消息表

1. A 系统在自己本地一个事务里操作同时，插入一条数据到消息表；
2. 接着 A 系统将这个消息发送到 MQ 中去；
3. B 系统接收到消息之后，在一个事务里，往自己本地消息表里插入一条数据，同时执行其他的业务操作，如果这个消息已经被处理过了，那么此时这个事务会回滚，这样**保证不会重复处理消息**；
4. B 系统执行成功之后，就会更新自己本地消息表的状态以及 A 系统消息表的状态；
5. 如果 B 系统处理失败了，那么就不会更新消息表状态，那么此时 A 系统会定时扫描自己的消息表，如果有未处理的消息，会再次发送到 MQ 中去，让 B 再次处理；
6. 这个方案保证了最终一致性，哪怕 B 事务失败了，但是 A 会不断重发消息，直到 B 那边成功为止。

**严重依赖于数据库的消息表来管理事务**，不适合高并发场景。

[![distributed-transaction-local-message-table](distributed-transaction-local-message-table.png)

## 可靠消息最终一致性方案

基于 MQ 来实现事务。比如 RocketMQ 就支持消息事务。

[[MQ#RocketMQ如何实现的分布式事务？]]

## 最大努力通知方案

1. 系统 A 本地事务执行完之后，发送个消息到 MQ；
2. 这里会有个专门消费 MQ 的**最大努力通知服务**，这个服务会消费 MQ 然后写入数据库中记录下来，或者是放入个内存队列也可以，接着调用系统 B 的接口；
3. 要是系统 B 执行成功就 ok 了；要是系统 B 执行失败了，那么最大努力通知服务就定时尝试重新调用系统 B，反复 N 次，最后还是不行就放弃。

## SETA的XA和AT模式
TODO
# 分布式锁

### 数据库

通过唯一索引实现。

### [Redis 分布式锁](https://github.com/doocs/advanced-java/blob/main/docs/distributed-system/distributed-lock-redis-vs-zookeeper.md#redis-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81)

官方叫做 `RedLock` 算法，是 Redis 官方支持的分布式锁算法。

这个分布式锁有 3 个重要的考量点：

- 互斥（只能有一个客户端获取锁）
- 不能死锁
- 容错（只要大部分 Redis 节点创建了这把锁就可以）

#### Redis 最普通的分布式锁

第一个最普通的实现方式，就是在 Redis 里使用 `SET key value [EX seconds] [PX milliseconds] NX` 创建一个 key，这样就算加锁。

比如执行以下命令：

```r
SET resource_name my_random_value PX 30000 NX
```

释放锁就是删除 key ，但是一般可以用 `lua` 脚本删除，判断 value 一样才删除：

```lua
-- 删除锁的时候，找到 key 对应的 value，跟自己传过去的 value 做比较，如果是一样的才删除。
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

为啥要用 `random_value` 随机值呢？因为如果某个客户端获取到了锁，但是阻塞了很长时间才执行完，比如说超过了 30s，此时可能已经自动释放锁了，此时可能别的客户端已经获取到了这个锁，要是你这个时候直接删除 key 的话会有问题，所以得用随机值加上面的 `lua` 脚本来释放锁。

但是这样是肯定不行的。因为如果是普通的 Redis 单实例，那就是单点故障。或者是 Redis 普通主从，那 Redis 主从异步复制，如果主节点挂了（key 就没有了），key 还没同步到从节点，此时从节点切换为主节点，别人就可以 set key，从而拿到锁。

#### [RedLock 算法](https://github.com/doocs/advanced-java/blob/main/docs/distributed-system/distributed-lock-redis-vs-zookeeper.md#redlock-%E7%AE%97%E6%B3%95)

这个场景是假设有一个 Redis cluster，有 5 个 Redis master 实例。然后执行如下步骤获取一把锁：

1. 获取当前时间戳，单位是毫秒；
2. 跟上面类似，轮流尝试在每个 master 节点上创建锁，超时时间较短，一般就几十毫秒（客户端为了获取锁而使用的超时时间比自动释放锁的总时间要小。例如，如果自动释放时间是 10 秒，那么超时时间可能在 `5~50` 毫秒范围内）；
3. 尝试在**大多数节点**上建立一个锁，比如 5 个节点就要求是 3 个节点 `n / 2 + 1` ；
4. 客户端计算建立好锁的时间，如果建立锁的时间小于超时时间，就算建立成功了；
5. 要是锁建立失败了，那么就依次之前建立过的锁删除；
6. 只要别人建立了一把分布式锁，你就得**不断轮询去尝试获取锁**。

[![redis-redlock](https://github.com/doocs/advanced-java/raw/main/docs/distributed-system/images/redis-redlock.png)](https://github.com/doocs/advanced-java/blob/main/docs/distributed-system/images/redis-redlock.png)

[Redis 官方](https://redis.io/)给出了以上两种基于 Redis 实现分布式锁的方法，详细说明可以查看：[https://redis.io/topics/distlock](https://redis.io/topics/distlock) 。

### [zk 分布式锁](https://github.com/doocs/advanced-java/blob/main/docs/distributed-system/distributed-lock-redis-vs-zookeeper.md#zk-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81)

zk 分布式锁，其实可以做的比较简单，就是某个节点尝试创建临时 znode，此时创建成功了就获取了这个锁；这个时候别的客户端来创建锁会失败，只能**注册个监听器**监听这个锁。释放锁就是删除这个 znode，一旦释放掉就会通知客户端，然后有一个等待着的客户端就可以再次重新加锁。

```java
/**
 * ZooKeeperSession
 */
public class ZooKeeperSession {

    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);

    private ZooKeeper zookeeper;
    private CountDownLatch latch;

    public ZooKeeperSession() {
        try {
            this.zookeeper = new ZooKeeper("192.168.31.187:2181,192.168.31.19:2181,192.168.31.227:2181", 50000, new ZooKeeperWatcher());
            try {
                connectedSemaphore.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("ZooKeeper session established......");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取分布式锁
     *
     * @param productId
     */
    public Boolean acquireDistributedLock(Long productId) {
        String path = "/product-lock-" + productId;

        try {
            zookeeper.create(path, "".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
            return true;
        } catch (Exception e) {
            while (true) {
                try {
                    // 相当于是给node注册一个监听器，去看看这个监听器是否存在
                    Stat stat = zk.exists(path, true);

                    if (stat != null) {
                        this.latch = new CountDownLatch(1);
                        this.latch.await(waitTime, TimeUnit.MILLISECONDS);
                        this.latch = null;
                    }
                    zookeeper.create(path, "".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
                    return true;
                } catch (Exception ee) {
                    continue;
                }
            }

        }
        return true;
    }

    /**
     * 释放掉一个分布式锁
     *
     * @param productId
     */
    public void releaseDistributedLock(Long productId) {
        String path = "/product-lock-" + productId;
        try {
            zookeeper.delete(path, -1);
            System.out.println("release the lock for product[id=" + productId + "]......");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 建立 zk session 的 watcher
     */
    private class ZooKeeperWatcher implements Watcher {

        public void process(WatchedEvent event) {
            System.out.println("Receive watched event: " + event.getState());

            if (KeeperState.SyncConnected == event.getState()) {
                connectedSemaphore.countDown();
            }

            if (this.latch != null) {
                this.latch.countDown();
            }
        }

    }

    /**
     * 封装单例的静态内部类
     */
    private static class Singleton {

        private static ZooKeeperSession instance;

        static {
            instance = new ZooKeeperSession();
        }

        public static ZooKeeperSession getInstance() {
            return instance;
        }

    }

    /**
     * 获取单例
     *
     * @return
     */
    public static ZooKeeperSession getInstance() {
        return Singleton.getInstance();
    }

    /**
     * 初始化单例的便捷方法
     */
    public static void init() {
        getInstance();
    }

}
```

也可以采用另一种方式，创建临时顺序节点：

如果有一把锁，被多个人给竞争，此时多个人会排队，第一个拿到锁的人会执行，然后释放锁；后面的每个人都会去监听**排在自己前面**的那个人创建的 node 上，一旦某个人释放了锁，排在自己后面的人就会被 ZooKeeper 给通知，一旦被通知了之后，就 ok 了，自己就获取到了锁，就可以执行代码了。

```java
public class ZooKeeperDistributedLock implements Watcher {

    private ZooKeeper zk;
    private String locksRoot = "/locks";
    private String productId;
    private String waitNode;
    private String lockNode;
    private CountDownLatch latch;
    private CountDownLatch connectedLatch = new CountDownLatch(1);
    private int sessionTimeout = 30000;

    public ZooKeeperDistributedLock(String productId) {
        this.productId = productId;
        try {
            String address = "192.168.31.187:2181,192.168.31.19:2181,192.168.31.227:2181";
            zk = new ZooKeeper(address, sessionTimeout, this);
            connectedLatch.await();
        } catch (IOException e) {
            throw new LockException(e);
        } catch (KeeperException e) {
            throw new LockException(e);
        } catch (InterruptedException e) {
            throw new LockException(e);
        }
    }

    public void process(WatchedEvent event) {
        if (event.getState() == KeeperState.SyncConnected) {
            connectedLatch.countDown();
            return;
        }

        if (this.latch != null) {
            this.latch.countDown();
        }
    }

    public void acquireDistributedLock() {
        try {
            if (this.tryLock()) {
                return;
            } else {
                waitForLock(waitNode, sessionTimeout);
            }
        } catch (KeeperException e) {
            throw new LockException(e);
        } catch (InterruptedException e) {
            throw new LockException(e);
        }
    }

    public boolean tryLock() {
        try {
            // 传入进去的locksRoot + “/” + productId
            // 假设productId代表了一个商品id，比如说1
            // locksRoot = locks
            // /locks/10000000000，/locks/10000000001，/locks/10000000002
            lockNode = zk.create(locksRoot + "/" + productId, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

            // 看看刚创建的节点是不是最小的节点
            // locks：10000000000，10000000001，10000000002
            List<String> locks = zk.getChildren(locksRoot, false);
            Collections.sort(locks);

            if (lockNode.equals(locksRoot + "/" + locks.get(0))) {
                // 如果是最小的节点,则表示取得锁
                return true;
            }

            // 如果不是最小的节点，找到比自己小1的节点
            int previousLockIndex = -1;
            for (int i = 0; i < locks.size(); i++) {
                if (lockNode.equals(locksRoot + "/" +locks.get(i))){
                    previousLockIndex = i - 1;
                    break;
                }
            }

            this.waitNode = locks.get(previousLockIndex);
        } catch (KeeperException e) {
            throw new LockException(e);
        } catch (InterruptedException e) {
            throw new LockException(e);
        }
        return false;
    }

    private boolean waitForLock(String waitNode, long waitTime) throws InterruptedException, KeeperException {
        Stat stat = zk.exists(locksRoot + "/" + waitNode, true);
        if (stat != null) {
            this.latch = new CountDownLatch(1);
            this.latch.await(waitTime, TimeUnit.MILLISECONDS);
            this.latch = null;
        }
        return true;
    }

    public void unlock() {
        try {
            // 删除/locks/10000000000节点
            // 删除/locks/10000000001节点
            System.out.println("unlock " + lockNode);
            zk.delete(lockNode, -1);
            lockNode = null;
            zk.close();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    public class LockException extends RuntimeException {
        private static final long serialVersionUID = 1L;

        public LockException(String e) {
            super(e);
        }

        public LockException(Exception e) {
            super(e);
        }
    }
}
```

但是，使用 zk 临时节点会存在另一个问题：由于 zk 依靠 session 定期的心跳来维持客户端，如果客户端进入长时间的 GC，可能会导致 zk 认为客户端宕机而释放锁，让其他的客户端获取锁，但是客户端在 GC 恢复后，会认为自己还持有锁，从而可能出现多个客户端同时获取到锁的情形。[#209](https://github.com/doocs/advanced-java/issues/209)

针对这种情况，可以通过 JVM 调优，尽量避免长时间 GC 的情况发生。

### [redis 分布式锁和 zk 分布式锁的对比](https://github.com/doocs/advanced-java/blob/main/docs/distributed-system/distributed-lock-redis-vs-zookeeper.md#redis-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E5%92%8C-zk-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%E5%AF%B9%E6%AF%94)

- redis 分布式锁，其实**需要自己不断去尝试获取锁**，比较消耗性能。
- zk 分布式锁，获取不到锁，注册个监听器即可，不需要不断主动尝试获取锁，性能开销较小。

另外一点就是，如果是 Redis 获取锁的那个客户端 出现 bug 挂了，那么只能等待超时时间之后才能释放锁；而 zk 的话，因为创建的是临时 znode，只要客户端挂了，znode 就没了，此时就自动释放锁。

Redis 分布式锁大家没发现好麻烦吗？遍历上锁，计算时间等等......zk 的分布式锁语义清晰实现简单。

所以先不分析太多的东西，就说这两点，我个人实践认为 zk 的分布式锁比 Redis 的分布式锁牢靠、而且模型简单易用。

# 分布式ID


# 熔断限流

## 微服务中的雪崩效应

- 扇⼊：代表着该微服务被调⽤的次数，扇⼊⼤，说明该模块复⽤性好
- 扇出：该微服务调⽤其他微服务的个数，扇出⼤，说明业务逻辑复杂

> 扇⼊⼤是⼀个好事，扇出⼤不⼀定是好事

## 雪崩效应解决⽅案

我们介绍三种技术⼿段应对微服务中的雪崩效应，这三种⼿段都是从系统可⽤性、可靠性⻆度出发，尽量防⽌系统整体缓慢甚⾄瘫痪。

* 服务熔断
* 服务降级
* 服务限流

### 服务熔断

**服务熔断**机制是应对雪崩效应的⼀种微服务链路保护机制。在微服务架构中，熔断机制也是起着类似的作⽤。当扇出链路的某个微服务不可⽤或者响应时间太⻓时，熔断该节点微服务的调⽤，进⾏服务的降级，快速返回错误的响应信息。当检测到该节点微服务调⽤响应正常后，恢复调⽤链路。

注意：

- 服务熔断重点在“断”，切断对下游服务的调⽤
- 服务熔断和服务降级往往是⼀起使⽤的， Hystrix就是这样。

### 服务降级

通俗讲就是整体资源不够⽤了，先将⼀些不关紧的服务停掉（调⽤我的时候，给你返回⼀个预留的值，也叫做兜底数据），待⾼峰过去，再把那些服务打开。

服务降级⼀般是从整体考虑，就是当某个服务熔断之后，服务器将不再被调⽤，此刻客户端可以⾃⼰准备⼀个本地的fallback回调，返回⼀个缺省值，这样做，虽然服务⽔平下降，但保证核心服务可用。

### 服务限流

服务降级是当服务出问题或者影响到核⼼流程的性能时，暂时将服务屏蔽掉，待⾼峰或者问题解决后再打开；但是**有些场景并不能⽤服务降级来解决，⽐如项目中的核⼼功能**，这个时候可以结合服务限流来限制这些场景的并发/请求量限流措施也很多，⽐如

- 限制总并发数（⽐如数据库连接池、线程池）
- 限制瞬时并发数（如nginx限制瞬时并发连接数）
- 限制时间窗⼝内的平均速率（如Guava的RateLimiter、 nginx的limit_req模块，限制每秒的平均速率）
- 限制远程接⼝调⽤速率、限制MQ的消费速率等

# 我们的设计
- 注册中心  
	- 容灾
	- 对等集群
	- 配置管理
- RPC  
    - 调用方式
        - 同步调用
        - 异步调用
        - 并行调用  
    - 序列化与反序列化  
        - protobuf  
        - json  
        - thrift  
    - 通信协议
        - Http
        - TCP
- 服务熔断  
- 服务降级  
	- 屏蔽降级
	- 容错降级
- APM  
	- 接口性能KPI统计
	- 接口日志
	- 链路监控告警
- 服务路由  
	- 负载均衡
		- 随机
		- 轮询
		- 权重
		- 动态权重
	- 缓存
	- 发布订阅
- 集群容错
	- 自动切换
	- 失败通知
	- 失败缓存重试
	- 快速失败
- 流量控制
	- 静态流控
	- 动态流控
	- 分级流控
	- 并发流控
	- 连接数控制