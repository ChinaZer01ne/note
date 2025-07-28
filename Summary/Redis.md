#flashcards 

# 为什么要使用Redis

- DB缓存，减轻DB服务器压力
    >热点数据先查询Redis，查不到再查DB
- 提高系统响应
    >Redis支持高并发访问。写能到达11万，读请求 8万。
- 做Session共享
  > 登录token存储，可利用`spring-session-data-redis`
- 做分布式锁(Redis)
	>悲观锁：`setnx`
	>乐观锁：`watch` + `incr`
# Redis单线程架构

### 单线程模型

Redis客户端对服务端的每次调用都经历了**发送命令，执行命令，返回结果**三个过程。其中执行命令阶段，Redis是单线程来处理命令的，所有的命令都会进入一个队列中，然后逐个被执行。并且多个客户端发送的命令的执行顺序是不确定的。但是可以确定的是不会有两条命令被同时执行，不会产生并发问题，这就是Redis的单线程基本模型。

### 单线程模型每秒万级别处理能力的原因

1. **纯内存访问**。数据存放在内存中，内存的响应时间大约是100纳秒，这是Redis每秒万亿级别访问的重要基础。
2. **多路复用I/O**，Redis采用select(默认会根据操作系统进行选择O(1)的调用方式，比如epoll/kqueue/eport，如果没有则选择select)作为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的链接，读写关闭都转换为了时间，不在I/O上浪费过多的时间。
3. **单线程**避免了线程切换和竞态产生的消耗。
4. 高效的数据结构。哪些？举例？
	* **SDS（简单动态字符串）**：预分配内存、惰性释放，减少字符串操作开销。
	- **跳表（Skip List）**：有序集合（ZSet）范围查询效率O(log N)，优于平衡树。
	- **压缩列表（Ziplist）**：小规模数据存储紧凑，减少内存访问次数。
	- **Hash与Set**：基于哈希表实现，插入/查找接近O(1)。
5. **网络与协议优化**

> 6.x高版本出现了IO多线程；但执行命令还是在主线程中

# Redis内存淘汰策略

Redis的淘汰策略主要是指当内存达到最大配置时（maxmemory），Redis如何选择哪些数据淘汰以释放内存。
## maxmemory

`maxmemory`配置项用于Redis数据集设置可使用的最大内存量，用户可使用redis.conf文件来设置这个配置项或者使用`config set`命令来直接设置。

1. 直接修改`redis.conf`文件
	```shell
	maxmemory 100mb
	```
2. 直接连接redis执行命令
```shell
	config set maxmemory 100mb
   ```
3. 通过命令查看当前配置的最大内存量
	 ```shell
	config get maxmemory
	 ```

## key的内存淘汰策略

当内存使用量超过`maxmemory`配置的限制时，Redis可以使用以下策略来淘汰数据：

- noeviction  
	> 只返回错误，不会删除任何key。该策略是Redis的默认淘汰策略，一般不会选用。  
- volatile-ttl  
	> 将设置了过期时间的key中即将过期（剩余存活时间最短）的key删除掉。  
- volatile-random  
	> 在设置了过期时间的key中，随机删除某个key。  
- allkeys-random  
	> 从所有key中随机删除某个key。  
- volatile-lru  
	>基于LRU算法，从设置了过期时间的key中，删除掉最近最少使用的key。  
- allkeys-lru  
	> 基于LRU算法，从所有key中，删除掉最近最少使用的key。该策略是最常使用的策略。  （以访问时间为准）
- volatile-lfu  
	>基于LFU算法，从设置了过期时间的key中，删除掉最不经常使用（使用次数最少）的key。  （以访问次数为准）
- allkeys-lfu  
	>基于LFU算法，从所有key中，删除掉最不经常使用（使用次数最少）的key。

##### LRU

LRU (Least recently used) 最近最少使用，算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。

最常见的实现是使用一个链表保存缓存数据，详细算法实现如下:

1. 新数据插入到链表头部;
2. 每当缓存命中(即缓存数据被访问)，则将数据移到链表头部;
3. 当链表满的时候，将链表尾部的数据丢弃。
4. 在Java中可以使用LinkHashMap(哈希链表)去实现LRU

让我们以用户信息的需求为例，来演示一下LRU算法的基本思路:

1. 假设我们使用哈希链表来缓存用户信息，目前缓存了4个用户，这4个用户是按照时间顺序依次从链表右端插入的。
	
	![](lru演示1.jpg)
	
2. 此时，业务方访问用户5，由于哈希链表中没有用户5的数据，我们从数据库中读取出来，插入到缓存 当中。这时候，链表中最右端是最新访问到的用户5，最左端是最近最少访问的用户1。
	
	![](lru演示2.png)
	
3. 接下来，业务方访问用户2，哈希链表中存在用户2的数据，我们怎么做呢?我们把用户2从它的前驱 节点和后继节点之间移除，重新插入到链表最右端。这时候，链表中最右端变成了最新访问到的用户 2，最左端仍然是最近最少访问的用户1。
	
	![](lru演示3.jpg)
	![](lru演示3.png)
	
4. 接下来，业务方请求修改用户4的信息。同样道理，我们把用户4从原来的位置移动到链表最右侧，并 把用户信息的值更新。这时候，链表中最右端是最新访问到的用户4，最左端仍然是最近最少访问的用 户1。
	
	![](lru演示4.jpg)
	
	![](lru演示4.png)
	
5. 业务访问用户6，用户6在缓存里没有，需要插入到哈希链表。假设这时候缓存容量已经达到上限，必须先删除最近最少访问的数据，那么位于哈希链表最左端的用户1就会被删除掉，然后再把用户6插入到 最右端。
	
	![](lru演示5.png)
	![](lru演示6.png)

> 编者注：一个队列，缓存的时候往队列中加，如果缓存中的元素被访问，将其放到队列尾部。

**Redis的LRU 数据淘汰机制**

在服务器配置中保存了 lru 计数器 `server.lrulock`，会定时(redis 定时程序 `serverCorn()`)更新，`server.lrulock` 的值是根据 `server.unixtime` 计算出来的。  另外，从 `struct redisObject` 中可以发现，每一个 redis 对象都会设置相应的 lru。可以想象的是，每一次访问数据的时候，会更新`redisObject.lru`。

LRU 数据淘汰机制是这样的：在数据集中随机挑选几个键值对，取出其中 lru 最大的键值对淘汰。

* `volatile-lru`
	从已设置过期时间的数据集(`server.db[i].expires`)中挑选最近最少使用的数据淘汰

* `allkeys-lru`
	从数据集(`server.db[i].dict`)中挑选最近最少使用的数据淘汰

> 编者注：相较于传统的LRU，Redis会为每个对象计算LRU值，每次访问会更新，淘汰的时候淘汰LRU值最大的。
##### LFU

LFU (Least frequently used) 最不经常使用，如果一个数据在最近一段时间内使用次数很少，那么在将来一段时间内被使用的可能性也很小。

* `volatile-lfu`
* `allkeys-lfu`

> 当存在大量的热点缓存数据的时候，LFU可能更好。

> LRU和LFU区别：
> 一个是按使用时间淘汰，一个是按使用次数淘汰。
##### random

* `volatile-random`
	从已设置过期时间的数据集(`server.db[i].expires`)中任意选择数据淘汰  
* `allkeys-random`
	从数据集(`server.db[i].dict`)中任意选择数据淘汰

##### ttl

`volatile-ttl`

从已设置过期时间的数据集(`server.db[i].expires`)中挑选将要过期的数据淘汰。redis 数据集数据结构中保存了键值对过期时间的表，即 `redisDb.expires`。

TTL 数据淘汰机制：从过期时间的表中随机挑选几个键值对，取出其中 ttl 最小的键值对淘汰。

##### noenviction

禁止驱逐数据，不删除，这是默认的策略
# 缓存过期和删除策略

## expire的使用

expire命令的使用方法如下: `expire key ttl(单位秒)`

```bash
127.0.0.1:6379> expire name 2 #2秒失效 (integer) 1
127.0.0.1:6379> get name
(nil)
127.0.0.1:6379> set name zhangfei
OK
127.0.0.1:6379> ttl name #永久有效 (integer) -1
127.0.0.1:6379> expire name 30 #30秒失效 (integer) 1
127.0.0.1:6379> ttl name #还有24秒失效 (integer) 24
127.0.0.1:6379> ttl name #失效 (integer) -2

```

### expire原理

```C
typedef struct redisDb {
    dict *dict;  -- key Value
    dict *expires; -- key ttl
    dict *blocking_keys;
    dict *ready_keys;
    dict *watched_keys;
    int id;
} redisDb;
```

上面的代码是Redis 中关于数据库的结构体定义，这个结构体定义中除了 id 以外都是指向字典的指针， 其中我们只看 dict 和 expires。

dict 用来维护一个 Redis 数据库中包含的所有 Key-Value 键值对，expires则用于维护一个 Redis 数据库中设置了失效时间的键(即key与失效时间的映射)。

当我们使用 expire命令设置一个key的失效时间时，Redis 首先到 dict 这个字典表中查找要设置的key 是否存在，如果存在就将这个key和失效时间添加到 expires 这个字典表。

当我们使用 setex命令向系统插入数据时，Redis 首先将 Key 和 Value 添加到 dict 这个字典表中，然后将 Key 和失效时间添加到 expires 这个字典表中。

简单地总结来说就是，设置了失效时间的key和具体的失效时间全部都维护在 expires 这个字典表中。

> 注：Redis维护两个字典，一个保存 key - value的映射，另一个保存 key - ttl 的映射。
## 删除策略

Redis的数据删除有**定时删除**、**惰性删除**和**主动删除**三种方式。 Redis目前采用惰性删除+主动删除的方式。

### 定时删除(expire)
网上说的这种方式可能就是expire。

### 惰性删除

在key被访问时如果发现它已经失效，那么就删除它。

调用`expireIfNeeded`函数，该函数的意义是：读取数据之前先检查一下它有没有失效，如果失效了就删除它。

