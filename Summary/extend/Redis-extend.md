# Redis的数据结构
# Redis慢日志查询

## 慢查询设置

在`redis.conf`中可以配置和慢查询日志相关的选项:

```bash
#执行时间超过多少微秒的命令请求会被记录到日志上 0 :全记录 <0 不记录 
slowlog-log-slower-than 10000
#slowlog-max-len 存储慢查询日志条数
slowlog-max-len 128
```

Redis使用列表存储慢查询日志，采用队列方式(FIFO)

config set的方式可以临时设置，redis重启后就无效

- `config set slowlog-log-slower-than 微秒`
- `config set slowlog-max-len 条数`

查看日志：`slowlog get [n]`

```bash
127.0.0.1:6379> config set slowlog-log-slower-than 0
OK
127.0.0.1:6379> config set slowlog-max-len 2
OK
127.0.0.1:6379> set name:001 zhaoyun
OK
127.0.0.1:6379> set name:002 zhangfei
OK
127.0.0.1:6379> get name:002
"zhangfei"
127.0.0.1:6379> slowlog get
1) 1) (integer) 7 #日志的唯一标识符(uid)
   2) (integer) 1589774302 #命令执行时的UNIX时间戳
   3) (integer) 65  #命令执行的时长(微秒) 
   4) 1) "get" #执行命令及参数
      2) "name:002" 
   5) "127.0.0.1:37277"
   6) ""
2) 1) (integer) 6
   2) (integer) 1589774281
   3) (integer) 7
   4) 1) "set"
      2) "name:002"
      3) "zhangfei"
   5) "127.0.0.1:37277"
   6) ""

 # set和get都记录，第一条被移除了。
```

## Redis慢查询源码

在redisServer中保存和慢查询日志相关的信息

```c
struct redisServer {
    // ...
    // 下一条慢查询日志的 ID
    long long slowlog_entry_id;
    // 保存了所有慢查询日志的链表 FIFO 
    list *slowlog;
    // 服务器配置 slowlog-log-slower-than 选项的值 
    long long slowlog_log_slower_than;
    // 服务器配置 slowlog-max-len 选项的值 
    unsigned long slowlog_max_len;
    // ...
};
```

lowlog 链表保存了服务器中的所有慢查询日志， 链表中的每个节点都保存了一个 slowlogEntry 结构， 每个 slowlogEntry 结构代表一条慢查询日志。

```c
typedef struct slowlogEntry { 
    // 唯一标识符
    long long id;
    // 命令执行时的时间，格式为 UNIX 时间戳
    time_t time;
    // 执行命令消耗的时间，以微秒为单位
    long long duration; 
    // 命令与命令参数
    robj **argv;
    // 命令与命令参数的数量
    int argc;
} slowlogEntry;
```

### 慢查询日志的获取&删除

初始化日志列表

```c
void slowlogInit(void) {
    server.slowlog = listCreate(); /* 创建一个list列表 */ 
    server.slowlog_entry_id = 0;     /* 日志ID从0开始 */ 
    listSetFreeMethod(server.slowlog,slowlogFreeEntry);     /* 指定慢查询日志list空间的释放方法 */ 
}
```

获得慢查询日志记录 `slowlog get [n]`

```c
def SLOWLOG_GET(number=None):
    // 用户没有给定 number 参数
    // 那么打印服务器包含的全部慢查询日志 
    if number is None:
        number = SLOWLOG_LEN() 
    // 遍历服务器中的慢查询日志
    for log in redisServer.slowlog:
        if number <= 0:
           // 打印的日志数量已经足够，跳出循环
            break 
        else:
          // 继续打印，将计数器的值减一 
          number -= 1
        // 打印日志 
        printLog(log) 
```

查看日志数量的 `slowlog len`

```c
def SLOWLOG_LEN():
    // slowlog 链表的长度就是慢查询日志的条目数量 
    return len(redisServer.slowlog)
```

清除日志 `slowlog reset`

```C
def SLOWLOG_RESET():
// 遍历服务器中的所有慢查询日志
for log in redisServer.slowlog:
    // 删除日志 
    deleteLog(log)
```

### 添加日志实现