```c
int expireIfNeeded(redisDb *db, robj *key) {
    //获取主键的失效时间 get当前时间-创建时间>ttl
    long long when = getExpire(db,key); //假如失效时间为负数，说明该主键未设置失效时间(失效时间默认为-1)，直接返回0 
    if (when < 0) return 0; //假如Redis服务器正在从RDB文件中加载数据，暂时不进行失效主键的删除，直接返回0 
    if (server.loading) return 0;
    ...
    //如果以上条件都不满足，就将主键的失效时间与当前时间进行对比，如果发现指定的主键
    //还未失效就直接返回0
    if (mstime() <= when) return 0; //如果发现主键确实已经失效了，那么首先更新关于失效主键的统计个数，然后将该主键失 //效的信息进行广播，最后将该主键从数据库中删除
    server.stat_expiredkeys++;
    propagateExpire(db,key);
    return dbDelete(db,key);
}

```

### 主动删除

在redis.conf文件中可以配置主动删除策略，默认是`no-enviction`(不删除)

```config
maxmemory-policy allkeys-lru
```

## 复制功能对过期键的处理  

- 执行 SAVE 命令或者 BGSAVE 命令所产生的新 RDB 文件不会包含已经过期的键。  
- 执行 BGREWRITEAOF 命令所产生的重写 AOF 文件不会包含已经过期的键。  
- 当一个过期键被删除之后， 服务器会追加一条 DEL 命令到现有 AOF 文件的末尾， 显式地删除过期键。 
- 当主服务器删除一个过期键之后， 它会向所有从服务器发送一条 DEL 命令， 显式地删除过期键。  
- 从服务器即使发现过期键， 也不会自作主张地删除它， 而是等待主节点发来 DEL 命令， 这种统一、中心化的过期键删除策略可以保证主从服务器数据的一致性。  
> 编者注：主从复制不包含过期key，主节点key过期，会追加删除命令到从节点。从节点不会自己删除key而是等到主节点的删除命令。


# 持久化
* RDB：状态机复制
* AOF：状态转移

[Redis持久化方式](http://doc.redisfans.com/topic/persistence.html)
## Redis的持久化方式

- RDB方式(Redis DataBase)：定期备份快照，常用于灾难恢复。
    - 优点：通过fork出的进程进行备份，不影响主进程、RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快，文件也小。
    - 缺点：会丢数据，无法保存最近一次快照之后的数据。
- AOF方式(Append Only File)：保存操作日志方式。
    - 优点：恢复时数据丢失少
    - 缺点：文件大，恢复慢。
- 也可以两者结合使用。

## RDB（快照）持久化

### 触发RDB持久化方式

- 符合自定义配置的快照规则
- 执行save或者`bgsave`命令
- 执行`flushall`命令
- 执行主从复制操作 (第一次)

#### 命令式触发

```bash
127.0.0.1:6379> bgsave
Background saving started
```

```PowerShell
# 通过lastsave查询上次快照时间
127.0.0.1:6379> LASTSAVE
(integer) 1560912871
```

可以通过`SAVE`和`BGSAVE`生成RDB快照文件

- SAVE：阻塞Redis的服务器进程，直到RDB文件被创建完毕
- **BGSAVE：Fork（Copy-on-Write）出一个子进程来创建RDB文件，不阻塞服务器进程**

> Copy-on-Write：如果有多个调用者同时要求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，知道某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本给该调用者，而其他调用者所见到的最初的资源（之前的快照）仍保持不变

#### 配置参数定期自动化触发

根据`redis.conf`配置中的`save m n`定时触发（用的是BGSAVE）

```Bash
save "" # 不使用RDB存储 不能主从
save 900 1 # 表示15分钟(900秒钟)内至少1个键被更改则进行快照。 
save 300 10 # 表示5分钟(300秒)内至少10个键被更改则进行快照。 
save 60 10000 # 表示1分钟内至少10000个键被更改则进行快照。
```

- 主从复制时，主节点自动触发
- 执行Debug Reload
- 执行Shutdown且没有开启AOF持久化

### RDB持久化原理

![](RDB持久化.jpg)

1. Redis父进程首先判断：当前是否在执行save或bgsave/bgrewriteaof(aof文件重写命令)的子进程，如果在执行则bgsave命令直接返回。
2. 父进程执行fork(调用OS函数复制主进程)操作创建子进程，这个复制过程中父进程是阻塞的， Redis不能执行来自客户端的任何命令。
3. 父进程fork后，bgsave命令返回”Background saving started”信息并不再阻塞父进程，并可以响应其他命令。
4. 子进程创建RDB文件，根据父进程内存快照生成临时快照文件，完成后对原有文件进行原子替换。 (RDB始终完整)
5. 子进程发送信号给父进程表示完成，父进程更新统计信息。
6. 父进程fork子进程后，继续工作。
> 通过fork子进程的方式生成新的RDB文件，替换旧的文件。fork系统调用是阻塞的，fork完成后不会再阻塞主进程。子进程执行完替换RDB文件。

### RDB文件结构

[RDB文件结构](Redis-extension.md#RDB文件结构)

### RDB优缺点

* 优点
	- RDB是二进制压缩文件，占用空间小，便于传输(传给slave)
	- 主进程fork子进程，可以最大化Redis性能

* 缺点
	- 内存数据的全量同步，数据量大会由于I/O而严重影响性能
	- 可能会因为Redis挂掉而丢失从当前至最近一次快照期间的数据

## AOF持久化

AOF（Append-Only-File）持久化：通过保存redis的写状态来持久化数据库的。记录下除了查询以外的所有变更数据库状态的指令以append的形式追加保存到AOF文件中（增量）

### AOF持久化实现

配置 `redis.conf`

```bash
# 可以通过修改redis.conf配置文件中的appendonly参数开启 
appendonly yes
# AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的。 
dir ./
# 默认的文件名是appendonly.aof，可以通过appendfilename参数修改 
appendfilename appendonly.aof
```

### AOF原理

AOF文件中存储的是redis的命令，同步命令到 AOF 文件的整个过程可以分为三个阶段:

- **命令传播**：
  >Redis 将执行完的命令、命令的参数、命令的参数个数等信息发送到 AOF 程序中。
- **缓存追加**：
  >AOF 程序根据接收到的命令数据，将命令转换为网络通讯协议的格式，然后将协议内容追加到服务器的 AOF 缓存中。
- **文件写入和保存**：
  >AOF 缓存中的内容被写入到 AOF 文件末尾，如果设定的 AOF 保存条件被满足的话， fsync 函数或者 fdatasync 函数会被调用，将写入的内容真正地保存到磁盘中。

#### 命令传播

当一个 Redis 客户端需要执行命令时， 它通过网络连接， 将协议文本发送给 Redis 服务器。服务器在接到客户端的请求之后， 它会根据协议文本的内容， 选择适当的命令函数， 并将各个参数从字符串文本转换为 Redis 字符串对象(StringObject)。每当命令函数成功执行之后， 命令参数都会被传播到 AOF 程序。

#### 缓存追加

当命令被传播到 AOF 程序之后， 程序会根据命令以及命令的参数， 将命令从字符串对象转换回原来的协议文本(RESP)。协议文本生成之后， 它会被追加到 `redis.h/redisServer` 结构的 `aof_buf` 末尾。

redisServer 结构维持着 Redis 服务器的状态， `aof_buf` 域则保存着所有等待写入到 AOF 文件的协议文本(RESP)。

#### 文件写入和保存

每当服务器常规任务函数被执行、 或者事件处理器被执行时， `aof.c/flushAppendOnlyFile` 函数都会被调用， 这个函数执行以下两个工作：
- WRITE：根据条件，将 `aof_buf` 中的缓存写入到 AOF 文件。
- SAVE：根据条件，调用 `fsync` 或 `fdatasync` 函数，将 AOF 文件保存到磁盘中。

### AOF 保存模式

Redis 目前支持三种 AOF 保存模式，它们分别是:
* `AOF_FSYNC_NO`
	* 不保存
* `AOF_FSYNC_EVERYSEC`
	* 每一秒钟保存一次。(默认)
* `AOF_FSYNC_ALWAYS`
	* 每执行一个命令保存一次。(不推荐) 

#### AOF_FSYNC_NO

在这种模式下， 每次调用 `flushAppendOnlyFile` 函数， WRITE 都会被执行， 但 SAVE 会被略过。 SAVE 只会在以下任意一种情况中被执行:
- Redis 被关闭
- AOF 功能被关闭
- 系统的写缓存被刷新(可能是缓存已经被写满，或者定期保存操作被执行)

这三种情况下的 SAVE 操作都会引起 Redis 主进程阻塞。

#### AOF_FSYNC_EVERYSEC

在这种模式中， SAVE 原则上每隔一秒钟就会执行一次， 因为 SAVE 操作是由后台子线程(fork)调用的， 所以它不会引起服务器主进程阻塞。

> Save不阻塞
#### AOF_FSYNC_ALWAYS
    
在这种模式下，每次执行完一个命令之后， WRITE 和 SAVE 都会被执行。另外，因为 SAVE 是由 Redis 主进程执行的，所以在 SAVE 执行期间，主进程会被阻塞，不能接受命令请求。

对于三种 AOF 保存模式， 它们对服务器主进程的阻塞情况如下:

| 模式                   | WRITE是否阻塞 | SAVE是否阻塞 | 停机时丢失的数据量                   |
| -------------------- | --------- | -------- | --------------------------- |
| `AOF_FSYNC_NO`       | 阻塞        | 阻塞       | 操作系统最后一次对AOF文件触发SAVE操作之后的数据 |
| `AOF_FSYNC_EVERYSEC` | 阻塞        | 不阻塞      | 一般情况下不超过2秒的数据               |
| `AOF_FSYNC_ALWAYS`   | 阻塞        | 阻塞       | 最多值丢失一个命令的数据                |
### AOF重写

> fork子进程，不阻塞主进程；
> 重写过程中的增量命令，通过重写缓存保存，最后合并

Redis可以在 AOF体积变得过大时，自动地在后台(Fork子进程)对 AOF进行重写。重写后的新AOF文件包含了恢复当前数据集所需的最小命令集合。 所谓的“重写”其实是一个有歧义的词语， 实际上， AOF 重写并不需要对原有的 AOF 文件进行任何写入和读取， **它针对的是数据库中键的当前值**。

Redis 不希望 AOF 重写造成服务器无法处理请求， 所以 Redis 决定将 AOF 重写程序放到(后台)子进程里执行， 这样处理的最大好处是:

1. 子进程进行 AOF 重写期间，主进程可以继续处理命令请求。
2. 子进程带有主进程的数据副本，使用子进程而不是线程，可以在避免锁的情况下，保证数据的安全性。

不过， 使用子进程也有一个问题需要解决：因为子进程在进行 AOF 重写期间， 主进程还需要继续处理命令， 而新的命令可能对现有的数据进行修改， 这会让当前数据库的数据和重写后的 AOF 文件中的数据不一致。

为了解决这个问题， Redis 增加了一个 AOF 重写缓存， 这个缓存在 fork 出子进程之后开始启用， Redis 主进程在接到新的写命令之后， 除了会将这个写命令的协议内容追加到现有的 AOF 文件之外， 还会追加到这个缓存中。

![](AOF重写.png)

**重写过程分析(整个重写操作是绝对安全的)：**

Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。

当子进程在执行 AOF 重写时， 主进程需要执行以下三个工作:

- 处理命令请求。
- 将写命令追加到现有的 AOF 文件中。
- 将写命令追加到 AOF 重写缓存中。

这样一来可以保证:

现有的 AOF 功能会继续执行，即使在 AOF 重写期间发生停机，也不会有任何数据丢失。 所有对数据库进行修改的命令都会被记录到 AOF 重写缓存中。 当子进程完成 AOF 重写之后， 它会向父进程发送一 个完成信号， 父进程在接到完成信号之后， 会调用一个信号处理函数， 并完成以下工作：

- 将 AOF 重写缓存中的内容全部写入到新 AOF 文件中。
    >现有 AOF 文件、新 AOF 文件和数据库三者的状态就完全一致了。
- 对新的 AOF 文件进行改名，覆盖原有的 AOF 文 件。

Redis数据库里的+AOF重写过程中的命令→新的AOF文件→覆盖老的

这个信号处理函数执行完毕之后， 主进程就可以继续像往常一样接受命令请求了。 在整个 AOF 后台重写过程中， 只有最后的写入缓存和改名操作会造成主进程阻塞， 在其他时候， AOF 后台重写都不会对主进程造成阻塞， 这将 AOF 重写对性能造成的影响降到了最低。

以上就是 AOF 后台重写， 也即是 `BGREWRITEAOF` 命令(AOF重写)的工作原理。

![](AOF重写流程.jpg)

#### 重写触发方式

1. 配置触发 在`redis.conf`中配置

	```bash
	# 表示当前aof文件大小超过上一次aof文件大小的百分之多少的时候会进行重写。如果之前没有重写过，以 启动时aof文件大小为准
	auto-aof-rewrite-percentage 100
	# 限制允许重写最小aof文件大小，也就是文件大小小于64mb的时候，不需要进行优化 
	auto-aof-rewrite-min-size 64mb
	```

2. 执行`bgrewriteaof`命令

	```bash
	127.0.0.1:6379> bgrewriteaof
	Background append only file rewriting started
	```

### 混合持久化

RDB和AOF各有优缺点，Redis 4.0 开始支持 rdb 和 aof 的混合持久化。如果把混合持久化打开，aof rewrite 的时候就直接把 rdb 的内容写到 aof 文件开头。

RDB的头+AOF的身体→appendonly.aof 开启混合持久化

```bash
aof-use-rdb-preamble yes
```

![](混合持久化文件.png)

我们可以看到该AOF文件是rdb文件的头和aof格式的内容，在加载时，首先会识别AOF文件是否以 REDIS字符串开头，如果是就按RDB格式加载，加载完RDB后继续按AOF格式加载剩余部分。

## Redis数据恢复

**RDB和AOF文件共存情况下的恢复流程：**

由于`appendonly.aof`文件保存的数据集要比`dump.rdb`文件保存的数据集更完整，当Redis重启时，会优先载入AOF文件来恢复原始数据。

**RDB-AOF混合持久化方式：**

- `BGSAVE`做镜像全量持久化，AOF做增量持久化
- AOF文件前半段是RDB格式的全量数据，后半段是Redis命令格式的增量数据。

### AOF文件的数据恢复

因为AOF文件里面包含了重建数据库状态所需的所有写命令，所以服务器只要读入并重新执行一遍AOF 文件里面保存的写命令，就可以还原服务器关闭之前的数据库状态。

Redis读取AOF文件并还原数据库状 态的详细步骤如下:

1. 创建一个不带网络连接的伪客户端(fake client)：因为Redis的命令只能在客户端上下文中执行，而载入AOF文件时所使用的命令直接来源于AOF文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行AOF文件保存的写命令，伪客户端执行命令 的效果和带网络连接的客户端执行命令的效果完全一样
2. 从AOF文件中分析并读取出一条写命令
3. 使用伪客户端执行被读出的写命令
4. 一直执行步骤2和步骤3，直到AOF文件中的所有写命令都被处理完毕为止

当完成以上步骤之后，AOF文件所保存的数据库状态就会被完整地还原出来，整个过程如下图所示:

![](aof恢复.png)

## RDB与AOF对比

1. RDB保存某个时刻的数据快照，采用二进制压缩存储，AOF保存操作命令，采用文本存储(混合)
2. RDB性能高、AOF性能较低
3. RDB在配置触发状态会丢失最后一次快照以后更改的所有数据，AOF设置为每秒保存一次，则最多丢2秒的数据
4. Redis以主服务器模式运行，RDB不会保存过期键值对数据，Redis以从服务器模式运行，RDB会保存过期键值对，当主服务器向从服务器同步时，再清空过期键值对。

AOF写入文件时，对过期的key会追加一条del命令，当执行AOF重写时，会忽略过期key和del命令。

> 编者注：RDB是内存快照保存，AOF是状态转移。
## 应用场景

- 内存数据库
  >RDB + AOF，数据不容易丢
- 有原始数据源
  >每次启动时都从原始数据源中初始化 ，则不用开启持久化 (数据量较小)
- 缓存服务器
  >一般RDB，性能高

## 线上Redis持久化策略一般如何设置？

如果对性能要求较高，在Master最好不要做持久化，可以在某个Slave开启AOF备份数据，策略设置为每秒同步一次即可。

# 事务  

所谓事务(Transaction) ，是指作为单个逻辑工作单元执行的一系列操作
## 命令  
- watch  
- multi  
- execute  
## acid特性  

* 原子性：但即使错误不会回滚，事务没有原子性  
* 持久性：RDB和AOF
* 隔离性：单线程
* 一致性：错误会继续执行完，没有一致性
## Redis事务

- Redis的事务是通过multi、exec、discard和watch这四个命令来完成的。
- Redis的单个命令都是原子性的，所以这里需要确保事务性的对象是命令集合。
- Redis将命令集合序列化并确保处于同一事务的命令集合连续且不被打断的执行
- Redis不支持回滚操作
## 事务命令

- `multi`：用于标记事务块的开始,Redis会将后续的命令逐个放入队列中，然后使用exec原子化地执行这个命令队列
- `exec`：执行命令队列
- `discard`：清除命令队列
- `watch`：监视key
- `unwatch`：清除监视key

![](https://secure2.wostatic.cn/static/jPwGSZE9guoqnsfieXnchz/image.png?auth_key=1716829395-aKG5Th4tNKHRwqcVQvSs2s-0-aa24c231033690492f637c9d30738b70)

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set s1 222
QUEUED
127.0.0.1:6379> hset set1 name zhangfei
QUEUED
127.0.0.1:6379> exec
1) OK
2) (integer) 1

127.0.0.1:6379> multi
OK
127.0.0.1:6379> set s2 333
QUEUED
127.0.0.1:6379> hset set2 age 23
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> exec
(error) ERR EXEC without MULTI

127.0.0.1:6379> watch s1
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set s1 555 
QUEUED
127.0.0.1:6379> exec # 此时在没有exec之前，通过另一个命令窗口对监控的s1字段进行修改
(nil)
127.0.0.1:6379> get s1
222
127.0.0.1:6379> unwatch
OK
```

## 事务机制

### 事务的执行

1. 事务开始
    
    在RedisClient中，有属性flags，用来表示是否在事务中，flags=`REDIS_MULTI`
    
2. 命令入队
    
    RedisClient将命令存放在事务队列中 (EXEC,DISCARD,WATCH,MULTI除外)
    
3. 事务队列
    
    `multiCmd *commands` 用于存放命令
    
4. 执行事务
    
    RedisClient向服务器端发送exec命令，RedisServer会遍历事务队列，执行队列中的命令，最后将执行的结果一次性返回给客户端。
    
5. 如果某条命令在入队过程中发生错误，redisClient将flags置为`REDIS_DIRTY_EXEC`，EXEC命令将会失败返回。
    

![](https://secure2.wostatic.cn/static/6KqYvCvA48p1VnqyTVfxMy/image.png?auth_key=1716829395-djmAzJxGuiP1duJpegxTHB-0-e39c881940b48c772e3b6e85a334c6ae)

```c
typedef struct redisClient{
    // flags
    int flags 
    //状态
    // 事务状态 
    multiState mstate; 
    // .....
}redisClient;

// 事务状态
typedef struct multiState{
    // 事务队列,FIFO顺序
    // 是一个数组,先入队的命令在前,后入队在后 
    multiCmd *commands;
    // 已入队命令数
    int count;
}multiState;

// 事务队列
typedef struct multiCmd{
    // 参数
    robj **argv;
    // 参数数量
    int argc;
    // 命令指针
    struct redisCommand *cmd;
}multiCmd;
```

### Watch的执行

- 使用WATCH命令监视数据库键
    
    redisDb有一个watched_keys字典，key是某个被监视的数据的key，值是一个链表.记录了所有监视这个数据的客户端。
    
- 监视机制的触发
    
    当修改数据后，监视这个数据的客户端的flags置为`REDIS_DIRTY_CAS`
    
- 事务执行
    
    RedisClient向服务器端发送exec命令，服务器判断RedisClient的flags，如果为REDIS_DIRTY_CAS，则清空事务队列。
    

![](https://secure2.wostatic.cn/static/ni8sXTvDtP3HFZPx6L248q/image.png?auth_key=1716829395-848PLwN266ofBZH75XQzEL-0-eea41a8c50ab27ae928b26055ec213d2)

```c
typedef struct redisDb {
    // .....
    // 正在被WATCH命令监视的键 
    dict *watched_keys;
    // .....
}redisDb;
```

### Redis的弱事务性

- Redis语法错误
    
    整个事务的命令在队列里都清除
    

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> sets m1 44
(error) ERR unknown command `sets`, with args beginning with: `m1`, `44`,
127.0.0.1:6379> set m2 55
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get m1
"22"
```

```
flags=`REDIS_DIRTY_EXEC`
```

- Redis运行错误
    
    在队列里正确的命令可以执行 (弱事务性)
    
    弱事务性 :
    
    1、在队列里正确的命令可以执行 (非原子操作)
    
    2、不支持回滚
    

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set m1 55
QUEUED
127.0.0.1:6379> lpush m1 1 2 3 #不能是语法错误 QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> get m1
"55"
```

- Redis不支持事务回滚(为什么呢)
    1. 大多数事务失败是因为语法错误或者类型错误，这两种错误，在开发阶段都是可以预见的
    2. Redis为了性能方面就忽略了事务回滚。 (回滚记录历史版本)


# 服务器  
    
- 命令请求流程  
	- 客户端打印  
	- 返回结果到输出缓冲区  
	- 执行命令处理程序  
	- 输入缓冲区->协议解析->关联命令处理程序  
	- 命令->协议格式->输入缓冲区  
	- 客户端命令请求  
- serverCron函数
# 集群

## **为什么需要Redis集群？**

在讲Redis集群架构之前，我们先简单讲下Redis单实例的架构，从最开始的一主N从，到读写分离，再到Sentinel哨兵机制，单实例的Redis缓存足以应对大多数的使用场景，也能实现主从故障迁移。

![](redis主从.webp)

但是，在某些场景下，单实例存Redis缓存会存在的几个问题：

1. 写并发：
   >Redis单实例读写分离可以解决读操作的负载均衡，但对于写操作，仍然是全部落在了master节点上面，在海量数据高并发场景，一个节点写数据容易出现瓶颈，造成master节点的压力上升。
2. 海量数据的存储压力：
   >单实例Redis本质上只有一台Master作为存储，如果面对海量数据的存储，一台Redis的服务器就应付不过来了，而且数据量太大意味着持久化成本高，严重时可能会阻塞服务器，造成服务请求成功率下降，降低服务的稳定性。

针对以上的问题，Redis集群提供了较为完善的方案，解决了存储能力受到单机限制，写操作无法负载均衡的问题。

## **什么是Redis集群？**

Redis集群采用去中心化的思想，没有中心节点的说法，对于客户端来说，整个集群可以看成一个整体，可以连接任意一个节点进行操作，就像操作单一Redis实例一样，不需要任何代理中间件，当客户端操作的key没有分配到该node上时，Redis会返回转向指令，指向正确的node。

Redis也内置了高可用机制，支持N个master节点，每个master节点都可以挂载多个slave节点，当master节点挂掉时，集群会提升它的某个slave节点作为新的master节点。

![](redis集群.webp)

如上图所示，Redis集群可以看成多个主从架构组合起来的，每一个主从架构可以看成一个节点（其中，只有master节点具有处理请求的能力，slave节点主要是用于节点的高可用）

## 复制  
    
### 完全同步  
	
- 旧：发送Sync命令；新：发送PSYNC命令  
- 发送RDB文件  
- 发送缓冲区增量命令  
- 命令传播  
	- 实时增量命令  

### 增量同步  
- 复制偏移量  
- 双写复制积压缓冲区  
- 断线重连根据偏移量复制积压缓冲区的数据  
	- 连接服务器ID必须相同  
	- 偏移量之后的数据必须在复制积压缓冲区  
- 命令传播  
	- 实时增量命令
## Redis集群的数据分布算法：哈希槽算法

### **什么是哈希槽算法？**

Redis集群通过分布式存储的方式解决了单节点的海量数据存储的问题，对于分布式存储，需要考虑的重点就是如何将数据进行拆分到不同的Redis服务器上。常见的分区算法有hash算法、一致性hash算法，关于这些算法这里就不多介绍。

- 普通hash算法：将key使用hash算法计算之后，按照节点数量来取余，即hash(key)%N。优点就是比较简单，但是扩容或者摘除节点时需要重新根据映射关系计算，会导致数据重新迁移。
- 一致性hash算法：为每一个节点分配一个token，构成一个哈希环；查找时先根据key计算hash值，然后顺时针找到第一个大于等于该哈希值的token节点。优点是在加入和删除节点时只影响相邻的两个节点，缺点是加减节点会造成部分数据无法命中，所以一般用于缓存，而且用于节点量大的情况下，扩容一般增加一倍节点保障数据负载均衡。

Redis集群采用的算法是哈希槽分区算法。Redis集群中有16384个哈希槽（槽的范围是 0 -16383，哈希槽），将不同的哈希槽分布在不同的Redis节点上面进行管理，也就是说每个Redis节点只负责一部分的哈希槽。在对数据进行操作的时候，集群会对使用CRC16算法对key进行计算并对16384取模（slot = **CRC16(key)%16384**），得到的结果就是 Key-Value 所放入的槽，通过这个值，去找到对应的槽所对应的Redis节点，然后直接到这个对应的节点上进行存取操作。

使用哈希槽的好处就在于可以方便的添加或者移除节点，并且无论是添加删除或者修改某一个节点，都不会造成集群不可用的状态。当需要增加节点时，只需要把其他节点的某些哈希槽挪到新节点就可以了；当需要移除节点时，只需要把移除节点上的哈希槽挪到其他节点就行了；哈希槽数据分区算法具有以下几种特点：

- 解耦数据和节点之间的关系，简化了扩容和收缩难度；
- 节点自身维护槽的映射关系，不需要客户端代理服务维护槽分区元数据
- 支持节点、槽、键之间的映射查询，用于数据路由，在线伸缩等场景

> 槽的迁移与指派命令：CLUSTER ADDSLOTS 0 1 2 3 4 ... 5000

默认情况下，**Redis集群的读和写都是到master上去执行的，不支持slave节点读和写**，跟Redis主从复制下读写分离不一样，因为redis集群的核心的理念，主要是使用**Slave做数据的热备，以及master故障时的主备切换，实现高可用的**。Redis的读写分离，是为了横向任意扩展slave节点去支撑更大的读吞吐量。而Redis集群架构下，本身master就是可以任意扩展的，如果想要支撑更大的读或写的吞吐量，都可以直接对master进行横向扩展。

### **Redis中哈希槽相关的数据结构：**

（1）clusterNode数据结构：保存节点的当前状态，比如节点的创建时间，节点的名字，节点当前的配置纪元，节点的IP和地址，等等。

![](clusterNode.webp)

（2）clusterState数据结构：记录当前节点所认为的集群目前所处的状态。

![](clusterState.webp)

（3）节点的槽指派信息：

clusterNode数据结构的slots属性和numslot属性记录了节点负责处理那些槽：

slots属性是一个二进制位数组(bit array)，这个数组的长度为16384/8=2048个字节，共包含16384个二进制位。Master节点用bit来标识对于某个槽自己是否拥有，时间复杂度为O(1)

![](slot示例.jpg)

（4）集群所有槽的指派信息：

当收到集群中其他节点发送的信息时，通过将节点槽的指派信息保存在本地的clusterState.slots数组里面，程序要检查槽i是否已经被指派，又或者取得负责处理槽i的节点，只需要访问clusterState.slots[i]的值即可，时间复杂度仅为O(1)

![](slot数组.jpg)

如上图所示，ClusterState 中保存的 Slots 数组中每个下标对应一个槽，每个槽信息中对应一个 clusterNode 也就是缓存的节点。这些节点会对应一个实际存在的 Redis 缓存服务，包括 IP 和 Port 的信息。Redis Cluster 的通讯机制实际上保证了每个节点都有其他节点和槽数据的对应关系。无论Redis 的客户端访问集群中的哪个节点都可以路由到对应的节点上，因为每个节点都有一份 ClusterState，它记录了所有槽和节点的对应关系。

### **集群的请求重定向：**

Redis集群在客户端层面没有采用代理，并且无论Redis 的客户端访问集群中的哪个节点都可以路由到对应的节点上，下面来看看 Redis 客户端是如何通过路由来调用缓存节点的：

1. MOVED请求：

![](客户端跳转过程.jpg)
![](客户端跳转过程2.webp)
如上图所示，Redis 客户端通过 CRC16(key)%16383 计算出 Slot 的值，发现需要找“缓存节点1”进行数据操作，但是由于缓存数据迁移或者其他原因导致这个对应的 Slot 的数据被迁移到了“缓存节点2”上面。那么这个时候 Redis 客户端就无法从“缓存节点1”中获取数据了。但是由于“缓存节点1”中保存了所有集群中缓存节点的信息，因此它知道这个 Slot 的数据在“缓存节点2”中保存，因此向 Redis 客户端发送了一个 MOVED 的重定向请求。这个请求告诉其应该访问的“缓存节点2”的地址。Redis 客户端拿到这个地址，继续访问“缓存节点2”并且拿到数据。

2. ASK请求：

上面的例子说明了，数据 Slot 从“缓存节点1”已经迁移到“缓存节点2”了，那么客户端可以直接找“缓存节点2”要数据。那么如果两个缓存节点正在做节点的数据迁移，此时客户端请求会如何处理呢？

![](客户端ask请求.webp)

Redis 客户端向“缓存节点1”发出请求，此时“缓存节点1”正向“缓存节点 2”迁移数据，如果没有命中对应的 Slot，它会返回客户端一个 ASK 重定向请求并且告诉“缓存节点2”的地址。客户端向“缓存节点2”发送 Asking 命令，询问需要的数据是否在“缓存节点2”上，“缓存节点2”接到消息以后返回数据是否存在的结果。

#### 频繁重定向造成的网络开销的处理：smart客户端

##### 什么是 smart客户端：

在大部分情况下，可能都会出现一次请求重定向才能找到正确的节点，这个重定向过程显然会增加集群的网络负担和单次请求耗时。所以大部分的客户端都是smart的。所谓 smart客户端，就是指客户端本地维护一份hashslot => node的映射表缓存，大部分情况下，直接走本地缓存就可以找到hashslot => node，不需要通过节点进行moved重定向，

##### JedisCluster的工作原理：

- 在JedisCluster初始化的时候，就会随机选择一个node，初始化hashslot => node映射表，同时为每个节点创建一个JedisPool连接池。
- 每次基于JedisCluster执行操作时，首先会在本地计算key的hashslot，然后在本地映射表找到对应的节点node。
- 如果那个node正好还是持有那个hashslot，那么就ok；如果进行了reshard操作，可能hashslot已经不在那个node上了，就会返回moved。
- 如果JedisCluter API发现对应的节点返回moved，那么利用该节点返回的元数据，更新本地的hashslot => node映射表缓存
- 重复上面几个步骤，直到找到对应的节点，如果重试超过5次，那么就报错JedisClusterMaxRedirectionException

##### HashSlot迁移和ask重定向：

如果hashslot正在迁移，那么会返回ask重定向给客户端。客户端接收到ask重定向之后，会重新定位到目标节点去执行，但是因为ask发生在hashslot迁移过程中，所以JedisCluster API收到ask是不会更新hashslot本地缓存。

虽然ASK与MOVED都是对客户端的重定向控制，但是有本质区别。ASK重定向说明集群正在进行slot数据迁移，客户端无法知道迁移什么时候完成，因此只能是临时性的重定向，客户端不会更新slots缓存。但是MOVED重定向说明键对应的槽已经明确指定到新的节点，客户端需要更新slots缓存。

## Redis集群中节点的通信机制：goosip协议

Redis集群的哈希槽算法解决的是数据的存取问题，不同的哈希槽位于不同的节点上，而不同的节点维护着一份它所认为的当前集群的状态，同时，Redis集群是去中心化的架构。那么，当集群的状态发生变化时，比如新节点加入、slot迁移、节点宕机、slave提升为新Master等等，我们希望这些变化尽快被其他节点发现，Redis是如何进行处理的呢？也就是说，Redis不同节点之间是如何进行通信进行维护集群的同步状态呢？

在Redis集群中，不同的节点之间采用gossip协议进行通信，节点之间通讯的目的是为了维护节点之间的元数据信息。这些元数据就是每个节点包含哪些数据，是否出现故障，通过gossip协议，达到最终数据的一致性。

> gossip协议，是基于流行病传播方式的节点或者进程之间信息交换的协议。原理就是在不同的节点间不断地通信交换信息，一段时间后，所有的节点就都有了整个集群的完整信息，并且所有节点的状态都会达成一致。每个节点可能知道所有其他节点，也可能仅知道几个邻居节点，但只要这些节可以通过网络连通，最终他们的状态就会是一致的。Gossip协议最大的好处在于，即使集群节点的数量增加，每个节点的负载也不会增加很多，几乎是恒定的。

Redis集群中节点的通信过程如下：

- 集群中每个节点都会单独开一个TCP通道，用于节点间彼此通信。
- 每个节点在固定周期内通过待定的规则选择几个节点发送ping消息
- 接收到ping消息的节点用pong消息作为响应

使用gossip协议的优点在于将元数据的更新分散在不同的节点上面，降低了压力；但是缺点就是元数据的更新有延时，可能导致集群中的一些操作会有一些滞后。另外，由于 gossip 协议对服务器时间的要求较高，时间戳不准确会影响节点判断消息的有效性。而且节点数量增多后的网络开销也会对服务器产生压力，同时结点数太多，意味着达到最终一致性的时间也相对变长，因此官方推荐最大节点数为1000左右。

> redis cluster架构下的每个redis都要开放两个端口号，比如一个是6379，另一个就是加1w的端口号16379。

- 6379端口号就是redis服务器入口。
- 16379端口号是用来进行节点间通信的，也就是 cluster bus 的通信，用来进行故障检测、配置更新、故障转移授权。cluster bus 用的是一种叫gossip 协议的二进制协议

### **gossip协议的常见类型：**

gossip协议常见的消息类型包含： ping、pong、meet、fail等等。

* `meet`：主要用于通知新节点加入到集群中，通过「cluster meet ip port」命令，已有集群的节点会向新的节点发送邀请，加入现有集群。
* `ping`：用于交换节点的元数据。每个节点每秒会向集群中其他节点发送 ping 消息，消息中封装了自身节点状态还有其他部分节点的状态数据，也包括自身所管理的槽信息等等。
	- 因为发送ping命令时要携带一些元数据，如果很频繁，可能会加重网络负担。因此，一般每个节点每秒会执行 10 次 ping，每次会选择 5 个最久没有通信的其它节点。
	- 如果发现某个节点通信延时达到了 cluster_node_timeout / 2，那么立即发送 ping，避免数据交换延时过长导致信息严重滞后。比如说，两个节点之间都 10 分钟没有交换数据了，那么整个集群处于严重的元数据不一致的情况，就会有问题。所以 cluster_node_timeout 可以调节，如果调得比较大，那么会降低 ping 的频率。
	- 每次 ping，会带上自己节点的信息，还有就是带上 1/10 其它节点的信息，发送出去，进行交换。至少包含 3 个其它节点的信息，最多包含 （总节点数 - 2）个其它节点的信息。
- pong：ping和meet消息的响应，同样包含了自身节点的状态和集群元数据信息。
- fail：某个节点判断另一个节点 fail 之后，向集群所有节点广播该节点挂掉的消息，其他节点收到消息后标记已下线。

由于Redis集群的去中心化以及gossip通信机制，Redis集群中的节点只能保证最终一致性。例如当加入新节点时(meet)，只有邀请节点和被邀请节点知道这件事，其余节点要等待 ping 消息一层一层扩散。除了 Fail 是立即全网通知的，其他诸如新节点、节点重上线、从节点选举成为主节点、槽变化等，都需要等待被通知到，也就是Gossip协议是最终一致性的协议。

### **meet命令的实现：**

![](gossi-meet.webp)

1. 节点A会为节点B创建一个clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面。
2. 节点A根据CLUSTER MEET命令给定的IP地址和端口号，向节点B发送一条MEET消息。
3. 节点B接收到节点A发送的MEET消息，节点B会为节点A创建一个clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面。
4. 节点B向节点A返回一条PONG消息。
5. 节点A将受到节点B返回的PONG消息，通过这条PONG消息，节点A可以知道节点B已经成功的接收了自己发送的MEET消息。
6. 之后，节点A将向节点B返回一条PING消息。
7. 节点B将接收到的节点A返回的PING消息，通过这条PING消息节点B可以知道节点A已经成功的接收到了自己返回的PONG消息，握手完成。
8. 之后，节点A会将节点B的信息通过Gossip协议传播给集群中的其他节点，让其他节点也与节点B进行握手，最终，经过一段时间后，节点B会被集群中的所有节点认识。

## 集群的扩容与收缩：

作为分布式部署的缓存节点总会遇到缓存扩容和缓存故障的问题。这就会导致缓存节点的上线和下线的问题。由于每个节点中保存着槽数据，因此当缓存节点数出现变动时，这些槽数据会根据对应的虚拟槽算法被迁移到其他的缓存节点上。所以对于redis集群，集群伸缩主要在于槽和数据在节点之间移动。

### **扩容：**

- （1）启动新节点
- （2）使用cluster meet命令将新节点加入到集群
- （3）迁移槽和数据：添加新节点后，需要将一些槽和数据从旧节点迁移到新节点

![](集群扩容.webp)

如上图所示，集群中本来存在“缓存节点1”和“缓存节点2”，此时“缓存节点3”上线了并且加入到集群中。此时根据虚拟槽的算法，“缓存节点1”和“缓存节点2”中对应槽的数据会应该新节点的加入被迁移到“缓存节点3”上面。

新节点加入到集群的时候，作为孤儿节点是没有和其他节点进行通讯的。因此需要在集群中任意节点执行 cluster meet 命令让新节点加入进来。假设新节点是 192.168.1.1 5002，老节点是 192.168.1.1 5003，那么运行以下命令将新节点加入到集群中。

> cluster meet 192.168.1.1 5002

这个是由老节点发起的，有点老成员欢迎新成员加入的意思。新节点刚刚建立没有建立槽对应的数据，也就是说没有缓存任何数据。如果这个节点是主节点，需要对其进行槽数据的扩容；如果这个节点是从节点，就需要同步主节点上的数据。总之就是要同步数据。

![](集群扩容数据同步.webp)

如上图所示，由客户端发起节点之间的槽数据迁移，数据从源节点往目标节点迁移。

- （1）客户端对目标节点发起准备导入槽数据的命令，让目标节点准备好导入槽数据。这里使用 cluster setslot {slot} importing {sourceNodeId} 命令。
- （2）之后对源节点发起送命令，让源节点准备迁出对应的槽数据。使用命令 cluster setslot {slot} importing {sourceNodeId}。
- （3）此时源节点准备迁移数据了，在迁移之前把要迁移的数据获取出来。通过命令 cluster getkeysinslot {slot} {count}。Count 表示迁移的 Slot 的个数。
- （4）然后在源节点上执行，migrate {targetIP} {targetPort} “” 0 {timeout} keys {keys} 命令，把获取的键通过流水线批量迁移到目标节点。
- （5）重复 3 和 4 两步不断将数据迁移到目标节点。
- （6）完成数据迁移到目标节点以后，通过 cluster setslot {slot} node {targetNodeId} 命令通知对应的槽被分配到目标节点，并且广播这个信息给全网的其他主节点，更新自身的槽节点对应表。

### **收缩：**

- 迁移槽。
- 忘记节点。通过命令 cluster forget {downNodeId} 通知其他的节点

![](集群缩容.webp)

为了安全删除节点，Redis集群只能下线没有负责槽的节点。因此如果要下线有负责槽的master节点，则需要先将它负责的槽迁移到其他节点。迁移的过程也与上线操作类似，不同的是下线的时候需要通知全网的其他节点忘记自己，此时通过命令 cluster forget {downNodeId} 通知其他的节点。

## 数据分区算法

### 概念

分布式数据存储中，数据是分布式在不同的服务器上的，那么每条数据应该存储到哪台服务器？取的时候又应该去哪台服务器去取？分布式数据存储算法就是解决此类问题的算法

### 种类

#### hash算法

##### 过程

1. 客户端开始操作数据
2. 服务器对数据的key进行hash计算，得到一个数字
3. 服务器对得到的数字与服务器数量做取余计算，得到服务器的编号
4. 服务器在相应的服务器上进行操作

hash算法的数据存储过程图解：

![](https://secure2.wostatic.cn/static/pKaQRcM9dSsGiusH9e8C2f/image.png?auth_key=1716829063-uoLseP6r16bh3jid2Eff3d-0-b81f5208538928a6f80f83239720147d)

![](https://secure2.wostatic.cn/static/mi99MNwZbL4ufrZpxPA45E/image.png?auth_key=1716829063-6i2zsSHqfpn3J5MmmKmxrM-0-4fa9e0ad0bf980c099595bbbc6ac197f)

##### 缺点和使用现状

可能出现某个服务器上的热点数据特别多，导致该服务器出现性能瓶颈，如果某个服务器出现故障，会导致大部分数据的hash错乱，导致大部分数据读写失效或错误。由于hash算法的上述缺点，现在很少有分布式数据存储使用该算法

#### 原始一致性hash算法

##### 过程

1. 将服务器使用Hash函数进行一个哈希（一致性Hash算法将整个Hash空间组织成一个虚拟的圆环，Hash函数的值空间为0 ~ 2^32 - 1(一个32位无符号整型)），分布在一个圆环上，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台服务器就确定在了哈希环的一个位置上。
2. 客户端开始进行数据操作
3. 将数据Key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针查找，遇到的服务器就是其应该定位到的服务器。
4. 用得到的hash值在圆环对应的各个点上去对比，得到数据在圆环上的落点
5. 服务器顺时针寻找距离该落点最近的一个服务器节点
6. 服务器在相应的服务器上进行操作

服务器分布样式图解：

![](https://secure2.wostatic.cn/static/adgHTn4qGystLjDCMQR5Vm/image.png?auth_key=1716829063-vb41Y6ikFt5PjxriKNK9g4-0-c8d8d206c7bbdb14db672eb990315e3d)

##### 一致性hash算法的优点

如果某个服务器节点出现故障，只会导致该节点对应区间的数据出现错误，其他节点的数据仍然保存完整。

##### 一致性hash算法的缺点

因为实体服务器节点少量相比哈希环分片数据很少，这种特性决定了一致性哈希的数据倾斜，由于数量少导致服务节点分布不均，造成机器负载失衡。

### 基于虚拟节点的一致性hash算法

基于虚拟节点的一致性hash算法指的是在一致性hash算法的基础上，在每个虚拟节对应的区间上增加若干个其他节点的虚拟节点，这样就能最大限度的解决热点数据导致的服务器数据分布不均的问题。

基于虚拟节点的一致性hash算法图解：

![](https://secure2.wostatic.cn/static/sMUvZdX2cQCsiSBUJfx2QU/image.png?auth_key=1716829063-oSYhvZsVRkCcemXJ9HagFT-0-6c69ad5e8006abae05a67e6ab668ce04)

### Redis Hash Slot算法

#### 过程分析

Redis Hash Slot算法是Redis分布式中使用的分布式数据存储算法，他将数据存储在16384个hash槽中，然后再把这16384个hash槽平均分布在每个服务器上，这样用户进行数据操作，只需要找到数据对应的hash槽，然后再找到hash槽对应的服务器即可，这些slot有点类似一致性哈希中的虚拟节点整个过程如下：

1. 客户端开始操作数据
2. 服务器对数据的key进行CRC16值计算，得到一个数字
3. 服务器对得到的数字与16384做取余计算，得到对应的hash槽编号
4. 根据hash槽找到对应的服务器
5. 服务器在相应的服务器上进行操作

#### 优缺点分析

1. 如果某个服务器节点出现故障，只会导致该服务器节点上对应的hash槽上的数据出现故障，不影响其他数据
2. 如果某个服务器节点出现故障，redis会迅速把该节点上的hash槽转移到其他服务器节点上（redis底层做了优化），保证把损失降低到最低
3. 由于存在16384个hash槽，所以很好的解决了局部数据热点问题。

# 慢查询日志

[慢查询日志](Redis-extension.md#Redis慢日志查询)

# Redis的Scan和Keys命令

## Keys

### 简介

通过简单的正则就可以进行模糊匹配，没有分页，没有游标。就是暴力查找遍历。 好处就是方便，坏处应有仅有，Redis执行命令是单线程的，那就是如果说我这个线程查询的内容过多，导致查询时间很长就会出现其他线程的阻塞，或者超时的问题。查询的时间复杂度是O（n）

## Scan

### 简介

1. scan 复杂度为O（n）可带游标进行分步进行查询，不会阻塞线程
2. 可以进行模糊匹配和keys一样，只不过每一次都要带上一次返回的游标，可以使用limit限制最大条数，有可能少但是不会超过（[http://doc.redisfans.com/key/scan.html#scan）](http://doc.redisfans.com/key/scan.html#scan%EF%BC%89)
3. 每次根据游标返回的数据有可能为空也有可能为多个。只要返回的游标不为0，就不代表数据没有了。
4. 但是问题还是有的，就是有可能返回重复的key值这个得我们应用程序进行一次去重，可以使用set进行存储或者Map因为他两的数据结构本身就是不可以重复的。

### SCAN 内部探究

1. Redis 的全局就是使用的是key-value形式存储，使用的也就是他底层的数据结构dict字典。字典内部存储和java中的hashmap差不多，其底层都是通过数组和链表实现的。
2. 在dict中我们所存储的key就是底下的数组下标，数组下表是通过计算hash值出来的。正是因为有了hash冲突也就有了链表。
3. 在使用scan的时候我们其中scan的游标就是数组的下标，因为在存储的时候进行的是计算hash后进行存储，所以在数组上不是顺序存储，所以在一段数组上有可能有值也有可能没有值。也有可能一个slot上有多个值。所以这就是scan为什么会在增量式的过程中出现多个和0个的原因，如下图
    
    ![](https://secure2.wostatic.cn/static/mv5WFoKx5kzLJwPMEs73xN/image.png?auth_key=1716829170-eV9WU98NZ4sykB66CoCadA-0-09f1f71ba48b675540d5cae4793fea94)
    
4. 如果说按数组的下标顺序便利下去那要是扩容了怎么办，因为**扩容之后需要进行进行重新hash**，数组下标的位置就会改变，那么这个过程中我们我们在扩容之前scan返回的游标就不准确了吗？
5. **Redis处理扩容下表的方案是**：它不是从第一维数组的第 0 位一直遍历到末尾，而是采用了高位进位加法来遍历。之所以使用这样特殊的方式进行遍历，是考虑到字典的扩容和缩容时避免槽位的遍历重复和遗漏。
6. 选择高位进位加法的主要原因还是他进行扩容的特点，和hashMap的差不多，采用的是： *Java 中的 HashMap 有扩容的概念，当 loadFactor 达到阈值时，需要重新分配一个新的2 倍大小的数组，然后将所有的元素全部 rehash 挂到新的数组下面。rehash 就是将元素的hash 值对数组长度进行取模运算，因为长度变了，所以每个元素挂接的槽位可能也发生了变化。又因为数组的长度是 2n（我们在进行扩容的时候其容量都为2n的原因） 次方，所以取模运算等价于位与操作。
    
    ![](https://secure2.wostatic.cn/static/9eUqWQmrPuLN7dcVCZ6CRn/image.png?auth_key=1716829170-bAmBWWbLnGpNxPLRudUAfY-0-a30a6b96aaca1419ff300a0a0e075bf7)
    
    抽象一点说，假设开始槽位的二进制数是 xxx，那么该槽位中的元素将被 rehash 到 0xxx 和 1xxx(xxx+8) 中。 如果字典长度由 16 位扩容到 32 位，那么对于二进制槽位 xxxx 中的元素将被 rehash 到 0xxxx 和 1xxxx(xxxx+16) 中。*
    

```java
a mod 8 = a & (8-1) = a & 7
a mod 16 = a & (16-1) = a & 15
a mod 32 = a & (32-1) = a & 31
```

```
如下图就是缩容和扩容的一个对比

![](https://secure2.wostatic.cn/static/xnKBHaneiye1HoEqFHa49R/image.png?auth_key=1716829170-fdsJq74NGMCCJEFX4fkbyb-0-61189a6fc50cf150e47490fe1c0ffa17)

所以说redis采用了高位进位加法来遍历的。
```

```java
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); Java 1.8计算hash的方法

```

1. 还有就是在进行扩容的时候，会进行copy和rehash，redis的数据量会很大，所以一次性进行rehash会出现卡顿问题。所以redis采用的是渐进式的rehash。也就是不一次性进行rehash 而是慢慢的继续进行，所以这也是scan乎过程中得注意的，也就新老dict 都进行扫描，最后合并返回。
2. 我们可以再想想keys 是不是就不用考虑以上问题呢？，因为他是每次都是全量扫，不担心扩容了等问题。


# 命令处理流程

整个流程包括：服务器启动监听、接收命令请求并解析、执行命令请求、返回命令回复等。

![](Redis命令处理流程.png)

#### Server 启动时监听 socket

启动调用 initServer 方法：

1. 创建 eventLoop（事件机制）
2. 注册时间事件处理器
3. 注册文件事件（socket）处理器
4. 监听 socket 建立连接

#### 建立 Client

- redis-cli 建立 socket
- redis-server 为每个连接（socket）创建一个 Client 对象
- 创建文件事件监听 socket
- 指定事件处理函数

#### 读取socket数据到输入缓冲区

- 从 client 中读取客户端的查询缓冲区内容

#### 解析获取命令

- 将输入缓冲区中的数据解析成对应的命令
- 判断是单条命令还是多条命令并调用相应的解析器解析

#### 执行命令

解析成功后调用 `processCommand` 方法执行命令，如下图：
1. Server启动时监听socket
    
    启动调用 initServer方法：
    
    - 创建eventLoop(事件机制)
    - 注册时间事件处理器
    - 注册文件事件(socket)处理器
    - 监听 socket 建立连接
2. 建立Client
    
    - redis-cli建立socket
    - redis-server为每个连接(socket)创建一个 Client 对象
    - 创建文件事件监听socket
    - 指定事件处理函数
    - 读取socket数据到输入缓冲区
    - 从client中读取客户端的查询缓冲区内容。
3. 解析获取命令
    
    - 将输入缓冲区中的数据解析成对应的命令
    - 判断是单条命令还是多条命令并调用相应的解析器解析
4. 执行命令
    
    解析成功后调用processCommand 方法执行命令，如下图:
    
    ![](Redis命令执行流程图.png)
    
    大致分三个部分:
    1. 调用 `lookupCommand` 方法获得对应的 `redisCommand`
    2. 检测当前 Redis 是否可以执行该命令
    3. 调用 `call` 方法真正执行命令


# 事件处理机制

Redis服务器是典型的事件驱动系统。Redis将事件分为两大类：**文件事件**和**时间事件**。

## 文件事件

文件事件即Socket的读写事件，也就是IO事件。 客户端的连接、命令请求、数据回复、连接断开。

### 文件事件分派器

在redis中，对于文件事件的处理采用了Reactor模型。采用的是epoll的实现方式。

![](https://secure2.wostatic.cn/static/raoSnsaTgVexEg6Zoj2xGq/image.png?auth_key=1716829338-evUG5MYubrhFCvZ163mxb7-0-e050b016257d8354629143731f27fcd4)

**Redis在主循环中统一处理文件事件和时间事件，信号事件则由专门的handler来处理。**

### 主循环

```c
void aeMain(aeEventLoop *eventLoop) { 
    eventLoop->stop = 0;
    while (!eventLoop->stop) { //循环监听事件
        // 阻塞之前的处理
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        // 事件处理，第二个参数决定处理哪类事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    } 
}

```

### 事件处理器

#### 连接处理函数 acceptTCPHandler

当客户端向 Redis 建立 socket时，`aeEventLoop` 会调用 `acceptTcpHandler` 处理函数，服务器会为每个链接创建一个 Client 对象，并创建相应文件事件来监听socket的可读事件，并指定事件处理函数。

```c
// 当客户端建立链接时进行的eventloop处理函数 networking.c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
....
    // 层层调用，最后在anet.c 中 anetGenericAccept 方法中调用 socket 的 accept 方法 
    cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
    if (cfd == ANET_ERR) {
        if (errno != EWOULDBLOCK)
            serverLog(LL_WARNING, "Accepting client connection: %s", server.neterr);
        return;
    }
    serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
    /**
    * 进行socket 建立连接后的处理
    */
    acceptCommonHandler(cfd,0,cip);
} 
```

#### 请求处理函数 readQueryFromClient

当客户端通过 socket 发送来数据后，Redis 会调用 `readQueryFromClient` 方法,`readQueryFromClient` 方法会调用 read 方法从 socket 中读取数据到输入缓冲区中，然后判断其大小是否大于系统设置的 `client_max_querybuf_len`，如果大于，则向 Redis返回错误信息，并关闭 client。

```c
// 处理从client中读取客户端的输入缓冲区内容。
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    ....
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    // 从 fd 对应的socket中读取到 client 中的 querybuf 输入缓冲区
    nread = read(fd, c->querybuf+qblen, readlen);
    ....
    // 如果大于系统配置的最大客户端缓存区大小，也就是配置文件中的client-query-buffer-limit 
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();
        // 返回错误信息，并且关闭client
        bytes = sdscatrepr(bytes,c->querybuf,64);
        serverLog(LL_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClient(c);
        return;
    }
    if (!(c->flags & CLIENT_MASTER)) {
        // processInputBuffer 处理输入缓冲区
        processInputBuffer(c);
    } else {
        // 如果client是master的连接
        size_t prev_offset = c->reploff; processInputBuffer(c);
        // 判断是否同步偏移量发生变化，则通知到后续的slave 
        size_t applied = c->reploff - prev_offset;
        if (applied) {
            replicationFeedSlavesFromMasterStream(server.slaves, c->pending_querybuf, applied);
            sdsrange(c->pending_querybuf,applied,-1);
        } 
    }
}