在每次执行命令的之前和之后， 程序都会记录微秒格式的当前 UNIX 时间戳， 这两个时间戳之间的差就是服务器执行命令所耗费的时长， 服务器会将这个时长作为参数之一传给`slowlogPushEntryIfNeeded`函数， 而`slowlogPushEntryIfNeeded` 函数则负责检查是否需要为这次执行的命令创建慢查询日志

```c
// 记录执行命令前的时间
before = unixtime_now_in_us()
//执行命令
execute_command(argv, argc, client)
//记录执行命令后的时间
after = unixtime_now_in_us()
// 检查是否需要创建新的慢查询日志 
slowlogPushEntryIfNeeded(argv, argc, before-after)
void slowlogPushEntryIfNeeded(robj **argv, int argc, long long duration) {
    if (server.slowlog_log_slower_than < 0) return; /* Slowlog disabled */ /* 负数表示禁用 */
      if (duration >= server.slowlog_log_slower_than) /* 如果执行时间 > 指定阈值*/
        listAddNodeHead(server.slowlog,slowlogCreateEntry(argv,argc,duration)); /* 创建一个slowlogEntry对象,添加到列表首部*/
    while (listLength(server.slowlog) > server.slowlog_max_len) /* 如果列表长度 > 指定长度 */
        listDelNode(server.slowlog,listLast(server.slowlog)); /* 移除列表尾部元素*/
}
```

`slowlogPushEntryIfNeeded`函数的作用有两个:

1. 检查命令的执行时长是否超过 `slowlog-log-slower-than` 选项所设置的时间， 如果是的话， 就为命令创建一个新的日志， 并将新日志添加到 slowlog 链表的表头。
2. 检查慢查询日志的长度是否超过 `slowlog-max-len` 选项所设置的长度， 如果是的话， 那么将多出来的日志从 slowlog 链表中删除掉。

## 慢查询定位&处理

使用`slowlog get` 可以获得执行较慢的redis命令，针对该命令可以进行优化:

1. 尽量使用短的key，对于value有些也可精简，能使用int就int。
2. 避免使用keys \*、hgetall等全量操作。
3. 减少大key的存取，打散为小key
4. 将rdb改为aof模式，rdb fork 子进程，数据量过大主进程阻塞，redis性能大幅下降，关闭持久化 ， (适合于数据量较小) 改aof 命令式
5. 想要一次添加多条数据的时候可以使用管道
6. 尽可能地使用哈希存储
7. 尽量限制下redis使用的内存大小，这样可以避免redis使用swap分区或者出现OOM错误，避免内存与硬盘的swap

# Redis监视器

Redis客户端通过执行`MONITOR`命令可以将自己变为一个监视器，实时地接受并打印出服务器当前处理的命令请求的相关信息。

此时，当其他客户端向服务器发送一条命令请求时，服务器除了会处理这条命令请求之外，还会将这条命令请求的信息发送给所有监视器。

![](Redis监视器.jpg)

Redis客户端1

```bash
127.0.0.1:6379> monitor
OK
1589706136.030138 [0 127.0.0.1:42907] "COMMAND"
1589706145.763523 [0 127.0.0.1:42907] "set" "name:10" "zhaoyun"
1589706163.756312 [0 127.0.0.1:42907] "get" "name:10"
```

Redis客户端2

```bash
127.0.0.1:6379>
127.0.0.1:6379> set name:10 zhaoyun
OK
127.0.0.1:6379> get name:10
"zhaoyun"
```

## 实现监视器

redisServer 维护一个 monitors 的链表，记录自己的监视器，每次收到 MONITOR 命令之后，将客户端追加到链表尾。

```c
void monitorCommand(redisClient *c) {
    /* ignore MONITOR if already slave or in monitor mode */
    if (c->flags & REDIS_SLAVE) return;
}
c->flags |= (REDIS_SLAVE|REDIS_MONITOR); listAddNodeTail(server.monitors,c); addReply(c,shared.ok); //回复OK
```

## 向监视器发送命令信息

利用call函数实现向监视器发送命令

```c
// call() 函数是执行命令的核心函数，这里只看监视器部分 /*src/redis.c/call*/
/* Call() is the core of Redis execution of a command */ 
void call(redisClient *c, int flags) {
    long long dirty, start = ustime(), duration;
    int client_old_flags = c->flags;
    /* Sent the command to clients in MONITOR mode, only if the commands are
    * not generated from reading an AOF. */
    if (listLength(server.monitors) && !server.loading && !(c->cmd->flags & REDIS_CMD_SKIP_MONITOR)) {
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
    }
    ...... 
}
```

call 主要调用了`replicationFeedMonitors`，这个函数的作用就是将命令打包为协议，发送给监视器。

## Redis监控平台

- **Grafana** 是一个开箱即用的可视化工具，具有功能齐全的度量仪表盘和图形编辑器，有灵活丰富的图形化选项，可以混合多种风格，支持多个数据源特点。
- **Prometheus**是一个开源的服务监控系统，它通过HTTP协议从远程的机器收集数据并存储在本地的时序数据库上。
- **redis_exporter**为Prometheus提供了redis指标的导出，配合Prometheus以及grafana进行可视化及监控。

![](Redis监控.png)

## Redis集群的搭建

该部分可以参考这篇文章：[https://juejin.cn/post/6922690589347545102#heading-1](https://juejin.cn/post/6922690589347545102#heading-1)

Redis集群的搭建可以分为以下几个部分：

1. 启动节点：将节点以集群模式启动，读取或者生成集群配置文件，此时节点是独立的。
2. 节点握手：节点通过gossip协议通信，将独立的节点连成网络，主要使用meet命令。
3. 槽指派：将16384个槽位分配给主节点，以达到分片保存数据库键值对的效果。

# Redis集群的运维

**1、数据迁移问题：**

Redis集群可以进行节点的动态扩容缩容，这一过程目前还处于半自动状态，需要人工介入。在扩缩容的时候，需要进行数据迁移。而 Redis为了保证迁移的一致性，迁移所有操作都是同步操作，执行迁移时，两端的 Redis均会进入时长不等的阻塞状态，对于小Key，该时间可以忽略不计，但如果一旦Key的内存使用过大，严重的时候会接触发集群内的故障转移，造成不必要的切换。

**2、带宽消耗问题：**

Redis集群是无中心节点的集群架构，依靠Gossip协议协同自动化修复集群的状态，但goosip有消息延时和消息冗余的问题，在集群节点数量过多的时候，goosip协议通信会消耗大量的带宽，主要体现在以下几个方面：

- 消息发送频率：跟cluster-node-timeout密切相关，当节点发现与其他节点的最后通信时间超过 cluster-node-timeout/2时会直接发送ping消息
- 消息数据量：每个消息主要的数据占用包含：slots槽数组（2kb）和整个集群1/10的状态数据
- 节点部署的机器规模：机器的带宽上限是固定的，因此相同规模的集群分布的机器越多，每台机器划分的节点越均匀，则整个集群内整体的可用带宽越高

集群带宽消耗主要分为：读写命令消耗+Gossip消息消耗，因此搭建Redis集群需要根据业务数据规模和消息通信成本做出合理规划：

- 在满足业务需求的情况下尽量避免大集群，同一个系统可以针对不同业务场景拆分使用若干个集群。
- 适度提供cluster-node-timeout降低消息发送频率，但是cluster-node-timeout还影响故障转移的速度，因此需要根据自身业务场景兼顾二者平衡
- 如果条件允许尽量均匀部署在更多机器上，避免集中部署。如果有60个节点的集群部署在3台机器上每台20个节点，这是机器的带宽消耗将非常严重

**3、Pub/Sub广播问题：**

集群模式下内部对所有publish命令都会向所有节点进行广播，加重带宽负担，所以集群应该避免频繁使用Pub/sub功能

**4、集群倾斜：**

集群倾斜是指不同节点之间数据量和请求量出现明显差异，这种情况将加大负载均衡和开发运维的难度。因此需要理解集群倾斜的原因

1. 数据倾斜：

	- 节点和槽分配不均
	- 不同槽对应键数量差异过大
	- 集合对象包含大量元素
	- 内存相关配置不一致

2. 请求倾斜：

	合理设计键，热点大集合对象做拆分或者使用hmget代替hgetall避免整体读取

**5、集群读写分离：**

集群模式下读写分离成本比较高，直接扩展主节点数量来提高集群性能是更好的选择。

[Redis主从复制原理](https://www.cnblogs.com/hepingqingfeng/p/7263782.html)


# Lua脚本

lua是一种轻量小巧的**脚本语言**，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。

## Lua环境协作组件

从Redis2.6.0版本开始，通过**内置的lua编译/解释器**，可以使用EVAL命令对lua脚本进行求值。

- 脚本的命令是原子的，RedisServer在执行脚本命令中，不允许插入新的命令
- 脚本的命令可以复制，RedisServer在获得脚本后不执行，生成标识返回，Client根据标识就可以随时执行

## EVAL/EVALSHA命令实现

### EVAL命令

通过执行redis的eval命令，可以运行一段lua脚本。

```Bash
EVAL script numkeys key [key ...] arg [arg ...]
```

命令说明:

- script参数
  >是一段Lua脚本程序，它会被运行在Redis服务器上下文中，这段脚本不必(也不应该)定义为一个Lua函数。
- numkeys参数：
  >用于指定键名参数的个数。
- key [key ...]参数：
  >从EVAL的第三个参数开始算起，使用了numkeys个键(key)，表示在脚本中 所用到的那些Redis键(key)，这些键名参数可以在Lua中通过全局变量KEYS数组，用1为基址的形式访问( KEYS[1] ， KEYS[2] ，以此类推)。
- arg [arg ...]参数：
  >可以在Lua中通过全局变量ARGV数组访问，访问的形式和KEYS变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。

```bash
eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
```

### lua脚本中调用Redis命令

- `redis.call()`：
    - 返回值就是redis命令执行的返回值
    - 如果出错，则返回错误信息，不继续执行
- `redis.pcall()`：
    - 返回值就是redis命令执行的返回值
    - 如果出错，则记录错误信息，继续执行

> 在脚本中，使用return语句将返回值返回给客户端，如果没有return，则返回nil

```bash
eval "return redis.call('set',KEYS[1],ARGV[1])" 1 n1 zhaoyun
```

### EVALSHA

EVAL 命令要求你在每次执行脚本的时候都发送一次脚本主体(script body)。Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本，不过在很多场合，付出无谓的带宽来传送脚本主体并不是最佳选择。

为了减少带宽的消耗， Redis 实现了 EVALSHA 命令，它的作用和 EVAL 一样，都用于对脚本求值，但它接受的第一个参数不是脚本，而是脚本的 SHA1 校验和(sum)

#### SCRIPT命令

- `SCRIPT FLUSH`：清除所有脚本缓存
- `SCRIPT EXISTS`：根据给定的脚本校验和，检查指定的脚本是否存在于脚本缓存
- `SCRIPT LOAD`：将一个脚本装入脚本缓存，返回SHA1摘要，但并不立即运行它

	```bash
	192.168.24.131:6380> script load "return redis.call('set',KEYS[1],ARGV[1])"
	"c686f316aaf1eb01d5a4de1b0b63cd233010e63d"
	192.168.24.131:6380> evalsha c686f316aaf1eb01d5a4de1b0b63cd233010e63d 1 n2
	zhangfei
	OK
	192.168.24.131:6380> get n2
	```

- `SCRIPT KILL`：杀死当前正在运行的脚本

## 脚本管理命令实现

使用redis-cli直接执行lua脚本。

test.lua：

```lua
return redis.call('set',KEYS[1],ARGV[1])
```

```Bash
# ，两边有空格
./redis-cli -h 127.0.0.1 -p 6379 --eval test.lua name:6 , 'caocao' 
```

list.lua：

```lua
local key=KEYS[1]
local list=redis.call("lrange",key,0,-1);
return list;
```

```Bash
./redis-cli --eval list.lua list
```

利用Redis整合Lua，主要是为了性能以及**事务的原子性**。因为redis帮我们提供的事务功能太差。

## 脚本复制

Redis 传播 Lua 脚本，在使用主从模式和开启AOF持久化的前提下：当执行lua脚本时，Redis 服务器有两种模式：**脚本传播模式**和**命令传播模式**。

### 脚本传播模式

脚本传播模式是Redis 复制脚本时默认使用的模式，Redis会将被执行的脚本及其参数复制到 AOF 文件以及从服务器里面。 执行以下命令:

```bash
eval "redis.call('set',KEYS[1],ARGV[1]);redis.call('set',KEYS[2],ARGV[2])" 2 n1
n2
zhaoyun1 zhaoyun2
```

那么主服务器将向从服务器发送完全相同的 eval 命令:

```bash
eval "redis.call('set',KEYS[1],ARGV[1]);redis.call('set',KEYS[2],ARGV[2])" 2 n1
n2
zhaoyun1 zhaoyun2
```

注意：在这一模式下执行的脚本不能有时间、内部状态、随机函数(spop)等。执行相同的脚本以及参数必须产生相同的效果。在Redis5，也是处于同一个事务中。

### 命令传播模式

处于命令传播模式的主服务器会将执行脚本产生的所有写命令用事务包裹起来，然后将事务复制到 AOF 文件以及从服务器里面。

因为命令传播模式复制的是写命令而不是脚本本身，所以即使脚本本身包含时间、内部状态、随机函数等，主服务器给所有从服务器复制的写命令仍然是相同的。

为了开启命令传播模式，用户在使用脚本执行任何写操作之前，需要先在脚本里面调用以下函数:

```C
redis.replicate_commands()
```

redis.replicate_commands() 只对调用该函数的脚本有效：在使用命令传播模式执行完当前脚本之后， 服务器将自动切换回默认的脚本传播模式。

如果我们在主服务器执行以下命令:

```c
eval "redis.replicate_commands();redis.call('set',KEYS[1],ARGV[1]);redis.call('set',KEYS[2],ARGV[2])" 2 n1 n2 zhaoyun11 zhaoyun22
```

那么主服务器将向从服务器复制以下命令:

```bash
EXEC
*1
$5
MULTI
*3
$3
set
$2
n1
$9
zhaoyun11
*3
$3
set
$2
n2
$9
zhaoyun22
*1
$4
EXEC
```

**管道(pipeline),事务和脚本(lua)三者的区别**

- 三者都可以批量执行命令
- 管道无原子性，命令都是独立的，属于无状态的操作
- 事务和脚本是有原子性的，其区别在于脚本可借助Lua语言可在服务器端存储的便利性定制和简化操作
- 脚本的原子性要强于事务，脚本执行期间，另外的客户端其它任何脚本或者命令都无法执行，脚本的执行时间应该尽量短，不能太耗时的脚本


# [Redisson](https://redisson.org/)

[Redisson](https://redisson.org/)是架设在[Redis](http://redis.cn/)基础上的一个Java驻内存数据网格（In-Memory Data Grid），基于NIO的Netty框架，简化了分布式环境中程序相互之间的协作。

[https://github.com/redisson/redisson/wiki/目录](https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95)

## 使用

- 加入jar包的依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>2.7.0</version>
</dependency>
```

- 配置Redisson

```java
public class RedissonManager {
    private static Config config = new Config(); //声明redisso对象
    private static Redisson redisson = null;
    //实例化redisson 
    static{
        config.useClusterServers() 
        // 集群状态扫描间隔时间，单位是毫秒
        .setScanInterval(2000) 
        //cluster方式至少6个节点(3主3从，3主做sharding，3从用来保证主宕机后可以高可用) 
        .addNodeAddress("redis://127.0.0.1:6379" ) 
        .addNodeAddress("redis://127.0.0.1:6380") 
        .addNodeAddress("redis://127.0.0.1:6381") 
        .addNodeAddress("redis://127.0.0.1:6382") 
        .addNodeAddress("redis://127.0.0.1:6383") 
        .addNodeAddress("redis://127.0.0.1:6384");
        //得到redisson对象
        redisson = (Redisson) Redisson.create(config);
    }
    
    //获取redisson对象的方法
    public static Redisson getRedisson(){
        return redisson;
    }
}

```

- 锁的获取和释放

```java
public class DistributedRedisLock { 
    //从配置类中获取redisson对象
    private static Redisson redisson = RedissonManager.getRedisson();
    private static final String LOCK_TITLE = "redisLock_"; 
    //加锁
    public static boolean acquire(String lockName){ 
        //声明key对象
        String key = LOCK_TITLE + lockName; 
        //获取锁对象
        RLock mylock = redisson.getLock(key); 
        //加锁，并且设置锁过期时间3秒，防止死锁的产生 uuid+threadId
        mylock.lock(2,3,TimeUtil.SECOND);
        //加锁成功
        return  true;
    }
    
    //锁的释放
    public static void release(String lockName){
        //必须是和加锁时的同一个key
        String key = LOCK_TITLE + lockName;
        //获取所对象
        RLock mylock = redisson.getLock(key);
        //释放锁(解锁) 
        mylock.unlock();
    } 
}
```

- 业务逻辑中使用分布式锁

```java
public String discount() throws IOException{
    String key = "lock001";
    //加锁 
    DistributedRedisLock.acquire(key); 
    //执行具体业务逻辑 
    dosoming();
    //释放锁 
    DistributedRedisLock.release(key);
    //返回结果 
    return soming;
}
```

## Redisson分布式锁的实现原理

![](Redisson.jpg)

## 加锁机制

如果该客户端面对的是一个redis cluster集群，他首先会根据hash节点选择一台机器。 发送lua脚本到redis服务器上，脚本如下:

```lua
--看有没有锁
if (redis.call('exists',KEYS[1])==0) 
then 
  --无锁 加锁
  redis.call('hset',KEYS[1],ARGV[2],1) ;  
  redis.call('pexpire',KEYS[1],ARGV[1]) ; 
  return nil; 
end ;
--我加的锁
if (redis.call('hexists',KEYS[1],ARGV[2]) ==1 ) 
then 
  --重入锁 
  redis.call('hincrby',KEYS[1],ARGV[2],1) ; 
  redis.call('pexpire',KEYS[1],ARGV[1]) ; 
  return nil; 
end ;
--不能加锁，返回锁的时间
return redis.call('pttl',KEYS[1]) ;


```

lua的作用：保证这段复杂业务逻辑执行的原子性。

lua的解释：

- KEYS[1]) : 加锁的key
- ARGV[1] : key的生存时间，默认为30秒
- ARGV[2] : 加锁的客户端ID (UUID.randomUUID()

第一段if判断语句，就是用“exists myLock”命令判断一下，如果你要加锁的那个锁key不存在的话，你就进行加锁。如何加锁呢?很简单，用下面的命令:

```bash
hset myLock 8743c9c0-0795-4907-87fd-6c719a6b4586:1 1
```

通过这个命令设置一个hash数据结构，这行命令执行后，会出现一个类似下面的数据结构：`myLock:{"8743c9c0-0795-4907-87fd-6c719a6b4586:1":1}`

上述就代表“8743c9c0-0795-4907-87fd-6c719a6b4586:1”这个客户端对“myLock”这个锁key完成了加锁。

接着会执行“`pexpire myLock 30000`”命令，设置myLock这个锁key的生存时间是30秒。

## 锁互斥机制

那么在这个时候，如果客户端2来尝试加锁，执行了同样的一段lua脚本，会咋样呢?

很简单，第一个if判断会执行“exists myLock”，发现myLock这个锁key已经存在了。

接着第二个if判断，判断一下，myLock锁key的hash数据结构中，是否包含客户端2的ID，但是明显不是的，因为那里包含的是客户端1的ID。

所以，客户端2会获取到`pttl myLock`返回的一个数字，这个数字代表了myLock这个锁key的剩余生存时间。比如还剩15000毫秒的生存时间。

此时客户端2会进入一个while循环，不停的尝试加锁。

## 自动延时机制

只要客户端1一旦加锁成功，就会启动一个watch dog看门狗，他是一个后台线程，会每隔10秒检查一 下，如果客户端1还持有锁key，那么就会不断的延长锁key的生存时间。

## 可重入锁机制

第一个if判断肯定不成立，“exists myLock”会显示锁key已经存在了。 第二个if判断会成立，因为myLock的hash数据结构中包含的那个ID，就是客户端1的那个ID，也就是“8743c9c0-0795-4907-87fd-6c719a6b4586:1”

此时就会执行可重入加锁的逻辑，他会用:

```bash
incrby myLock 8743c9c0-0795-4907-87fd-6c71a6b4586:1 1
```

通过这个命令，对客户端1的加锁次数，累加1。数据结构会变成: `myLock:{"8743c9c0-0795-4907-87fd-6c719a6b4586:1":2}` 释放锁机制。

执行lua脚本如下:

```lua
-- 如果key已经不存在，说明已经被解锁，直接发布(publish)redis消息 
if (redis.call('exists', KEYS[1]) == 0)  then 
  redis.call('publish', KEYS[2], ARGV[1]); 
  return 1; 
end;
-- key和field不匹配，说明当前客户端线程没有持有锁，不能主动解锁。 不是我加的锁 不能解锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then 
  -- 将value减1
  return nil;
end; 
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 
-- 如果counter>0说明锁在重入，不能删除key
if (counter > 0) then 
  redis.call('pexpire', KEYS[1], ARGV[2]); 
  return 0; 
else 
  -- 删除key并且publish 解锁消息 
  redis.call('del', KEYS[1]); 
  -- 删除锁 
  redis.call('publish', KEYS[2], ARGV[1]); 
  return 1; 
end; 
return nil;

```

- KEYS[1]：需要加锁的key，这里需要是字符串类型。
- KEYS[2]：redis消息的ChannelName,一个分布式锁对应唯一的一个channelName:“redisson_lockchannel{” + getName() + “}”
- ARGV[1] :reids消息体，这里只需要一个字节的标记就可以，主要标记redis的key已经解锁，再结合redis的Subscribe，能唤醒其他订阅解锁消息的客户端线程申请锁。
- ARGV[2] :锁的超时时间，防止死锁
- ARGV[3] :锁的唯一标识，也就是刚才介绍的 id(UUID.randomUUID()) + “:” + threadId

如果执行`lock.unlock()`，就可以释放分布式锁，此时的业务逻辑也是非常简单的。 其实说白了，就是每次都对myLock数据结构中的那个加锁次数减1。 如果发现加锁次数是0了，说明这个客户端已经不再持有锁了，此时就会用: `del myLock`命令，从redis里删除这个key。 然后呢，另外的客户端2就可以尝试完成加锁了。


# Redis使用规范

本文主要介绍在使用阿里云Redis的开发规范，从下面几个方面进行说明。

- 键值设计
- 命令使用
- 客户端使用

## 键值设计

### key名设计

- 可读性和可管理性 
  > 以业务名(或数据库名)为前缀(防止key冲突)，用冒号分隔，比如业务名:表名:id
```redis key
ugc:video:1
```
	
- 简洁性
	>保证语义的前提下，控制key的长度，当key较多时，内存占用也不容忽视
- 不要包含特殊字符。
  > 反例:包含空格、换行、单双引号以及其他转义字符
	
### value设计
* 拒绝bigkey 
>	防止网卡流量、慢查询
* string类型控制在10KB以内，hash、list、set、zset元素个数不要超过5000。
> 	大key操作会导致Redis阻塞，数据倾斜以及主从同步延迟问题 
* 控制key的生命周期
>	redis不是垃圾桶，建议使用expire设置过期时间(条件允许可以打散过期时间，防止集中过期)，不过期的数据重点关注idletime。

## 命令使用

1. O(N)命令关注N的数量
    
    例如`hgetall`、`lrange`、`smembers`、`zrange`、`sinter`等并非不能使用，但是需要明确N的值。有遍历需求可以使用`hscan`、`sscan`、`zscan`代替。
    
2. 禁用命令
    
    禁止线上使用`keys`、`flushall`、`flushdb`等，通过redis的`rename`机制禁掉命令，或者使用`scan`的方式渐进式处理。
    
3. 合理使用select
    
    redis的多数据库较弱，使用数字进行区分，很多客户端支持较差，同时多业务用多数据库实际还是单线程处理，会有干扰。
    
4. 使用批量操作提高效率
    
    - 原生命令：例如`mget`、`mset`。
    - 非原生命令：可以使用`pipeline`提高效率。 但要注意控制一次批量操作的元素个数(例如500以内，实际也和元素字节数有关)。
    - 注意两者不同:
        * 原生是原子操作，pipeline是非原子操作。
        * pipeline可以打包不同的命令，原生做不到
        * pipeline需要客户端和服务端同时支持。
5. 不建议过多使用Redis事务功能
    
    Redis的事务功能较弱(不支持回滚)，而且集群版本(自研和官方)要求一次事务操作的key必须在一个slot 上(可以使用hashtag功能解决)
    
6. Redis集群版本在使用Lua上有特殊要求
    
    * 所有key都应该由 KEYS 数组来传递，redis.call/pcall 里面调用的redis命令，key的位置，必须是 KEYS array, 否则直接返回error，"-ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS arrayrn"
    * 所有key，必须在1个slot上，否则直接返回error, "-ERR eval/evalsha command keys must in same slotrn"
      
7. monitor命令 必要情况下使用monitor命令时，要注意不要长时间使用。
    

## 客户端使用

1. 避免多个应用使用一个Redis实例（共用Redis的问题）
    
    不相干的业务拆分，公共数据做服务化。
    
2. 使用连接池
    
    可以有效控制连接，同时提高效率，标准使用方式:
    ```java
    Jedis jedis = null;
    try {
        jedis = jedisPool.getResource();
        // 具体的命令
        jedis.executeCommand();
    } catch (Exception e) {
    } finally {
        // 注意这里不是关闭连接，在JedisPool模式下，Jedis会被归还给资源池
        if (jedis != null) 
            jedis.close();
    }
    ```
    
3. 熔断功能
    
    高并发下建议客户端添加熔断功能(例如netflix hystrix)
    
4. 合理的加密
    
    设置合理的密码，如有必要可以使用SSL加密访问(阿里云Redis支持)
    
5. 淘汰策略
    
    根据自身业务类型，选好`maxmemory-policy`(最大内存淘汰策略)，设置好过期时间。默认策略是volatile-lru，即超过最大内存后，在过期键中使用lru算法进行key的剔除，保证不过期数据不被删除，但是可能会出现OOM问题。
    
    其他策略如下:
    - allkeys-lru:根据LRU算法删除键，不管数据有没有设置超时属性，直到腾出足够空间为止。
    - allkeys-random:随机删除所有键，直到腾出足够空间为止。
    - volatile-random:随机删除过期键，直到腾出足够空间为止。
    - volatile-ttl:根据键值对象的ttl属性，删除最近将要过期数据。如果没有，回退到noeviction策略。
    - noeviction:不会剔除任何数据，拒绝所有写入操作并返回客户端错误信息"(error) OOM command not allowed when used memory"，此时Redis只响应读操作。

## 相关工具

1. 数据同步
    
    redis间数据同步可以使用：`redis-port`
    
2. big key搜索
    
    redis大key搜索工具
    
3. 热点key寻找 
	
	内部实现使用monitor，所以建议短时间使用facebook的redis-faina 阿里云Redis已经在内核层面解决热点key问题

## 删除bigkey

1.下面操作可以使用pipeline加速。  
2.redis 4.0已经支持key的异步删除，欢迎使用。

1. Hash删除: hscan + hdel
    
    ![](https://secure2.wostatic.cn/static/eiRAr2VWg9z4qYsv5rmC3J/image.png?auth_key=1716829498-hi1A4QmiG9gJjy62DbpwWo-0-86084c74229d95eec70b0751b9b3ef38)
    
2. List删除: ltrim
    
    ![](https://secure2.wostatic.cn/static/tCHT2zA4jBmQacH3FZ1PU/image.png?auth_key=1716829499-ib139F4HW2wJVwEDauxrQ8-0-cd348e01a9e50835319fdb3a62c59e57)
    
3. Set删除: sscan + srem
    
    ![](https://secure2.wostatic.cn/static/df1jMoZB3PbjzRmn6UNqsB/image.png?auth_key=1716829499-9rfRumieVqzgR1Jx3Qaf16-0-35d45b3d68b0d776be6f62076c1a04b3)
    
4. SortedSet删除: zscan + zrem
    
    ![](https://secure2.wostatic.cn/static/8RgwqiPMmP6EnRh6MtaE21/image.png?auth_key=1716829502-wt3hg5iobqKoVbxk9u4Bd4-0-4dd3d1df1b27183dd7735e46c6c675ec)