```

#### 命令回复处理器 sendReplyToClient

`sendReplyToClient`函数是Redis的命令回复处理器，这个处理器负责将服务器执行命令后得到的命令回复通过套接字返回给客户端。

1. 将`outbuf`内容写入到套接字描述符并传输到客户端
2. `aeDeleteFileEvent` 用于删除文件写事件

## 时间事件

时间事件是服务器对定时操作的抽象，分为定时事件与周期事件。一个时间事件主要由以下三个属性组成:

- id(全局唯一id)
- when (毫秒时间戳，记录了时间事件的到达时间)
- timeProc(时间事件处理器，当时间到达时，Redis就会调用相应的处理器来处理事件)

	```C
	/* Time event structure
	 *
	 * 时间事件结构
	 */
	typedef struct aeTimeEvent {
	    // 时间事件的唯一标识符
	    long long id; /* time event identifier. */
	    // 事件的到达时间，存贮的是UNIX的时间戳
	    long when_sec; /* seconds */
	    long when_ms; /* milliseconds */
	    // 事件处理函数，当到达指定时间后调用该函数处理对应的问题 
	    aeTimeProc *timeProc;
	    // 事件释放函数
	    aeEventFinalizerProc *finalizerProc;
	    // 多路复用库的私有数据
	    void *clientData;
	    // 指向下个时间事件结构，形成链表
	    struct aeTimeEvent *next;
	} aeTimeEvent;
	```

> 编者注：关于时间事件这部分内容主要关注三点：
> 1. serverCron函数
> 2. 定时事件
> 3. 周期性事件
### serverCron

时间事件的最主要的应用是在redis服务器需要对自身的资源与配置进行定期的调整，从而确保服务器的长久运行，这些操作由`redis.c`中的`serverCron`函数实现。该时间事件主要进行以下操作:

1. 更新redis服务器各类统计信息，包括时间、内存占用、数据库占用等情况。
2. 清理数据库中的过期键值对。
3. 关闭和清理连接失败的客户端。
4. 尝试进行aof和rdb持久化操作。
5. 如果服务器是主服务器，会定期将数据向从服务器做同步操作。
6. 如果处于集群模式，对集群定期进行同步与连接测试操作。

redis服务器开启后，就会周期性执行此函数，直到redis服务器关闭为止。默认每秒执行10次，平均100毫秒执行一次，可以在redis配置文件的hz选项，调整该函数每秒执行的次数。

#### server.hz

serverCron在一秒内执行的次数 ， 在`redis/conf`中可以配置

```bash
hz 100
```

比如：server.hz是100，也就是servreCron的执行间隔是10ms

#### run_with_period

```c
#define run_with_period(_ms_) \
if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))
```

定时任务执行都是在10毫秒的基础上定时处理自己的任务(run_with_period(ms))，即调用 run_with_period(ms)[ms是指多长时间执行一次，单位是毫秒]来确定自己是否需要执行。

返回1表示执行。

假如有一些任务需要每500ms执行一次，就可以在serverCron中用run_with_period(500)把每500ms需要执行一次的工作控制起来。

### 定时事件

定时事件：让一段程序在指定的时间之后执行一次

aeTimeProc(时间处理器)的返回值是AE_NOMORE

该事件在达到后删除，之后不会再重复。

### 周期性事件

周期性事件：让一段程序每隔指定时间就执行一次

aeTimeProc(时间处理器)的返回值不是AE_NOMORE

当一个时间事件到达后，服务器会根据时间处理器的返回值，对时间事件的 when 属性进行更新，让这个事件在一段时间后再次达到。

serverCron就是一个典型的周期性事件。

## aeEventLoop

aeEventLoop 是整个事件驱动的核心，Redis自己的事件处理机制，它管理着文件事件表和时间事件列表， 不断地循环处理着就绪的文件事件和到期的时间事件。

![](https://secure2.wostatic.cn/static/brkew4FPXRHWqEy8igDPYQ/image.png?auth_key=1716829338-hnTAyaASB7xU9xCA6Bz9A7-0-e049df83e49d7bdcd469965d419b13c4)

```c
 
typedef struct aeEventLoop {
    //最大文件描述符的值
    int maxfd; /* highest file descriptor currently registered */ 
    //文件描述符的最大监听数
    int setsize; /* max number of file descriptors tracked */ 
    //用于生成时间事件的唯一标识id
    long long timeEventNextId;
    //用于检测系统时间是否变更(判断标准 now<lastTime)
    time_t lastTime; /* Used to detect system clock skew */ 
    //注册的文件事件
    aeFileEvent *events; /* Registered events */
    //已就绪的事件
    aeFiredEvent *fired; /* Fired events */
    //注册要使用的时间事件
    aeTimeEvent *timeEventHead;
    //停止标志，1表示停止
    int stop;
    //这个是处理底层特定API的数据，对于epoll来说，该结构体包含了epoll fd和epoll_event void *apidata; /* This is used for polling API specific data */ 
    //在调用processEvent前(即如果没有事件则睡眠)，调用该处理函数
    aeBeforeSleepProc *beforesleep;
    //在调用aeApiPoll后，调用该函数
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```

#### 初始化

Redis 服务端在其初始化函数 initServer 中，会创建事件管理器 aeEventLoop 对象。

函数 aeCreateEventLoop 将创建一个事件管理器，主要是初始化 aeEventLoop 的各个属性值，比如events 、 fired 、 timeEventHead 和 apidata :

- 首先创建 aeEventLoop 对象。、
- 初始化注册的文件事件表、就绪文件事件表。 指针指向注册的文件事件表、 指针指 向就绪文件事件表。表的内容在后面添加具体事件时进行初变更。
- 初始化时间事件列表，设置 timeEventHead 和 timeEventNextId 属性。
- 调用 aeApiCreate 函数创建 epoll 实例，并初始化 apidata 。

#### stop

停止标志，1表示停止，初始化为0。

文件事件: events, fired, apidata

aeFileEvent 结构体为已经注册并需要监听的事件的结构体。

```c
typedef struct aeFileEvent {
    // 监听事件类型掩码，
    // 值可以是 AE_READABLE 或 AE_WRITABLE ，
    // 或者 AE_READABLE | AE_WRITABLE
    int mask; /* one of AE_(READABLE|WRITABLE) */
    // 读事件处理器 
    aeFileProc *rfileProc;
    // 写事件处理器 
    aeFileProc *wfileProc;
    // 多路复用库的私有数据 
    void *clientData;
} aeFileEvent;
```

aeFiredEvent:已就绪的文件事件

```c
typedef struct aeFiredEvent { 
    // 已就绪文件描述符
    int fd;
    // 事件类型掩码，
    // 值可以是 AE_READABLE 或 AE_WRITABLE 
    // 或者是两者的或
    int mask;
} aeFiredEvent;
```

void *apidata: 在ae创建的时候，会被赋值为aeApiState结构体，结构体的定义如下:

```c
typedef struct aeApiState { 
    // epoll_event 实例描述符
    int epfd; 
    // 事件槽
    struct epoll_event *events;
} aeApiState;
```

这个结构体是为了epoll所准备的数据结构。redis可以选择不同的io多路复用方法。因此 apidata 是个 void类型，根据不同的io多路复用库来选择不同的实现

ae.c里面使用如下的方式来决定系统使用的机制:

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

### 时间事件: timeEventHead, beforesleep, aftersleep

aeTimeEvent结构体为时间事件，Redis 将所有时间事件都放在一个无序链表中，每次 Redis 会遍历整 个链表，查找所有已经到达的时间事件，并且调用相应的事件处理器。

```c
typedef struct aeTimeEvent { /* 全局唯一ID */
    long long id; /* time event identifier. */ 
    /* 秒精确的UNIX时间戳，记录时间事件到达的时间*/ 
    long when_sec; /* seconds */
    /* 毫秒精确的UNIX时间戳，记录时间事件到达的时间*/ 
    long when_ms; /* milliseconds */
    /* 时间处理器 */
    aeTimeProc *timeProc;
    /* 事件结束回调函数，析构一些资源*/ 
    aeEventFinalizerProc *finalizerProc; 
    /* 私有数据 */
    void *clientData;
    /* 前驱节点 */
    struct aeTimeEvent *prev;
    /* 后继节点 */
    struct aeTimeEvent *next;
} aeTimeEvent;
```

**beforesleep **对象是一个回调函数，在 redis-server 初始化时已经设置好了。

功能:

- 检测集群状态
- 随机释放已过期的键
- 在数据同步复制阶段取消客户端的阻塞
- 处理输入数据，并且同步副本信息
- 处理非阻塞的客户端请求
- AOF持久化存储策略，类似于mysql的bin log 使用挂起的输出缓冲区处理写入

aftersleep对象是一个回调函数，在IO多路复用与IO事件处理之间被调用。

### aeMain

aeMain 函数其实就是一个封装的 while 循环，循环中的代码会一直运行直到 eventLoop 的 stop 被设 置为1(true)。它会不停尝试调用 aeProcessEvents 对可能存在的多种事件进行处理，而 aeProcessEvents 就是实际用于处理事件的函数。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

aemain函数中，首先调用Beforesleep。这个方法在Redis每次进入sleep/wait去等待监听的端口发生 I/O事件之前被调用。当有事件发生时，调用aeProcessEvent进行处理。

### aeProcessEvent

首先计算距离当前时间最近的时间事件，以此计算一个超时时间；然后调用 aeApiPoll 函数去等待底层的I/O多路复用事件就绪；aeApiPoll 函数返回之后，会处理所有已经产生文件事件和已经达到的时间事件。

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    //processed记录这次调度执行了多少事件
    int processed = 0, numevents;
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;
    if (eventLoop->maxfd != -1 || ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT)) 
          //获取最近将要发生的时间事件
          shortest = aeSearchNearestTimer(eventLoop);
        //计算aeApiPoll的超时时间 
        if (shortest) {
            long now_sec, now_ms; //获取当前时间
            aeGetTime(&now_sec, &now_ms); tvp = &tv; 
            //计算距离下一次发生时间时间的时间间隔 
            long long ms = (shortest->when_sec - now_sec)*1000 + shortest->when_ms - now_ms;
            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            //没有时间事件
            if (flags & AE_DONT_WAIT) {
                //马上返回，不阻塞
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                tvp = NULL; //阻塞到文件事件发生 
            }
        }//等待文件事件发生，tvp为超时时间，超时马上返回(tvp为0表示马上，为null表示阻塞到事 件发生)
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            //处理触发的文件事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                //处理读事件 
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc) 
                    //处理写事件
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                processed++;
            } 
        }
    }
    if (flags & AE_TIME_EVENTS)
        //时间事件调度和执行 
        processed += processTimeEvents(eventLoop);
    return processed;
}

```

**计算最早时间事件的执行时间，获取文件时间可执行时间 **

aeSearchNearestTimer

aeProcessEvents 都会先 **计算最近的时间事件发生所需要等待的时间** ，然后调用 aeApiPoll 方法在这 段时间中等待事件的发生，在这段时间中如果发生了文件事件，就会优先处理文件事件，否则就会一直 等待，直到最近的时间事件需要触发

**堵塞等待文件事件产生**

aeApiPoll 用到了epoll，select，kqueue和evport四种实现方式。

**处理文件事件**

rfileProc 和 wfileProc 就是在文件事件被创建时传入的函数指针

处理读事件：rfileProc

处理写事件：wfileProc

**处理时间事件**

processTimeEvents 取得当前时间，循环时间事件链表，如果当前时间>=预订执行时间，则执行时间处理函数。

# 缓存问题

### 数据并发竞争

这里的并发指的是多个redis的client同时set 同一个key引起的并发问题。 多客户端(Jedis)同时并发写一个key，一个key的值是1，本来按顺序修改为2,3,4，最后是4，但是顺序变成了4,3,2，最后变成了2。

#### 第一种方案：分布式锁+时间戳

1. 整体技术方案
    
    这种情况，主要是准备一个分布式锁，大家去抢锁，抢到锁就做set操作。 加锁的目的实际上就是把并行读写改成串行读写的方式，从而来避免资源竞争。
    
    ![](https://secure2.wostatic.cn/static/2AATwFojdvr3pnvrKQDwoD/image.png?auth_key=1716829496-27UPfCMroxGTo42wkLkeKr-0-cb454f08f9ee12cd42a470ea80ab6f15)
    
2. Redis分布式锁的实现
    
    主要用到的redis函数是setnx()
    
    由于上面举的例子，要求key的操作需要顺序执行，所以需要保存一个时间戳判断set顺序。
    
    > 系统A key 1 {ValueA 7:00} 系统B key 1 { ValueB 7:05}
    
    假设系统B先抢到锁，将key1设置为{ValueB 7:05}。接下来系统A抢到锁，发现自己的key1的时间戳早于缓存中的时间戳(7:00<7:05)，那就不做set操作了。
    

#### 第二种方案：利用消息队列

在并发量过大的情况下,可以通过消息中间件进行处理,把并行读写进行串行化。 把Redis的set操作放在队列中使其串行化,必须的一个一个执行。

## Hot Key
[[#第一种方案：分布式锁+时间戳]]
当有大量的请求(几十万)访问某个Redis某个key时，由于流量集中达到网络上限，从而导致这个redis的 服务器宕机。造成缓存击穿，接下来对这个key的访问将直接访问数据库造成数据库崩溃，或者访问数据库回填Redis再访问Redis，继续崩溃。

![](https://secure2.wostatic.cn/static/kSiLGK3QoDYXmgfPKwabgD/image.png?auth_key=1716829496-fSwuV2AgjXu6ZY9Rmv77tY-0-b67ae03502865a1807631ff393cb424a)

### 如何发现热key

1. 预估热key，比如秒杀的商品、火爆的新闻等
2. 在客户端进行统计，实现简单，加一行代码即可
3. 如果是Proxy，比如Codis，可以在Proxy端收集
4. 利用Redis自带的命令，monitor、hotkeys。但是执行缓慢(不要用)
5. 利用基于大数据领域的流式计算技术来进行实时数据访问次数的统计，比如 Storm、Spark Streaming、Flink，这些技术都是可以的。发现热点数据后可以写到zookeeper中

![](https://secure2.wostatic.cn/static/vAYjWtdvTrf6bs1Mn327yo/image.png?auth_key=1716829496-hSW79zr3PDF3HtZgN5sbfX-0-aeff68b85f2ae18200de826ed942d816)

### 如何处理热Key:

1. 变分布式缓存为本地缓存
    
    发现热key后，把缓存数据取出后，直接加载到本地缓存中。可以采用Ehcache、Guava Cache都可以，这样系统在访问热key数据时就可以直接访问自己的缓存了。(数据不要求实时一致)
    
2. 在每个Redis主节点上备份热key数据，这样在读取时可以采用随机读取的方式，将访问压力负载到每个Redis上。
    
3. 利用对热点数据访问的限流熔断保护措施
    
    每个系统实例每秒最多请求缓存集群读操作不超过 400 次，一超过就可以熔断掉，不让请求缓存集群，直接返回一个空白信息，然后用户稍后会自行再次重新刷新页面之类的。(首页不行，系统友好性差)。通过系统层自己直接加限流熔断保护措施，可以很好的保护后面的缓存集群。
    

## Big Key

大key指的是存储的值(Value)非常大，常见场景:

- 热门话题下的讨论 大V的粉丝列表
- 序列化后的图片
- 没有及时处理的垃圾数据
- .....
### 大key的影响:

- 大key会大量占用内存，在集群中无法均衡，导致数据倾斜。
- Redis的性能下降，主从复制异常。
- 在主动删除或过期删除时会操作时间过长而引起服务阻塞

### 如何发现大key:

1. `redis-cli --bigkeys`命令。可以找到某个实例5种数据类型(String、hash、list、set、zset)的最大 key。但如果Redis 的key比较多，执行该命令会比较慢。
2. 获取生产Redis的RDB文件，通过rdbtools分析RDB生成csv文件，再导入MySQL或其他数据库中进行分析统计，根据`size_in_bytes`统计bigkey

### 大key的处理:

优化big key的原则就是string减少字符串长度，list、hash、set、zset等减少成员数。

1. string类型的big key，尽量不要存入Redis中，可以使用文档型数据库MongoDB或缓存到CDN上。如果必须用Redis存储，最好单独存储，不要和其他的key一起存储。采用一主一从或多从。
2. 单个简单的key存储的value很大，可以尝试将对象分拆成几个key-value， 使用mget获取值，这样分拆的意义在于分拆单次操作的压力，将操作压力平摊到多次操作中，降低对redis的IO影响。
3. hash， set，zset，list 中存储过多的元素，可以将这些元素分拆。(常见)

	```text
	以hash类型举例来说，对于field过多的场景，可以根据field进行hash取模，生成一个新的key，例如原来的  
	hash_key:{filed1:value, filed2:value, filed3:value ...}，可以hash取模后形成如下 key:value形式
	
	hash_key:1:{filed1:value}  
	hash_key:2:{filed2:value}  
	hash_key:3:{filed3:value}  
	...  
	取模后，将原先单个key分成多个key，每个key filed个数为原先的1/N
	```

4. 使用 lazy delete (unlink命令)
    
    删除指定的key(s),若key不存在则该key被跳过。但是，相比DEL会产生阻塞，该命令会在另一个线程中回收内存，因此它是非阻塞的。 这也是该命令名字的由来：仅将keys从key空间中删除，真正的数据删除会在后续异步操作。
	```bash
	redis> SET key1 "Hello"
	"OK"
	redis> SET key2 "World"
	"OK"
	redis> UNLINK key1 key2 key3
	(integer) 2
	```

> 思路：
> * 大key做拆分
> * 把大key放到单独的文档数据库中
> * 大key需要做隔离
## 缓存倾斜
https://blog.csdn.net/GoGleTech/article/details/137931259

