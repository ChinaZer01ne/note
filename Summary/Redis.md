#flashcards 

# 为什么要使用Redis

- DB缓存，减轻DB服务器压力
    
    一般情况下数据存在数据库中，应用程序直接操作数据库。 当访问量上万，数据库压力增大，可以采取的方案有: 读写分离，分库分表  
    当访问量达到10万、百万，需要引入缓存。 将已经访问过的内容或数据存储起来，当再次访问时先找缓存，缓存命中返回数据。 不命中再找数据库，并更新缓存。
    
- 提高系统响应
    
    数据库的数据是存在文件里，也就是硬盘。与内存做交换(swap) 在大量瞬间访问时(高并发)MySQL单机会因为频繁IO而造成无法响应。MySQL的InnoDB是有行锁 将数据缓存在Redis中，也就是存在了内存中。 内存天然支持高并发访问。可以瞬间处理大量请求。qps到达11万/S，读请求 8万写/S 。
    
- 做Session分离
    
    传统的session是由tomcat自己进行维护和管理。 集群或分布式环境，不同的tomcat管理各自的session。 只能在各个tomcat之间，通过网络和Io进行session的复制，极大的影响了系统的性能。 
	- 1、各个Tomcat间复制session，性能损耗
	- 2、不能保证各个Tomcat的Session数据同步 将登录成功后的Session信息，存放在Redis中，这样多个服务器(Tomcat)可以共享Session信息。 Redis的作用是数据的临时存储
- 做分布式锁(Redis)
    
    一般讲锁是多线程的锁，是在一个进程中的 多个进程(JVM)在并发时也会产生问题，也要控制时序性可以采用分布式锁。使用Redis实现 setNX
    
- 做乐观锁(Redis)
    
    同步锁和数据库中的行锁、表锁都是悲观锁 悲观锁的性能是比较低的，响应性比较差高性能、高响应(秒杀)采用乐观锁 Redis可以实现乐观锁 watch + incr

# Redis单线程架构

### 单线程模型

Redis客户端对服务端的每次调用都经历了**发送命令，执行命令，返回结果**三个过程。其中执行命令阶段，由于Redis是单线程来处理命令的，所有每一条到达服务端的命令不会立刻执行，所有的命令都会进入一个队列中，然后逐个被执行。并且多个客户端发送的命令的执行顺序是不确定的。但是可以确定的是不会有两条命令被同时执行，不会产生并发问题，这就是Redis的单线程基本模型。

### 单线程模型每秒万级别处理能力的原因

1. 纯内存访问。数据存放在内存中，内存的响应时间大约是100纳秒，这是Redis每秒万亿级别访问的重要基础。
2. 非阻塞I/O，Redis采用select(默认会根据操作系统进行选择O(1)的调用方式，比如epoll/kqueue/evport，如果没有则选择select)作为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的链接，读写关闭都转换为了时间，不在I/O上浪费过多的时间。
3. 单线程避免了线程切换和竞态产生的消耗。
4. Redis采用单线程模型，每条命令执行如果占用大量时间，会造成其他线程阻塞，对于Redis这种高性能服务是致命的，所以Redis是面向高速执行的数据库。
5. 无论什么版本，工作线程就是一个，6.x高版本出现了IO多线程
# 数据结构
## Redis的基本数据类型
- string  
    - int  
    - raw  
    - embstr  
* hash  
    - ziplist  
    - hashtable  
- list  
    - ziplist  
    - linkedlist  
- set  
    - intset  
    - hashtable  
- zset  
    - ziplist  
    - skiplist
## Redis的底层数据类型
* sds
* linkedlist
* hashtable（字典）
* skiplist（跳表）
* intset（整数集合）
* ziplist（压缩链表）
* quicklist
### sds
#### 结构  
- free  
- len  
- buf  
        
#### 相比C字符串优点  

- 很容易获取字符串长度  
	- len记录了长度  
- 没有缓冲区溢出的问题  
	- SDS在需要增加字符串时，自动检查内存空间是否足够，不够时会先申请足够的内存再进行增加，这个过程不需要手动干预，不会发生缓冲区溢出问题  
- 减少修改字符串长度的时候内存重分配的次数  
	- C字符串每次修改需要重新分配，sds可以预分配和惰性释放  
- 二进制安全  
	- C字符串只能保存文本，需要编码，不能有空格  
- 兼容部分C字符串
### linkedlist
#### 结构  
- 双向链表
### hashtable
#### 结构  
#### 解决hash冲突  
- 拉链法  
- entry有next指针  
#### 哈希表扩展与收缩的时机  

- 负载因子 的计算：`load_factor = ht[0].used / ht[0].size` ；  
- 扩展 ：服务器没有执行 `BGSAVE` 和 `BGREWRITEAOF` 命令，并且负载因子大于等于1；  
- 扩展 ：服务器 正在执行`BGSAVE` 和 `BGREWRITEAOF` 命令，并且负载因子大于等于5；  
	- 避免在执行该命令（子进程存在期间）时进行扩展操作，避免不必要的内存写入操作；  
- 收缩 ：负载因子小于0.1；  
        
#### rehash  

![](rehash.jpg)  
	
- 为 ht[1] 分配空间，若扩展，则 ht[1].size 为第一个大于等于 ht[0].used\*2 的 2 n 。若收缩，则 ht[1].size 为第一个大于等于 ht[0].used 的 2 n ；  
- 将ht[0]中的所有键值对rehash到ht[1]上；  
- 迁移完后，释放ht[0]，将ht[1]设置为ht[0]，创建一个空白哈希表ht[1]  

##### 渐进式  
	
- 当键值对成万上亿时，需要分多次、渐进式完成rehash；  

- 渐进式rehash的步骤![](渐进式.jpg)  
	- 为ht[1]分配空间；  
	- 将字典的索引计数器变量rehashidx设置为0，表示rehash正式开始；  
	- rehash期间，每个对字典操作完成后，将rehashidx++；  
	- 当ht[0]中的所有键值对rehash到ht[1]后，rehashidx设置为 -1；  

##### 渐进式hash期间的访问  

- 查找操作先查 ht[0] ，再查 ht[1] ；  
- 新增操作只在 ht[1] 新增，保证 ht[0] 只减不增；
### skiplist
> 多层的有序链表
### intset
- 整数集合升级  
- 不支持降级
### ziplist
- 由连续内存块组成的顺序型数据结构  
- 要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N) 了，因此压缩列表不适合保存过多的元素。
### quicklist


### Redis对象类型简介

Redis是一种key/value型数据库，其中，每个key和value都是使用对象表示的。比如，我们执行以下代码：

```PowerShell
redis>SET message "hello redis"  
```

|||
|---|---|
|`REDIS_STRING`|字符串对象|
|`REDIS_LIST`|列表对象|
|`REDIS_HASH`|哈希对象|
|`REDIS_SET`|集合对象|
|`REDIS_ZSET`|有序集合对象|

```c
/* 
 * Redis 对象 
 */  
typedef struct redisObject {  
    // 类型  
    unsigned type:4;          
    // 不使用(对齐位)  
    unsigned notused:2;  
    // 编码方式  
    unsigned encoding:4;  
    // LRU 时间（相对于 server.lruclock）  
    unsigned lru:22;  
    // 引用计数  
    int refcount;  
    // 指向对象的值  
    void *ptr;  
} robj;  
```

### Redis对象底层数据结构

底层数据结构共有八种，如下表所示：

|||
|---|---|
|`REDIS_ENCODING_INT`|`long` 类型的整数|
|`REDIS_ENCODING_EMBSTR`|`embstr` 编码的简单动态字符串|
|`REDIS_ENCODING_RAW`|简单动态字符串|
|`REDIS_ENCODING_HT`|字典|
|`REDIS_ENCODING_LINKEDLIST`|双端链表|
|`REDIS_ENCODING_ZIPLIST`|压缩列表|
|`REDIS_ENCODING_INTSET`|整数集合|
|`REDIS_ENCODING_SKIPLIST`|跳跃表和字典|

### 字符串对象

Redis的String能表达3种值的类型：字符串、整数、浮点数。100.01 是个六位的串

字符串对象的编码可以是int、raw或者embstr。

如果一个字符串的内容可以转换为long，那么该字符串就会被转换成long类型，对象的ptr指针就会指向该long值，并且对象类型用int类型表示。

普通的字符串有两种，embstr和raw。embstr应该是Redis3.0新增的数据结构，在2.8中是没有的。如果字符串对象的长度小于39字节，就用embstr对象。否则用raw对象。可以从下面这段代码看出：

```c
#define REDIS_ENCODING_EMBSTR_SIZE_LIMIT 39  
robj *createStringObject(char *ptr, size_t len) {  
    if (len <= REDIS_ENCODING_EMBSTR_SIZE_LIMIT)  
        return createEmbeddedStringObject(ptr,len);  
    else  
        return createRawStringObject(ptr,len);  
}
```

embstr的好处有如下几点：

- embstr的创建只需分配一次内存，而raw为两次（一次为sds分配对象，另一次为object分配对象，emstr省略了第一次）
- 相对的，释放内存的次数也由两次变为一次
- embstr的objet和sds放在一起，更好的利用缓存带来的优势。

需要注意的是，redis并未提供任何修改embstr的方式，即embstr是只读形式。对embstr的修改实际上是先转换为raw在进行修改。

raw和embstr的区别可以用下面两幅图所示：

![raw编码的字符串对象.png](https://secure2.wostatic.cn/static/mypGxQxeZWgDCnNFrYPhb/raw%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png?auth_key=1716829253-q6JCnfKer7aosxBUgACo57-0-cc25e100e557d0c42421e7b7db0efb9b)

![embstr编码的字符串对象](https://secure2.wostatic.cn/static/4Sg6cUcPe56feN8mn5mbxi/embstr%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png?auth_key=1716829253-vGyU4PqaqotVp8GYPRwn26-0-53e50313438aefd2921440d0e43798f1)

应用场景:

1. key和命令是字符串
2. 普通的赋值
3. incr用于乐观锁 incr：递增数字，可用于实现乐观锁 watch(事务)
4. setnx用于分布式锁 当value不存在时采用赋值，可用于实现分布式锁

### 列表对象

list列表类型可以存储有序、可重复的元素，获取头部或尾部附近的记录是极快的，list的元素个数最多为2^32-1个(40亿)。

列表对象的编码可以是ziplist或者linkedlist。

ziplist是一种压缩链表，它的好处是更能节省内存空间，因为她所存储的内容都是在连续的内存区域当中。当列表对象元素不大，每个元素也不大的时候，就采用ziplist存储。但当数据量过大时就ziplist就不是那么好用了。因为为了保证他存储内容在内存中的连续性，插入的复杂度是0(N)，即每次插入都会重新进行realloc，如下图所示，对象结构中ptr所指向的就是一个ziplist。整个ziplist只需要malloc一次，它们在内存中是一块连续的区域。

使用 ziplist 存储链表，ziplist是一种压缩链表，它的好处是更能节省内存空间，因为它所存储的内容都是在连续的内存区域当中的。

![ziplist编码的numbers列表对象](https://secure2.wostatic.cn/static/wGw5x9y7dGqb61rWH7hZL4/ziplist%E7%BC%96%E7%A0%81%E7%9A%84numbers%E5%88%97%E8%A1%A8%E5%AF%B9%E8%B1%A1.png?auth_key=1716829253-iPpBKdh469hJUKERwji3Zi-0-f333e504f998a96f094320749ea7c4a6)

linkedlist是一种双向链表。它的结构比较简单，节点中存放pre和next两个指针，还有节点相关的信息。当每增加一个node的时候，既需要malloc一块内存。

![linkedlist编码的numbers列表对象](https://secure2.wostatic.cn/static/iDcB23aMBPs4N98NqRQRpm/linkedlist%E7%BC%96%E7%A0%81%E7%9A%84numbers%E5%88%97%E8%A1%A8%E5%AF%B9%E8%B1%A1.png?auth_key=1716829253-xkfnqftQBTHRTT6Lsk2PDN-0-e7202b3d9115a04a3f61b864190a2259)

应用场景:

1. 作为栈或队列使用
2. 可用于各种列表，比如用户列表、商品列表、评论列表等。

### 哈希对象

Redis hash 是一个 string 类型的 field 和 value 的映射表，它提供了字段和字段值的映射。 每个 hash 可以存储 2^32 - 1 键值对(40多亿)。

哈希对象的底层实现可以是ziplist或者hashtable

ziplist中的哈希对象是按照key1，value1，key2，value2这样的顺序存放来存储的。当对象数目不多且内容不大时，这种方式效率时很高的。

hashtable的是由dict这个结构来实现的。

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */

```

dict是一个字典，其中的指针dicht ht[2]指向了两个哈希表

```c
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

dict[0]是用于真正存放数据，dict[1]一般在哈希表元素过多进行rehash的时候用于中传数据。dictht中的table用于真正存放元素了，每个key/value对用一个dictEntry表示，放在dictEntry数组中。

![普通状态下的字典](https://secure2.wostatic.cn/static/ajxENq6FeryaXkuwvJtbdM/%E6%99%AE%E9%80%9A%E7%8A%B6%E6%80%81%E4%B8%8B%E7%9A%84%E5%AD%97%E5%85%B8.png?auth_key=1716829253-kYoJUWq6Wz1u2A5K6Z4NXV-0-32f0d2bd53bd50bd628c249858d3ba64)

应用场景:

对象的存储 ，表数据的映射

### 集合对象

Set：无序、唯一元素，集合中最大的成员数为 2^32 - 1

集合对象的编码可以是intset或者hashtable

intset是一个整数集合，里面存的为某种同一类型的证书，支持如下三种长度的整数：

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

intset是一个有序集合，查找元素的复杂度为O(logN)，但插入时不一定为O(logN)，因为可能涉及到升级操作。比如当集合里全是int16_t型的整数，这时要插入一个int32_t，那么为了维持集合中数据类型的一致，那么所有的数据都会被转成int31_t类型，涉及到内存的重新分配，这时插入的复杂度就为O(N)了。intset不支持降级操作。

应用场景：适用于不能重复的且不需要顺序的数据结构，比如：关注的用户，还可以通过spop进行随机抽奖

### 有序集合

SortedSet(ZSet) 有序集合元素本身是无序不重复的，每个元素关联一个分数(score) 可按分数排序，分数可重复

有序集合的编码可能两种，一种是ziplist，另一种是skiplist与dict的结合。

ziplist作为集合和作为哈希对象是一样的，member和score顺序存放。按照score从小到大顺序排列。它的结构不再复述。

skiplist是一种跳跃表，它实现了有序集合中的快速查找，在大多数情况下它的速度都可以和平衡树差不多。但她的实现比较简单，可以作为平衡树的替代品，他的结构比较特殊。下面分别是跳表skiplist和它内部的节点skiplistNode的结构体：

```c
/*
 * 跳跃表
 */
typedef struct zskiplist {
    // 头节点，尾节点
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 目前表内节点的最大层数
    int level;
} zskiplist;
/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {
    // member 对象
    robj *obj;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 这个层跨越的节点数量
        unsigned int span;
    } level[];
} zskiplistNode;
```

head和tail分别指向头结点和尾节点，然后每个skiplistNode里面的结构又是分层的（即level数组）

如图所示：

![跳表](https://secure2.wostatic.cn/static/5XVAX4PbwqLEinvk2r2eqd/%E8%B7%B3%E8%A1%A8.png?auth_key=1716829253-q2unXURj8Eby53EQNVe3dC-0-ef3dbaf27c032ff160116db41a9c6786)

每一列都代表一个节点，保存了member和score，按score从小到大排序。每个节点有不同的层数，这个层数是在生成节点的时候随机生成的数值。每一层都是一个指向后面某个节点的指针。这种结构使得跳跃表可以跨越很多节点来快速访问。

前面说到了，有序集合ZSET是有跳跃表和hashtable共同形成的。

```c
typedef struct zset {
    // 字典
    dict *dict;
    // 跳跃表
    zskiplist *zsl;
} zset;
```

应用场景:

由于可以按照分值排序，所以适用于各种排行榜。比如:点击排行榜、销量排行榜、关注排行榜等。

- string字符串类型
- hash类型
- list列表类型
- set集合类型
- sortedset(zset)有序集合类型
- bitmap位图类型
- geo地理位置类型
- stream类型（Redis5.0新增）

注意：Redis中命令是忽略大小写，(set SET)，key是不忽略大小写的 (NAME name)

# string字符串类型

Redis的String能表达3种值的类型:字符串、整数、浮点数 100.01 是个六位的串

应用场景:

1. key和命令是字符串
2. 普通的赋值
3. incr用于乐观锁 incr:递增数字，可用于实现乐观锁 watch(事务)
4. setnx用于分布式锁 当value不存在时采用赋值，可用于实现分布式锁

# list列表类型

list列表类型可以存储有序、可重复的元素，获取头部或尾部附近的记录是极快的，list的元素个数最多为2^32-1个(40亿)

应用场景:

1. 作为栈或队列使用
2. 可用于各种列表，比如用户列表、商品列表、评论列表等。

# set集合类型

Set：无序、唯一元素，集合中最大的成员数为 2^32 - 1

应用场景：适用于不能重复的且不需要顺序的数据结构，比如：关注的用户，还可以通过spop进行随机抽奖

# sortedset有序集合类型

SortedSet(ZSet) 有序集合元素本身是无序不重复的，每个元素关联一个分数(score) 可按分数排序，分数可重复

应用场景:

由于可以按照分值排序，所以适用于各种排行榜。比如:点击排行榜、销量排行榜、关注排行榜等。

# hash类型(散列表)

Redis hash 是一个 string 类型的 field 和 value 的映射表，它提供了字段和字段值的映射。 每个 hash 可以存储 2^32 - 1 键值对(40多亿)。

![](https://secure2.wostatic.cn/static/vwwR9bMFWRVqSUQ4z8SKjn/image.png?auth_key=1716829454-n386gUBdWFxqpwvaWW7FJ2-0-7e55e41fbebc91a718939018498b8ffa)

应用场景:

对象的存储 ，表数据的映射

# bitmap位图类型

bitmap是进行位操作的 通过一个bit位来表示某个元素对应的值或者状态,其中的key就是对应元素本身。 bitmap本身会极大的节省储存空间。

应用场景:

1. 用户每月签到，用户id为key ， 日期作为偏移量 1表示签到
2. 统计活跃用户, 日期为key，用户id为偏移量 1表示活跃
3. 查询用户在线状态， 日期为key，用户id为偏移量 1表示在线

# geo地理位置类型

geo是Redis用来处理位置信息的。在Redis3.2中正式使用。主要是利用了Z阶曲线、Base32编码和geohash算法

## Z阶曲线

在x轴和y轴上将十进制数转化为二进制数，采用x轴和y轴对应的二进制数依次交叉后得到一个六位数编码。把数字从小到大依次连起来的曲线称为Z阶曲线，Z阶曲线是把多维转换成一维的一种方法。

![](https://secure2.wostatic.cn/static/xjY7uax4m3uffL5Mas8fN3/image.png?auth_key=1716829454-p6HR4WTjFhd7nBBUw3QhU5-0-8b9ca2a70f79d21fa2a0d3d63823b737)

## Base32编码

Base32这种数据编码机制，主要用来把二进制数据编码成可见的字符串，其编码规则是:任意给定一 个二进制数据，以5个位(bit)为一组进行切分(base64以6个位(bit)为一组)，对切分而成的每个组进行编 码得到1个可见字符。Base32编码表字符集中的字符总数为32个(0-9、b-z去掉a、i、l、o)，这也是 Base32名字的由来。

![](https://secure2.wostatic.cn/static/4V5kuMfuEyjA9CS7BXpGir/image.png?auth_key=1716829454-b8hj1ESBXwWZbvXXTzYCLT-0-7f13199c3bcc8a851031f1c7ba067a05)

## geohash算法

Gustavo在2008年2月上线了geohash.org网站。Geohash是一种地理位置信息编码方法。 经过 geohash映射后，地球上任意位置的经纬度坐标可以表示成一个较短的字符串。可以方便的存储在数据库中，附在邮件上，以及方便的使用在其他服务中。以北京的坐标举例，[39.928167,116.389550]可以 转换成 wx4g0s8q3jf9 。

Redis中经纬度使用52位的整数进行编码，放进zset中，zset的value元素是key，score是GeoHash的 52位整数值。在使用Redis进行Geo查询时，其内部对应的操作其实只是zset(skiplist)的操作。通过zset 的score进行排序就可以得到坐标附近的其它元素，通过将score还原成坐标值就可以得到元素的原始坐标。

应用场景:

1. 记录地理位置
2. 计算距离
3. 查找"附近的人"

# stream数据流类型

stream是Redis5.0后新增的数据结构，用于可持久化的消息队列。 几乎满足了消息队列具备的全部内容，包括：消息ID的序列化生成、消息遍历、消息的阻塞和非阻塞读取、消息的分组消费、未完成消息的处理、消息队列监控

每个Stream都有唯一的名称，它就是Redis的key，首次使用 xadd 指令追加消息时自动创建。

应用场景:

消息队列的使用

# 底层数据结构

Redis作为Key-Value存储系统，数据结构如下:

![](https://secure2.wostatic.cn/static/t3ngk1rd8QUHptbjaJeMfD/image.png?auth_key=1716829454-o2T4qt99hzP5FpSBTsgCks-0-e9e9b680017d191e6c9d291ce47761a0)

Redis没有表的概念，Redis实例所对应的db以编号区分，db本身就是key的命名空间。 比如:user:1000作为key值，表示在user这个命名空间下id为1000的元素，类似于user表的id=1000的行。

## RedisDB结构

Redis中存在“数据库”的概念，该结构由redis.h中的redisDb定义。 当redis 服务器初始化时，会预先分配 16 个数据库，所有数据库保存到结构 redisServer 的一个成员 redisServer.db 数组中，redisClient中存在一个名叫db的指针指向当前使用的数据库

RedisDB结构体源码:

```c
typedef struct redisDb {
    int id; //id是数据库序号，为0-15(默认Redis有16个数据库)
    long avg_ttl; //存储的数据库对象的平均ttl(time to live)，用于统计
    dict *dict; //存储数据库所有的key-value
    dict *expires; //存储key的过期时间
    dict *blocking_keys;//blpop 存储阻塞key和客户端对象
    dict *ready_keys;//阻塞后push 响应阻塞客户端 存储阻塞后push的key和客户端对象 
    dict *watched_keys;//存储watch监控的的key和客户端对象
} redisDb;
```

id：id是数据库序号，为0-15(默认Redis有16个数据库)

dict：存储数据库所有的key-value，后面要详细讲解

expires：存储key的过期时间，后面要详细讲解

## RedisObject结构

Value是一个对象，包含字符串对象，列表对象，哈希对象，集合对象和有序集合对象

结构信息概览

```c
typedef struct redisObject {
    unsigned type:4;//类型 五种对象类型
    unsigned encoding:4;//编码
    void *ptr;//指向底层实现数据结构的指针
    //...
    int refcount;//引用计数
    //...
    unsigned lru:LRU_BITS; //LRU_BITS为24bit 记录最后一次被命令程序访问的时间 
    //...
}robj;
```

**4位type**：type 字段表示对象的类型，占 4 位;

REDIS_STRING(字符串)、REDIS_LIST (列表)、REDIS_HASH(哈希)、REDIS_SET(集合)、REDIS_ZSET(有序集合)。

当我们执行 type 命令时，便是通过读取 RedisObject 的 type 字段获得对象的类型。

```bash
127.0.0.1:6379> type a1
string
```

**4位encoding**：encoding 表示对象的内部编码，占 4 位

每个对象有不同的实现编码

Redis 可以根据不同的使用场景来为对象设置不同的编码，大大提高了 Redis 的灵活性和效率。 通过 object encoding 命令，可以查看对象采用的编码方式

```bash
127.0.0.1:6379>  object encoding a1
"int"
```

**24位LRU **

lru 记录的是对象最后一次被命令程序访问的时间，( 4.0 版本占 24 位，2.6 版本占 22 位)。 高16位存储一个分钟数级别的时间戳，低8位存储访问计数(lfu : 最近访问次数)  
淘汰策略lru----> 高16位： 最后被访问的时间  
淘汰策略lfu----->低8位：最近访问次数

**refcount **

refcount 记录的是该对象被引用的次数，类型为整型。

refcount 的作用，主要在于对象的引用计数和内存回收。 当对象的refcount>1时，称为共享对象

Redis 为了节省内存，当有一些对象重复出现时，新的程序不会创建新的对象，而是仍然使用原来的对象。

**ptr**

ptr 指针指向具体的数据，比如:set hello world，ptr 指向包含字符串 world 的 SDS。

## 7种type

### 字符串对象

C语言: 字符数组 "\0"

Redis 使用了 SDS(Simple Dynamic String)。用于存储字符串和整型数据。

![](https://secure2.wostatic.cn/static/mqTA13LVh7PiZxT2fgFUpR/image.png?auth_key=1716829454-sa9LLVcavxy7JX5DK6t6Pn-0-3c9c6d45cfedfcb0d69ea2b6aeba08bd)

```c
struct sdshdr{ 
    //记录buf数组中已使用字节的数量 
    int len;
    //记录 buf 数组中未使用字节的数量 
    int free; 
    //字符数组，用于保存字符串
    char buf[];
}
```

buf[] 的长度=len+free+1

**SDS的优势:**

1. SDS 在 C语言字符串的基础上加入了 free 和 len 字段，获取字符串长度:SDS 是 O(1)，C语言获取字符串长度是 O(n)。
2. SDS 由于记录了长度，在可能造成缓冲区溢出时会自动重新分配内存，杜绝了缓冲区溢出。
3. 可以存取二进制数据，以字符串长度len来作为结束标识  
    C: \0 空字符串 二进制数据包括空字符串，所以没有办法存取二进制数据  
    SDS : 存非二进制 \0作为判断 ，存二进制: 字符串长度作为判断

**使用场景: **

SDS的主要应用在：存储字符串和整型数据、存储key、AOF缓冲区和用户输入缓冲。

### 跳跃表(重点)

跳跃表是有序集合(sorted-set)的底层实现，效率高，实现简单。

跳跃表的基本思想:

将有序链表中的部分节点分层，每一层都是一个有序链表。

#### 查找

在查找时优先从最高层开始向后查找，当到达某个节点时，如果next节点值大于要查找的值或next指针 指向null，则从当前节点下降一层继续向后查找。

举例:

![](https://secure2.wostatic.cn/static/uqtUZiS7fYvn69WQ5gQqZh/image.png?auth_key=1716829456-KFq4SEwNBuy5M58HnXBEy-0-5e5da4d2044d3dc39e4f0216a9c0b472)

查找元素9，按道理我们需要从头结点开始遍历，一共遍历8个结点才能找到元素9。

第一次分层: 遍历5次找到元素9(红色的线为查找路径)

![](https://secure2.wostatic.cn/static/6n2dLJSeEPP5uYbQmM9QCN/image.png?auth_key=1716829456-pu3zy5WQaqVvUFYTAN62XD-0-bc645e91de0b3bf4efc8d07985559d6d)

第二次分层: 遍历4次找到元素9

![](https://secure2.wostatic.cn/static/hYUJXLkgSyGr7ks1Nxfkp4/image.png?auth_key=1716829456-psfLfsKB9uxbpz5P5LcSfQ-0-21aac8c35df3a9a353e98530ebb2b8e7)

第三层分层: 遍历4次找到元素9

![](https://secure2.wostatic.cn/static/eRFvNMXQJHAVA8ggkbE3za/image.png?auth_key=1716829456-vM8jJw6pisvVrxe3a42vu6-0-4664c7194ce91254e609194c409e03e5)

这种数据结构，就是跳跃表，它具有二分查找的功能。 插入与删除 上面例子中，9个结点，一共4层，是理想的跳跃表。

通过抛硬币(概率1/2)的方式来决定新插入结点跨越的层数：正面：插入上层，背面：不插入，达到1/2概率(计算次数)

#### 删除

找到指定元素并删除每层的该元素即可

#### 跳跃表特点

每层都是一个有序链表

查找次数近似于层数(1/2)

底层包含所有元素

空间复杂度 O(n) 扩充了一倍

#### Redis跳跃表的实现

```c
//跳跃表节点
typedef struct zskiplistNode {
    sds ele; /* 存储字符串类型数据 redis3.0版本中使用robj类型表示， 但是在redis4.0.1中直接使用sds类型表示 */
    double score;//存储排序的分值
    struct zskiplistNode *backward;//后退指针，指向当前节点最底层的前一个节点 
    /*
      层，柔性数组，随机生成1-64的值 
      */
    struct zskiplistLevel {
        struct zskiplistNode *forward; //指向本层下一个节点
        unsigned int span;//本层下个节点到本节点的元素个数 
    } level[];
} zskiplistNode;

//链表
typedef struct zskiplist{
    //表头节点和表尾节点
    structz skiplistNode *header, *tail; //表中节点的数量
    unsigned long length; //表中层数最大的节点的层数
    int level;
}zskiplist;
```

**完整的跳跃表结构体:**

![](https://secure2.wostatic.cn/static/4FmRuKdfBbsc9CXPteAzip/image.png?auth_key=1716829456-w26cwgMQ4uWHFs7CCPkbeK-0-4b9aad4d8569924896f104709496d267)

**跳跃表的优势: **

1、可以快速查找到需要的节点

2、可以在O(1)的时间复杂度下，快速获得跳跃表的头节点、尾结点、长度和高度。

**应用场景：**

有序集合的实现

### 字典(重点+难点)

字典dict又称散列表(hash)，是用来存储键值对的一种数据结构。

Redis整个数据库是用字典来存储的。(K-V结构)

对Redis进行CURD操作其实就是对字典中的数据进行CURD操作。

#### **数组**

数组：用来存储数据的容器，采用头指针+偏移量的方式能够以O(1)的时间复杂度定位到数据所在的内存地址。

Redis 海量存储 快

#### **Hash函数 **

Hash(散列)，作用是把任意长度的输入通过散列算法转换成固定类型、固定长度的散列值。 hash函数可以把Redis里的key:包括字符串、整数、浮点数统一转换成整数。

key=100.1 String “100.1” 5位长度的字符串

Redis-cli :times 33

Redis-Server : siphash

**数组下标**=hash(key)%数组容量(hash值%数组容量得到的余数)

#### **Hash冲突 **

不同的key经过计算后出现数组下标一致，称为Hash冲突。 采用单链表在相同的下标位置处存储原始key和value 当根据key找Value时，找到数组下标，遍历单链表可以找出key相同的value

![](https://secure2.wostatic.cn/static/6hWqGH69bGKi78Z6DYpMhr/image.png?auth_key=1716829456-rDDBYEBb5q3xQCX6XVgbrV-0-2b0c276690a39bfb0773c5e30637ad66)

#### Redis字典的实现

Redis字典实现包括:字典(dict)、Hash表(dictht)、Hash表节点(dictEntry)。

![](https://secure2.wostatic.cn/static/PniR1CcVS2m5SAMCLeTLN/image.png?auth_key=1716829456-7NsgcphwSKCCAL9wXQ27fb-0-b79cfcedf2af6c4b01b9dd89671c13c8)

**Hash表**

```c
typedef struct dictht {
    dictEntry **table; // 哈希表数组
    unsigned long size; // 哈希表数组的大小
    unsigned long sizemask; // 用于映射位置的掩码，值永远等于(size-1)
    unsigned long used; // 哈希表已有节点的数量,包含next单链表数据
} dictht; 

```

1. hash表的数组初始容量为4，随着k-v存储量的增加需要对hash表数组进行扩容，新扩容量为当前量的一倍，即4,8,16,32
2. 索引值=Hash值&掩码值(Hash值与Hash表容量取余) Hash表节点

```c
typedef struct dictEntry {
    void *key; // 键
    union {
        void *val; // 值v的类型可以是以下4种类型
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // 指向下一个哈希表节点，形成单向链表 解决hash冲突
} dictEntry;



```

key字段存储的是键值对中的键 v字段是个联合体，存储的是键值对中的值。 next指向下一个哈希表节点，用于解决hash冲突

![](https://secure2.wostatic.cn/static/kCE3vXzLzpxNA1dVueRgVH/image.png?auth_key=1716829456-bTBhJvQdfmFngfiv5mrpQW-0-9b12fedd770564ac749d8a8abad73552)

dict字典

```c
typedef struct dict {
    dictType *type; // 该字典对应的特定操作函数
    void *privdata; // 上述类型函数对应的可选参数
    dictht ht[2];  /* 两张哈希表，存储键值对数据，ht[0]为原生hash表
                          ht[1]为 rehash 哈希表 */ 
    long rehashidx; rehash， /*rehash标识 当等于-1时表示没有在
                            否则表示正在进行rehash操作，存储的值表示 ht[0]的rehash进行到哪个索引值(数组下标)*/
    int iterators; // 当前运行的迭代器数量
} dict;



```

type字段，指向dictType结构体，里边包括了对该字典操作的函数指针

```c
typedef struct dictType {
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 比较键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

Redis字典除了主数据库的K-V数据存储以外，还可以用于：散列表对象、哨兵模式中的主从节点管理等在不同的应用中，字典的形态都可能不同，dictType是为了实现各种形态的字典而抽象出来的操作函数 (多态)。

#### 完整的Redis字典数据结构:

![](https://secure2.wostatic.cn/static/wrZxJNc4nyviG5nA9vdNpq/image.png?auth_key=1716829456-iGJpVeCe1ZtP2E3rZyLqJA-0-34197c095bebfc0a3c41df1587861e41)

#### 字典扩容

字典达到存储上限，需要rehash(扩容) 扩容流程:

![](https://secure2.wostatic.cn/static/t1ix5kvXPRajiNthrPgdNJ/image.png?auth_key=1716829456-tsUjn5utQgftiJeEs9HsQ9-0-04c2e71de641c3f700a118b31bef3f93)

说明:

1. 初次申请默认容量为4个dictEntry，非初次申请为当前hash表容量的一倍。
2. rehashidx=0表示要进行rehash操作。
3. 新增加的数据在新的hash表h[1]
4. 修改、删除、查询在老hash表h[0]、新hash表h[1]中(rehash中)
5. 将老的hash表h[0]的数据重新计算索引值后全部迁移到新的hash表h[1]中，这个过程称为 rehash。

#### 渐进式rehash

当数据量巨大时rehash的过程是非常缓慢的，所以需要进行优化。 服务器忙，则只对一个节点进行rehash，服务器闲，可批量rehash(100节点)

**字典应用场景: **

1. 主数据库的K-V数据存储
2. 散列表对象(hash)
3. 哨兵模式中的主从节点管理

## 压缩列表

压缩列表(ziplist)是由一系列特殊编码的连续内存块组成的顺序型数据结构。节省内存

是一个字节数组，可以包含多个节点(entry)。每个节点可以保存一个字节数组或一个整数。

压缩列表的数据结构如下:

![](https://secure2.wostatic.cn/static/fbcxtkF27yqQTsGg7Hsu9Q/image.png?auth_key=1716829456-GDuigF2vJuzBvGsDqVjca-0-dca87514747e857335d0a38428afcf04)

- zlbytes：压缩列表的字节长度
    
- zltail：压缩列表尾元素相对于压缩列表起始地址的偏移量
    
- zllen：压缩列表的元素个数
    
- entry1..entryX：压缩列表的各个节点
    
- zlend：压缩列表的结尾，占一个字节，恒为0xFF(255)
    
- entryX元素的编码结构：
    
    ![](https://secure2.wostatic.cn/static/iH8aSoicc1GKPQw3Rj7Z7w/image.png?auth_key=1716829456-s1qPNhEZFX3bVPQrqZvnQk-0-18a0262a3ab3565544af371207f65bb5)
    
- previous_entry_length：前一个元素的字节长度
    
- encoding：表示当前元素的编码
    
- content：数据内容
    

ziplist结构体如下:

```c
typedef struct zlentry {
    unsigned int prevrawlensize; //previous_entry_length字段的长度
    unsigned int prevrawlen;  //previous_entry_length字段存储的内容
    unsigned int lensize; //encoding字段的长度
    unsigned int len;  //数据内容长度
    unsigned int headersize;  //当前元素的首部长度，即previous_entry_length字段长度与 encoding字段长度之和。、
    unsigned char encoding; //数据类型
    unsigned char *p;  //当前元素首地址
} zlentry;

```

**应用场景: **

sorted-set和hash元素个数少且是小整数或短字符串(直接使用) list用快速链表(quicklist)数据结构存储，而快速链表是双向列表与压缩列表的组合。(间接使用)

## 整数集合

整数集合(intset)是一个有序的(整数升序)、存储整数的连续存储结构。

当Redis集合类型的元素都是整数并且都处在64位有符号整数范围内(2^64)，使用该结构体存储。

```bash
127.0.0.1:6379> sadd set:001 1  3 5 6 2
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
127.0.0.1:6379> sadd set:004 1 100000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
```

intset的结构图如下:

![](https://secure2.wostatic.cn/static/me13p4vsyKaqkaRqz19PY5/image.png?auth_key=1716829456-f9YWR8ammBrawJimL7NxuT-0-77ef2b5259c3418b256d8b23b7f45d75)

```c
typedef struct intset{
    uint32_t encoding;  //编码方式
    uint32_t length;   //集合包含的元素数量
    int8_t contents[]; //保存元素的数组 
}intset;
```

**应用场景: **

可以保存类型为int16_t、int32_t 或者int64_t 的整数值，并且保证集合中不会出现重复元素。

## 快速列表(重要)

快速列表(quicklist)是Redis底层重要的数据结构。是列表的底层实现。(在Redis3.2之前，Redis采用双向链表(adlist)和压缩列表(ziplist)实现。)在Redis3.2以后结合adlist和ziplist的优势Redis设计出了quicklist。

```bash
127.0.0.1:6379> lpush list:001 1 2 5 4 3
(integer) 5
127.0.0.1:6379> object encoding list:001
"quicklist"
```

双向列表(adlist)

![](https://secure2.wostatic.cn/static/en76uTJFPypuQYMPyDEADM/image.png?auth_key=1716829456-72gVpzpQHmW968BVb5UJKS-0-98a94be26b4cec9c42a6ea020fcaefad)

双向链表优势:

1. 双向：链表具有前置节点和后置节点的引用，获取这两个节点时间复杂度都为O(1)。
    
2. 普通链表(单链表)：节点类保留下一节点的引用。链表类只保留头节点的引用，只能从头节点插入删除
    
3. 无环:表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL,对链表的访问都是以 NULL 结束。
    
    环状:头的前一个节点指向尾节点
    
4. 带链表长度计数器：通过 len 属性获取链表长度的时间复杂度为 O(1)。
    
5. 多态：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。
    

快速列表

quicklist是一个双向链表，链表中的每个节点时一个ziplist结构。quicklist中的每个节点ziplist都能够存储多个数据元素。

![](https://secure2.wostatic.cn/static/2bDpFApGsqP57ssYFdiA1Z/image.png?auth_key=1716829456-2C4tJ94P8aod31YFmQc8vr-0-a7d222605b82052dc9b0f02dcb8f140f)

quicklist的结构定义如下:

```c
typedef struct quicklist {
    quicklistNode *head;// 指向quicklist的头部
    quicklistNode *tail;// 指向quicklist的尾部
    unsigned long count;// 列表中所有数据项的个数总和
    unsigned int len;// quicklist节点的个数，即ziplist的个数
    int fill : 16; // ziplist大小限定，由list-max-ziplist-size给定(Redis设定)
    unsigned int compress : 16; // 节点压缩深度设置，由list-compress-depth给定(Redis设定)
} quicklist;

```

quicklistNode的结构定义如下:

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;// 指向上一个ziplist节点
    struct quicklistNode *next;// 指向下一个ziplist节点
    unsigned char *zl;// 数据指针，如果没有被压缩，就指向ziplist结构，反之指向quicklistLZF结构
    unsigned int sz;// 表示指向ziplist结构的总长度(内存占用长度)
    unsigned int count : 16;// 表示ziplist中的数据项个数
    unsigned int encoding : 2;// 编码方式，1--ziplist，2--quicklistLZF
    unsigned int container : 2;// 预留字段，存放数据的方式，1--NONE，2--ziplist 
    unsigned int recompress : 1;// 解压标记，当查看一个被压缩的数据时，需要暂时解压，标记此参数为1，之后再重新进行压缩 
    unsigned int attempted_compress : 1;// 测试相关
    unsigned int extra : 10; // 扩展字段，暂时没用
} quicklistNode;

```

**数据压缩**

quicklist每个节点的实际数据存储结构为ziplist，这种结构的优势在于节省存储空间。为了进一步降低 ziplist的存储空间，还可以对ziplist进行压缩。Redis采用的压缩算法是LZF。其基本思想是:数据与 面重复的记录重复位置及长度，不重复的记录原始数据。

压缩过后的数据可以分成多个片段，每个片段有两个部分:解释字段和数据字段。quicklistLZF的结构体如下:

```c
typedef struct quicklistLZF {
    unsigned int sz; // LZF压缩后占用的字节数 
    char compressed[]; // 柔性数组，指向数据部分
} quicklistLZF;
```

**应用场景**

列表(List)的底层实现、发布与订阅、慢查询、监视器等功能。

## 流对象

stream主要由:消息、生产者、消费者和消费组构成。

![](https://secure2.wostatic.cn/static/bgQnAKjxq4Ainb4N9T1RfJ/image.png?auth_key=1716829458-noNNagzKpsPXNaE6EUFAp1-0-6ed6c8e2ed6c36ca282f6414ce64331d)

Redis Stream的底层主要使用了listpack(紧凑列表)和Rax树(基数树)。

#### listpack

listpack表示一个字符串列表的序列化，listpack可用于存储字符串或整数。用于存储stream的消息内容。

结构如下图:

![](https://secure2.wostatic.cn/static/4ghhpvQ37Cj4cAssyMPj8U/image.png?auth_key=1716829459-9FgntWBv9sqNLL94Kxnqu8-0-abb9b23fd99ef4ea9a98e97229f80016)

#### Rax树

Rax 是一个有序字典树 (基数树 Radix Tree)，按照 key 的字典序排列，支持快速地定位、插入和删除操作。

![](https://secure2.wostatic.cn/static/duFXVnEb4myaZyTusMWS5X/image.png?auth_key=1716829459-fQTjJpJMFBidbP9fPfyt3Z-0-2f4fddcdcfabaf033187d4b6f1a88f17)

Rax 被用在 Redis Stream 结构里面用于存储消息队列，在 Stream 里面消息 ID 的前缀是时间戳 + 序号，这样的消息可以理解为时间序列消息。使用 Rax 结构进行存储就可以快速地根据消息 ID 定位到具体的消息，然后继续遍历指定消息之后的所有消息。

![](https://secure2.wostatic.cn/static/n4jWSh3gtZmU16hNfkuC2D/image.png?auth_key=1716829460-h49ESqCtFMAWgXjp2GFw96-0-72902e2600725b8640d8fcd769cee873)

应用场景: stream的底层实现

## 10种encoding

encoding 表示对象的内部编码，占 4 位。

Redis通过 encoding 属性为对象设置不同的编码，对于少的和小的数据，Redis采用小的和压缩的存储方式，体现Redis的灵活性，大大提高了 Redis 的存储量和执行效率

比如Set对象:

intset : 元素是64位以内的整数 hashtable:元素是64位以外的整数 如下所示:

```bash
127.0.0.1:6379> sadd set:001 1  3 5 6 2
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
127.0.0.1:6379> sadd set:004 1 100000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
```

### **String**

int、raw、embstr

- int
    
    REDIS_ENCODING_INT(int类型的整数)
    

```bash
127.0.0.1:6379> set n1 123
OK
127.0.0.1:6379> object encoding n1
"int"
```

- embstr
    
    REDIS_ENCODING_EMBSTR(编码的简单动态字符串) 小字符串 长度小于44个字节
    

```bash
127.0.0.1:6379> set name:001 zhangfei
OK
127.0.0.1:6379> object encoding name:001
"embstr"
```

- raw
    
    REDIS_ENCODING_RAW (简单动态字符串) 大字符串 长度大于44个字节
    

```bash
127.0.0.1:6379> set address:001
asdasdasdasdasdasdsadasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdas
dasdasdas
OK
127.0.0.1:6379> object encoding address:001
"raw"
```

### **list**

列表的编码是quicklist。 REDIS_ENCODING_QUICKLIST(快速列表)

```bash
127.0.0.1:6379> lpush list:001 1 2 5 4 3
(integer) 5
127.0.0.1:6379> object encoding list:001
"quicklist"
```

### **hash**

散列的编码是字典和压缩列表

- dict
    
    REDIS_ENCODING_HT(字典) 当散列表元素的个数比较多或元素不是小整数或短字符串时。
    

```bash
127.0.0.1:6379>  hmset user:003
username111111111111111111111111111111111111111111111111111111111111111111111111
11111111111111111111111111111111  zhangfei password 111 num
2300000000000000000000000000000000000000000000000000
OK
127.0.0.1:6379> object encoding user:003
"hashtable"
```

- ziplist
    
    REDIS_ENCODING_ZIPLIST(压缩列表) 当散列表元素的个数比较少，且元素都是小整数或短字符串时。
    

```bash
127.0.0.1:6379> hmset user:001  username zhangfei password 111 age 23 sex M
OK
127.0.0.1:6379> object encoding user:001
"ziplist"
```

### **set**

集合的编码是整形集合和字典

- intset
    
    REDIS_ENCODING_INTSET(整数集合) 当Redis集合类型的元素都是整数并且都处在64位有符号整数范围内(<18446744073709551616)
    

```bash
127.0.0.1:6379> sadd set:001 1  3 5 6 2
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
```

- dict
    
    REDIS_ENCODING_HT(字典) 当Redis集合类型的元素都是整数并且都处在64位有符号整数范围外(>18446744073709551616)
    

```bash
127.0.0.1:6379> sadd set:004 1 100000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
```

### **zset**

有序集合的编码是压缩列表和跳跃表+字典

- ziplist
    
    REDIS_ENCODING_ZIPLIST(压缩列表) 当元素的个数比较少，且元素都是小整数或短字符串时。
    

```bash
127.0.0.1:6379> zadd hit:1 100 item1 20 item2 45 item3
(integer) 3
127.0.0.1:6379> object encoding hit:1
"ziplist"
```

- skiplist + dict
    
    REDIS_ENCODING_SKIPLIST(跳跃表+字典) 当元素的个数比较多或元素不是小整数或短字符串时。
    

```bash
127.0.0.1:6379>  zadd hit:2 100
item1111111111111111111111111111111111111111111111111111111111111111111111111111
1111111111111111111111111111111111 20 item2 45 item3
(integer) 3
127.0.0.1:6379>  object encoding hit:2
"skiplist"
```

# 缓存过期和淘汰策略

Redis性能高:

- 读:110000次/s
    
- 写:81000次/s 长期使用，key会不断增加，Redis作为缓存使用，物理内存也会满 内存与硬盘交换(swap) 虚拟内存 ，频繁IO 性能急剧下降
    

## maxmemory

### 不设置的场景

Redis的key是固定的，不会增加

Redis作为DB使用，保证数据的完整性，不能淘汰 ， 可以做集群，横向扩展

缓存淘汰策略:禁止驱逐 (默认)

127.0.0.1:6379> sadd set:004 1 100000000000000000000000000 9999999999 (integer) 3 127.0.0.1:6379> object encoding set:004 "hashtable"

### 设置的场景

Redis是作为缓存使用，不断增加Key

maxmemory : 默认为0 不限制

问题:达到物理内存后性能急剧下架，甚至崩溃内存与硬盘交换(swap) 虚拟内存 ，频繁IO 性能急剧下降 设置多少?

与业务有关

1个Redis实例，保证系统运行 1 G ，剩下的就都可以设置Redis

物理内存的3/4

slaver : 留出一定的内存

在redis.conf中

```bash
maxmemory   1024mb
```

命令: 获得maxmemory数

```bash
CONFIG GET maxmemory
```

设置maxmemory后，当趋近maxmemory时，通过缓存淘汰策略，从内存中删除对象

不设置maxmemory 无最大内存限制 maxmemory-policy noeviction (禁止驱逐) 不淘汰

设置maxmemory maxmemory-policy 要配置 expire数据结构

在Redis中可以使用expire命令设置一个键的存活时间(ttl: time to live)，过了这段时间，该键就会自动 被删除。

## expire的使用

expire命令的使用方法如下: expire key ttl(单位秒)

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

dict 用来维护一个 Redis 数据库中包含的所有 Key-Value 键值对，expires则用于维护一个 Redis 数据 库中设置了失效时间的键(即key与失效时间的映射)。

当我们使用 expire命令设置一个key的失效时间时，Redis 首先到 dict 这个字典表中查找要设置的key 是否存在，如果存在就将这个key和失效时间添加到 expires 这个字典表。

当我们使用 setex命令向系统插入数据时，Redis 首先将 Key 和 Value 添加到 dict 这个字典表中，然后将 Key 和失效时间添加到 expires 这个字典表中。

简单地总结来说就是，设置了失效时间的key和具体的失效时间全部都维护在 expires 这个字典表中。

## 删除策略

Redis的数据删除有定时删除、惰性删除和主动删除三种方式。 Redis目前采用惰性删除+主动删除的方式。

### 定时删除

在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作。

需要创建定时器，而且消耗CPU，一般不推荐使用。

### 惰性删除

在key被访问时如果发现它已经失效，那么就删除它。

调用expireIfNeeded函数，该函数的意义是：读取数据之前先检查一下它有没有失效，如果失效了就删除它。

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

在redis.conf文件中可以配置主动删除策略,默认是no-enviction(不删除)

```text
maxmemory-policy allkeys-lru
```

- LRU
    
    LRU (Least recently used) 最近最少使用，算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。
    
    最常见的实现是使用一个链表保存缓存数据，详细算法实现如下:
    
    1. 新数据插入到链表头部;
    2. 每当缓存命中(即缓存数据被访问)，则将数据移到链表头部;
    3. 当链表满的时候，将链表尾部的数据丢弃。
    4. 在Java中可以使用LinkHashMap(哈希链表)去实现LRU
    
    让我们以用户信息的需求为例，来演示一下LRU算法的基本思路:
    
    1. 假设我们使用哈希链表来缓存用户信息，目前缓存了4个用户，这4个用户是按照时间顺序依次从链表右端插入的。
        
        ![](https://secure2.wostatic.cn/static/kAAzgSytSd9FjnN6kUmjyJ/image.png)
        
    2. 此时，业务方访问用户5，由于哈希链表中没有用户5的数据，我们从数据库中读取出来，插入到缓存 当中。这时候，链表中最右端是最新访问到的用户5，最左端是最近最少访问的用户1。
        
        ![](https://secure2.wostatic.cn/static/q1LjAeCTu1FSxt3XmAHTg4/image.png)
        
    3. 接下来，业务方访问用户2，哈希链表中存在用户2的数据，我们怎么做呢?我们把用户2从它的前驱 节点和后继节点之间移除，重新插入到链表最右端。这时候，链表中最右端变成了最新访问到的用户 2，最左端仍然是最近最少访问的用户1。
        
        ![](https://secure2.wostatic.cn/static/Pi5UguCmouEXUfUA2WSJS/image.png)
        
    4. 接下来，业务方请求修改用户4的信息。同样道理，我们把用户4从原来的位置移动到链表最右侧，并 把用户信息的值更新。这时候，链表中最右端是最新访问到的用户4，最左端仍然是最近最少访问的用 户1。
        
        ![](https://secure2.wostatic.cn/static/ww6nz3H2ZPWNbH6dfF4tuG/image.png)
        
        ![](https://secure2.wostatic.cn/static/tynTHhL8HWLuSe4xTMsJDR/image.png)
        
    5. 业务访问用户6，用户6在缓存里没有，需要插入到哈希链表。假设这时候缓存容量已经达到上限，必须先删除最近最少访问的数据，那么位于哈希链表最左端的用户1就会被删除掉，然后再把用户6插入到 最右端。
        
        ![](https://secure2.wostatic.cn/static/c4xWiyJg8QVmtApxQy4nWr/image.png)
        
    
    **Redis的LRU 数据淘汰机制 **
    
    在服务器配置中保存了 lru 计数器 server.lrulock，会定时(redis 定时程序 serverCorn())更新，
    
    server.lrulock 的值是根据 server.unixtime 计算出来的。  
    另外，从 struct redisObject 中可以发现，每一个 redis 对象都会设置相应的 lru。可以想象的是，每一次访问数据的时候，会更新 redisObject.lru。
    
    LRU 数据淘汰机制是这样的：在数据集中随机挑选几个键值对，取出其中 lru 最大的键值对淘汰。
    
    不可能遍历key 用当前时间-最近访问 越大 说明 访问间隔时间越长
    
    volatile-lru
    
    从已设置过期时间的数据集(server.db[i].expires)中挑选最近最少使用的数据淘汰
    
    allkeys-lru
    
    从数据集(server.db[i].dict)中挑选最近最少使用的数据淘汰
    
- LFU
    
    LFU (Least frequently used) 最不经常使用，如果一个数据在最近一段时间内使用次数很少，那么在将来一段时间内被使用的可能性也很小。
    
    volatile-lfu
    
    allkeys-lfu
    
- **random**
    
    随机
    
    volatile-random
    
    从已设置过期时间的数据集(server.db[i].expires)中任意选择数据淘汰  
    allkeys-random  
    从数据集(server.db[i].dict)中任意选择数据淘汰
    
- **ttl  
    **
    
    volatile-ttl
    
    从已设置过期时间的数据集(server.db[i].expires)中挑选将要过期的数据淘汰
    
    redis 数据集数据结构中保存了键值对过期时间的表，即 redisDb.expires。
    
    TTL 数据淘汰机制:从过期时间的表中随机挑选几个键值对，取出其中 ttl 最小的键值对淘汰。
    
- **noenviction **
    
    禁止驱逐数据，不删除 默认
    

### 缓存淘汰策略的选择

- allkeys-lru : 在不确定时一般采用策略。 冷热数据交换
    
- volatile-lru : 比allkeys-lru性能差 存 : 过期时间
    
- allkeys-random : 希望请求符合平均分布(每个元素以相同的概率被访问)
    
- 自己控制:volatile-ttl 缓存穿透
    

#### 案例分享:字典库失效

key-Value 业务表存 code 显示 文字 拉勾早期将字典库，设置了maxmemory，并设置缓存淘汰策略为allkeys-lru 结果造成字典库某些字段失效，缓存击穿 ， DB压力剧增，差点宕机。

分析:

字典库 : Redis做DB使用，要保证数据的完整性 maxmemory设置较小，采用allkeys-lru，会对没有经常访问的字典库随机淘汰 当再次访问时会缓存击穿，请求会打到DB上。

解决方案:

1、不设置maxmemory

2、使用noenviction策略

Redis是作为DB使用的，要保证数据的完整性，所以不能删除数据。 可以将原始数据源(XML)在系统启动时一次性加载到Redis中。 Redis做主从+哨兵 保证高可用

# 事件  
    
## 文件事件  

Redis基于Reactor模式开发了自己的网络事件处理器：这个处理器被称为文件事件处理器（file event handler）
	
- 服务器对套接字操作的抽象  
	- I/O多路复用  

![](redis事件.jpg)  
	
- 套接字  
- I/O多路复用程序  
- 文件事件分派器（dispatcher）  
- 事件处理器  
	- 连接应答处理器  
	- 命令回复处理器  
	- 复制处理器  

## 时间事件  

- 服务器对定时操作的抽象  
	- 定时事件  
	- 周期性事件  
- serverCron函数
# key的过期策略

- 惰性删除  
    >碰到过期键时才进行删除操作  
- 定期删除  
    >每隔一段时间， 主动查找并删除过期键

## 复制功能对过期键的处理  

- 执行 SAVE 命令或者 BGSAVE 命令所产生的新 RDB 文件不会包含已经过期的键。  
- 执行 BGREWRITEAOF 命令所产生的重写 AOF 文件不会包含已经过期的键。  
- 当一个过期键被删除之后， 服务器会追加一条 DEL 命令到现有 AOF 文件的末尾， 显式地删除过期键。  
- 当主服务器删除一个过期键之后， 它会向所有从服务器发送一条 DEL 命令， 显式地删除过期键。  
- 从服务器即使发现过期键， 也不会自作主张地删除它， 而是等待主节点发来 DEL 命令， 这种统一、中心化的过期键删除策略可以保证主从服务器数据的一致性。  

# key的内存淘汰策略
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
	> 基于LRU算法，从所有key中，删除掉最近最少使用的key。该策略是最常使用的策略。  
- volatile-lfu  
	>基于LFU算法，从设置了过期时间的key中，删除掉最不经常使用（使用次数最少）的key。  
- allkeys-lfu  
	>基于LFU算法，从所有key中，删除掉最不经常使用（使用次数最少）的key。

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

> Copy-on-Write 如果有多个调用者同时要求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，知道某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本给该调用者，而其他调用者所见到的最初的资源（之前的快照）仍保持不变

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

![](https://secure2.wostatic.cn/static/kSa8QGVPW57EgfzY73TShu/image.png?auth_key=1716829367-mpjFnXkemnbFFNVQJHAzzn-0-f0c006ece34ad3eb9e04aed52dc1bb84)

1. Redis父进程首先判断：当前是否在执行save或bgsave/bgrewriteaof(aof文件重写命令)的子进程，如果在执行则bgsave命令直接返回。
2. 父进程执行fork(调用OS函数复制主进程)操作创建子进程，这个复制过程中父进程是阻塞的， Redis不能执行来自客户端的任何命令。
3. 父进程fork后，bgsave命令返回”Background saving started”信息并不再阻塞父进程，并可以响应其他命令。
4. 子进程创建RDB文件，根据父进程内存快照生成临时快照文件，完成后对原有文件进行原子替换。 (RDB始终完整)
5. 子进程发送信号给父进程表示完成，父进程更新统计信息。
6. 父进程fork子进程后，继续工作。
> 通过fork子进程的方式生成新的RDB文件，替换旧的文件。fork系统调用是阻塞的，fork完成后不会再阻塞主进程。

### RDB文件结构

[RDB文件结构](Redis-extend#RDB文件结构)

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

在这种模式中， SAVE 原则上每隔一秒钟就会执行一次， 因为 SAVE 操作是由后台子线程(fork)调用 的， 所以它不会引起服务器主进程阻塞。

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

![](https://secure2.wostatic.cn/static/cif5fBL61hvKEt2tzBA1A3/image.png?auth_key=1716829368-dqjE3xBCSfcd5bTTWvk5HN-0-dcad9190386d5756e0e665d477542678)

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

以上就是 AOF 后台重写， 也即是 BGREWRITEAOF 命令(AOF重写)的工作原理。

![](https://secure2.wostatic.cn/static/q4wcZbfJZNfCJqztBq3YiQ/image.png?auth_key=1716829368-iKWjJtzW8z7ARcq4uzU7Z1-0-111427a41aa840f14dd2b8ee8854a85f)

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

![](https://secure2.wostatic.cn/static/344WjEP8kv4wGSXVGqnYgi/image.png?auth_key=1716829368-bdgG47JRgR6hpTexZW2kDt-0-828c6f440dce7468a81385e165e547dd)

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

![](https://secure2.wostatic.cn/static/nWDPoyBbEnj9mC9cVp2HEC/image.png?auth_key=1716829368-tfqkYkiJmGWgyDGkECzkJm-0-ccbb5458dcb508f50a75777ce463a3fd)

## RDB与AOF对比

1. RDB保存某个时刻的数据快照，采用二进制压缩存储，AOF保存操作命令，采用文本存储(混合)
2. RDB性能高、AOF性能较低
3. RDB在配置触发状态会丢失最后一次快照以后更改的所有数据，AOF设置为每秒保存一次，则最多丢2秒的数据
4. Redis以主服务器模式运行，RDB不会保存过期键值对数据，Redis以从服务器模式运行，RDB会保存过期键值对，当主服务器向从服务器同步时，再清空过期键值对。

AOF写入文件时，对过期的key会追加一条del命令，当执行AOF重写时，会忽略过期key和del命令。

![](https://secure2.wostatic.cn/static/nXSwhz1ve9svvtfmrujqBu/image.png?auth_key=1716829368-7jz8oUFr13CwN39qktFAW9-0-6e7eecb41809a3470798c88f807902a2)

## 应用场景

- 内存数据库
  >RDB + AOF，数据不容易丢
- 有原始数据源
  >每次启动时都从原始数据源中初始化 ，则不用开启持久化 (数据量较小)
- 缓存服务器
  >一般RDB，性能高

# 事务  

## 命令  
- watch  
- multi  
- execute  
## acid特性  

### 原子性  
- 但即使错误不会回滚，事务没有原子性  

### 持久性  
- RDB和AOF  
### 隔离性  
- 单线程  
### 一致性  
- 错误会继续执行完，没有一致性

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

## 分片  
- CRC16(KEY) & 16384  
- 重新分片  
- 故障转移

Redis的集群原理

如何从海量数据里快速找到所需？

- 分片：按照某种规则去划分数据，分散存储在多个节点上

一致性哈希算法：对2^32取模，将哈希值空间组织成虚拟的圆环

在节点较少的时候，会出现数据倾斜的问题，某一个节点的压力可能很大，解决这个问题，可以通过对节点求多个hash，构造多个虚拟节点，当数据打到虚拟节点是，我们再映射到，原来的节点上即可

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

# Redis集群介绍

## **为什么需要Redis集群？**

在讲Redis集群架构之前，我们先简单讲下Redis单实例的架构，从最开始的一主N从，到读写分离，再到Sentinel哨兵机制，单实例的Redis缓存足以应对大多数的使用场景，也能实现主从故障迁移。

![](http://images.12345.okgoes.com/blog/images/2021/2/17/114618/9b8fb43c96f752bbaee18f99c59509ed.png)

但是，在某些场景下，单实例存Redis缓存会存在的几个问题：

（1）写并发：

Redis单实例读写分离可以解决读操作的负载均衡，但对于写操作，仍然是全部落在了master节点上面，在海量数据高并发场景，一个节点写数据容易出现瓶颈，造成master节点的压力上升。

（2）海量数据的存储压力：

单实例Redis本质上只有一台Master作为存储，如果面对海量数据的存储，一台Redis的服务器就应付不过来了，而且数据量太大意味着持久化成本高，严重时可能会阻塞服务器，造成服务请求成功率下降，降低服务的稳定性。

针对以上的问题，Redis集群提供了较为完善的方案，解决了存储能力受到单机限制，写操作无法负载均衡的问题。

## **什么是Redis集群？**

Redis3.0加入了Redis的集群模式，实现了数据的分布式存储，对数据进行分片，将不同的数据存储在不同的master节点上面，从而解决了海量数据的存储问题。

Redis集群采用去中心化的思想，没有中心节点的说法，对于客户端来说，整个集群可以看成一个整体，可以连接任意一个节点进行操作，就像操作单一Redis实例一样，不需要任何代理中间件，当客户端操作的key没有分配到该node上时，Redis会返回转向指令，指向正确的node。

Redis也内置了高可用机制，支持N个master节点，每个master节点都可以挂载多个slave节点，当master节点挂掉时，集群会提升它的某个slave节点作为新的master节点。

![](http://images.12345.okgoes.com/blog/images/2021/2/17/179048/da559cf66bf39b98d52cb7d4fdde3b7a.png)

如上图所示，Redis集群可以看成多个主从架构组合起来的，每一个主从架构可以看成一个节点（其中，只有master节点具有处理请求的能力，slave节点主要是用于节点的高可用）

# Redis集群的数据分布算法：哈希槽算法

## **什么是哈希槽算法？**

前面讲到，Redis集群通过分布式存储的方式解决了单节点的海量数据存储的问题，对于分布式存储，需要考虑的重点就是如何将数据进行拆分到不同的Redis服务器上。常见的分区算法有hash算法、一致性hash算法，关于这些算法这里就不多介绍。

- 普通hash算法：将key使用hash算法计算之后，按照节点数量来取余，即hash(key)%N。优点就是比较简单，但是扩容或者摘除节点时需要重新根据映射关系计算，会导致数据重新迁移。
- 一致性hash算法：为每一个节点分配一个token，构成一个哈希环；查找时先根据key计算hash值，然后顺时针找到第一个大于等于该哈希值的token节点。优点是在加入和删除节点时只影响相邻的两个节点，缺点是加减节点会造成部分数据无法命中，所以一般用于缓存，而且用于节点量大的情况下，扩容一般增加一倍节点保障数据负载均衡。

Redis集群采用的算法是哈希槽分区算法。Redis集群中有16384个哈希槽（槽的范围是 0 -16383，哈希槽），将不同的哈希槽分布在不同的Redis节点上面进行管理，也就是说每个Redis节点只负责一部分的哈希槽。在对数据进行操作的时候，集群会对使用CRC16算法对key进行计算并对16384取模（slot = CRC16(key)%16383），得到的结果就是 Key-Value 所放入的槽，通过这个值，去找到对应的槽所对应的Redis节点，然后直接到这个对应的节点上进行存取操作。

使用哈希槽的好处就在于可以方便的添加或者移除节点，并且无论是添加删除或者修改某一个节点，都不会造成集群不可用的状态。当需要增加节点时，只需要把其他节点的某些哈希槽挪到新节点就可以了；当需要移除节点时，只需要把移除节点上的哈希槽挪到其他节点就行了；哈希槽数据分区算法具有以下几种特点：

- 解耦数据和节点之间的关系，简化了扩容和收缩难度；
- 节点自身维护槽的映射关系，不需要客户端代理服务维护槽分区元数据
- 支持节点、槽、键之间的映射查询，用于数据路由，在线伸缩等场景

> 槽的迁移与指派命令：CLUSTER ADDSLOTS 0 1 2 3 4 ... 5000

默认情况下，redis集群的读和写都是到master上去执行的，不支持slave节点读和写，跟Redis主从复制下读写分离不一样，因为redis集群的核心的理念，主要是使用slave做数据的热备，以及master故障时的主备切换，实现高可用的。Redis的读写分离，是为了横向任意扩展slave节点去支撑更大的读吞吐量。而redis集群架构下，本身master就是可以任意扩展的，如果想要支撑更大的读或写的吞吐量，都可以直接对master进行横向扩展。

## **Redis中哈希槽相关的数据结构：**

（1）clusterNode数据结构：保存节点的当前状态，比如节点的创建时间，节点的名字，节点当前的配置纪元，节点的IP和地址，等等。

![](http://images.12345.okgoes.com/blog/images/2021/2/17/199483/20210214012856729.png)

（2）clusterState数据结构：记录当前节点所认为的集群目前所处的状态。

![](http://images.12345.okgoes.com/blog/images/2021/2/17/130898/20210214013142733.png)

（3）节点的槽指派信息：

clusterNode数据结构的slots属性和numslot属性记录了节点负责处理那些槽：

slots属性是一个二进制位数组(bit array)，这个数组的长度为16384/8=2048个字节，共包含16384个二进制位。Master节点用bit来标识对于某个槽自己是否拥有，时间复杂度为O(1)

![](http://images.12345.okgoes.com/blog/images/2021/2/17/160410/20210214015506450.png)

（4）集群所有槽的指派信息：

当收到集群中其他节点发送的信息时，通过将节点槽的指派信息保存在本地的clusterState.slots数组里面，程序要检查槽i是否已经被指派，又或者取得负责处理槽i的节点，只需要访问clusterState.slots[i]的值即可，时间复杂度仅为O(1)

![](http://images.12345.okgoes.com/blog/images/2021/2/17/189954/20210214015537608.png)

如上图所示，ClusterState 中保存的 Slots 数组中每个下标对应一个槽，每个槽信息中对应一个 clusterNode 也就是缓存的节点。这些节点会对应一个实际存在的 Redis 缓存服务，包括 IP 和 Port 的信息。Redis Cluster 的通讯机制实际上保证了每个节点都有其他节点和槽数据的对应关系。无论Redis 的客户端访问集群中的哪个节点都可以路由到对应的节点上，因为每个节点都有一份 ClusterState，它记录了所有槽和节点的对应关系。

## **集群的请求重定向：**

前面讲到，Redis集群在客户端层面没有采用代理，并且无论Redis 的客户端访问集群中的哪个节点都可以路由到对应的节点上，下面来看看 Redis 客户端是如何通过路由来调用缓存节点的：

（1）MOVED请求：

![](http://images.12345.okgoes.com/blog/images/2021/2/17/192662/20210215024932122.png)

如上图所示，Redis 客户端通过 CRC16(key)%16383 计算出 Slot 的值，发现需要找“缓存节点1”进行数据操作，但是由于缓存数据迁移或者其他原因导致这个对应的 Slot 的数据被迁移到了“缓存节点2”上面。那么这个时候 Redis 客户端就无法从“缓存节点1”中获取数据了。但是由于“缓存节点1”中保存了所有集群中缓存节点的信息，因此它知道这个 Slot 的数据在“缓存节点2”中保存，因此向 Redis 客户端发送了一个 MOVED 的重定向请求。这个请求告诉其应该访问的“缓存节点2”的地址。Redis 客户端拿到这个地址，继续访问“缓存节点2”并且拿到数据。

（2）ASK请求：

上面的例子说明了，数据 Slot 从“缓存节点1”已经迁移到“缓存节点2”了，那么客户端可以直接找“缓存节点2”要数据。那么如果两个缓存节点正在做节点的数据迁移，此时客户端请求会如何处理呢？

![](http://images.12345.okgoes.com/blog/images/2021/2/17/174484/20210215025025121.png)

Redis 客户端向“缓存节点1”发出请求，此时“缓存节点1”正向“缓存节点 2”迁移数据，如果没有命中对应的 Slot，它会返回客户端一个 ASK 重定向请求并且告诉“缓存节点2”的地址。客户端向“缓存节点2”发送 Asking 命令，询问需要的数据是否在“缓存节点2”上，“缓存节点2”接到消息以后返回数据是否存在的结果。

（3）频繁重定向造成的网络开销的处理：smart客户端

① 什么是 smart客户端：

在大部分情况下，可能都会出现一次请求重定向才能找到正确的节点，这个重定向过程显然会增加集群的网络负担和单次请求耗时。所以大部分的客户端都是smart的。所谓 smart客户端，就是指客户端本地维护一份hashslot => node的映射表缓存，大部分情况下，直接走本地缓存就可以找到hashslot => node，不需要通过节点进行moved重定向，

② JedisCluster的工作原理：

- 在JedisCluster初始化的时候，就会随机选择一个node，初始化hashslot => node映射表，同时为每个节点创建一个JedisPool连接池。
- 每次基于JedisCluster执行操作时，首先会在本地计算key的hashslot，然后在本地映射表找到对应的节点node。
- 如果那个node正好还是持有那个hashslot，那么就ok；如果进行了reshard操作，可能hashslot已经不在那个node上了，就会返回moved。
- 如果JedisCluter API发现对应的节点返回moved，那么利用该节点返回的元数据，更新本地的hashslot => node映射表缓存
- 重复上面几个步骤，直到找到对应的节点，如果重试超过5次，那么就报错JedisClusterMaxRedirectionException

③ hashslot迁移和ask重定向：

如果hashslot正在迁移，那么会返回ask重定向给客户端。客户端接收到ask重定向之后，会重新定位到目标节点去执行，但是因为ask发生在hashslot迁移过程中，所以JedisCluster API收到ask是不会更新hashslot本地缓存。

虽然ASK与MOVED都是对客户端的重定向控制，但是有本质区别。ASK重定向说明集群正在进行slot数据迁移，客户端无法知道迁移什么时候完成，因此只能是临时性的重定向，客户端不会更新slots缓存。但是MOVED重定向说明键对应的槽已经明确指定到新的节点，客户端需要更新slots缓存。

# Redis集群中节点的通信机制：goosip协议

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
- 16379端口号是用来进行节点间通信的，也就是 cluster bus 的东西，cluster bus 的通信，用来进行故障检测、配置更新、故障转移授权。cluster bus 用的是一种叫gossip 协议的二进制协议

**1、gossip协议的常见类型：**

gossip协议常见的消息类型包含： ping、pong、meet、fail等等。

（1）meet：主要用于通知新节点加入到集群中，通过「cluster meet ip port」命令，已有集群的节点会向新的节点发送邀请，加入现有集群。

（2）ping：用于交换节点的元数据。每个节点每秒会向集群中其他节点发送 ping 消息，消息中封装了自身节点状态还有其他部分节点的状态数据，也包括自身所管理的槽信息等等。

- 因为发送ping命令时要携带一些元数据，如果很频繁，可能会加重网络负担。因此，一般每个节点每秒会执行 10 次 ping，每次会选择 5 个最久没有通信的其它节点。
- 如果发现某个节点通信延时达到了 cluster_node_timeout / 2，那么立即发送 ping，避免数据交换延时过长导致信息严重滞后。比如说，两个节点之间都 10 分钟没有交换数据了，那么整个集群处于严重的元数据不一致的情况，就会有问题。所以 cluster_node_timeout 可以调节，如果调得比较大，那么会降低 ping 的频率。
- 每次 ping，会带上自己节点的信息，还有就是带上 1/10 其它节点的信息，发送出去，进行交换。至少包含 3 个其它节点的信息，最多包含 （总节点数 - 2）个其它节点的信息。

（3）pong：ping和meet消息的响应，同样包含了自身节点的状态和集群元数据信息。

（4）fail：某个节点判断另一个节点 fail 之后，向集群所有节点广播该节点挂掉的消息，其他节点收到消息后标记已下线。

由于Redis集群的去中心化以及gossip通信机制，Redis集群中的节点只能保证最终一致性。例如当加入新节点时(meet)，只有邀请节点和被邀请节点知道这件事，其余节点要等待 ping 消息一层一层扩散。除了 Fail 是立即全网通知的，其他诸如新节点、节点重上线、从节点选举成为主节点、槽变化等，都需要等待被通知到，也就是Gossip协议是最终一致性的协议。

**2、meet命令的实现：**

![](http://images.12345.okgoes.com/blog/images/2021/2/17/162091/20210215020743116.png)

（1）节点A会为节点B创建一个clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面。

（2）节点A根据CLUSTER MEET命令给定的IP地址和端口号，向节点B发送一条MEET消息。

（3）节点B接收到节点A发送的MEET消息，节点B会为节点A创建一个clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面。

（4）节点B向节点A返回一条PONG消息。

（5）节点A将受到节点B返回的PONG消息，通过这条PONG消息，节点A可以知道节点B已经成功的接收了自己发送的MEET消息。

（6）之后，节点A将向节点B返回一条PING消息。

（7）节点B将接收到的节点A返回的PING消息，通过这条PING消息节点B可以知道节点A已经成功的接收到了自己返回的PONG消息，握手完成。

（8）之后，节点A会将节点B的信息通过Gossip协议传播给集群中的其他节点，让其他节点也与节点B进行握手，最终，经过一段时间后，节点B会被集群中的所有节点认识。

# 集群的扩容与收缩：

作为分布式部署的缓存节点总会遇到缓存扩容和缓存故障的问题。这就会导致缓存节点的上线和下线的问题。由于每个节点中保存着槽数据，因此当缓存节点数出现变动时，这些槽数据会根据对应的虚拟槽算法被迁移到其他的缓存节点上。所以对于redis集群，集群伸缩主要在于槽和数据在节点之间移动。

**1、扩容：**

- （1）启动新节点
- （2）使用cluster meet命令将新节点加入到集群
- （3）迁移槽和数据：添加新节点后，需要将一些槽和数据从旧节点迁移到新节点

![](http://images.12345.okgoes.com/blog/images/2021/2/17/127197/20210215042618721.png)

如上图所示，集群中本来存在“缓存节点1”和“缓存节点2”，此时“缓存节点3”上线了并且加入到集群中。此时根据虚拟槽的算法，“缓存节点1”和“缓存节点2”中对应槽的数据会应该新节点的加入被迁移到“缓存节点3”上面。

新节点加入到集群的时候，作为孤儿节点是没有和其他节点进行通讯的。因此需要在集群中任意节点执行 cluster meet 命令让新节点加入进来。假设新节点是 192.168.1.1 5002，老节点是 192.168.1.1 5003，那么运行以下命令将新节点加入到集群中。

> cluster meet 192.168.1.1 5002

这个是由老节点发起的，有点老成员欢迎新成员加入的意思。新节点刚刚建立没有建立槽对应的数据，也就是说没有缓存任何数据。如果这个节点是主节点，需要对其进行槽数据的扩容；如果这个节点是从节点，就需要同步主节点上的数据。总之就是要同步数据。

![](http://images.12345.okgoes.com/blog/images/2021/2/17/154320/20210215043619589.png)

如上图所示，由客户端发起节点之间的槽数据迁移，数据从源节点往目标节点迁移。

- （1）客户端对目标节点发起准备导入槽数据的命令，让目标节点准备好导入槽数据。这里使用 cluster setslot {slot} importing {sourceNodeId} 命令。
- （2）之后对源节点发起送命令，让源节点准备迁出对应的槽数据。使用命令 cluster setslot {slot} importing {sourceNodeId}。
- （3）此时源节点准备迁移数据了，在迁移之前把要迁移的数据获取出来。通过命令 cluster getkeysinslot {slot} {count}。Count 表示迁移的 Slot 的个数。
- （4）然后在源节点上执行，migrate {targetIP} {targetPort} “” 0 {timeout} keys {keys} 命令，把获取的键通过流水线批量迁移到目标节点。
- （5）重复 3 和 4 两步不断将数据迁移到目标节点。
- （6）完成数据迁移到目标节点以后，通过 cluster setslot {slot} node {targetNodeId} 命令通知对应的槽被分配到目标节点，并且广播这个信息给全网的其他主节点，更新自身的槽节点对应表。

**2、收缩：**

- 迁移槽。
- 忘记节点。通过命令 cluster forget {downNodeId} 通知其他的节点

![](http://images.12345.okgoes.com/blog/images/2021/2/17/165801/20210215042040412.png)

为了安全删除节点，Redis集群只能下线没有负责槽的节点。因此如果要下线有负责槽的master节点，则需要先将它负责的槽迁移到其他节点。迁移的过程也与上线操作类似，不同的是下线的时候需要通知全网的其他节点忘记自己，此时通过命令 cluster forget {downNodeId} 通知其他的节点。

# 集群的故障检测与故障转恢复机制：

**1、集群的故障检测：**

Redis集群的故障检测是基于gossip协议的，集群中的每个节点都会定期地向集群中的其他节点发送PING消息，以此交换各个节点状态信息，检测各个节点状态：在线状态、疑似下线状态PFAIL、已下线状态FAIL。

（1）主观下线（pfail）：当节点A检测到与节点B的通讯时间超过了cluster-node-timeout 的时候，就会更新本地节点状态，把节点B更新为主观下线。

> 主观下线并不能代表某个节点真的下线了，有可能是节点A与节点B之间的网络断开了，但是其他的节点依旧可以和节点B进行通讯。

（2）客观下线：

由于集群内的节点会不断地与其他节点进行通讯，下线信息也会通过 Gossip 消息传遍所有节点，因此集群内的节点会不断收到下线报告。

当半数以上的主节点标记了节点B是主观下线时，便会触发客观下线的流程（该流程只针对主节点，如果是从节点就会忽略）。将主观下线的报告保存到本地的 ClusterNode 的结构fail_reports链表中，并且对主观下线报告的时效性进行检查，如果超过 cluster-node-timeout*2 的时间，就忽略这个报告，否则就记录报告内容，将其标记为客观下线。

接着向集群广播一条主节点B的Fail 消息，所有收到消息的节点都会标记节点B为客观下线。

**2、集群地故障恢复：**

当故障节点下线后，如果是持有槽的主节点则需要在其从节点中找出一个替换它，从而保证高可用。此时下线主节点的所有从节点都担负着恢复义务，这些从节点会定时监测主节点是否进入客观下线状态，如果是，则触发故障恢复流程。故障恢复也就是选举一个节点充当新的master，选举的过程是基于Raft协议选举方式来实现的。

2.1、从节点过滤：

检查每个slave节点与master节点断开连接的时间，如果超过了cluster-node-timeout * cluster-slave-validity-factor，那么就没有资格切换成master

2.2、投票选举：

（1）节点排序：

对通过过滤条件的所有从节点进行排序，按照priority、offset、run id排序，排序越靠前的节点，越优先进行选举。

- priority的值越低，优先级越高
- offset越大，表示从master节点复制的数据越多，选举时间越靠前，优先进行选举
- 如果offset相同，run id越小，优先级越高

（2）更新配置纪元：

每个主节点会去更新配置纪元（clusterNode.configEpoch），这个值是不断增加的整数。这个值记录了每个节点的版本和整个集群的版本。每当发生重要事情的时候（例如：出现新节点，从节点精选）都会增加全局的配置纪元并且赋给相关的主节点，用来记录这个事件。更新这个值目的是，保证所有主节点对这件“大事”保持一致，大家都统一成一个配置纪元，表示大家都知道这个“大事”了。

（3）发起选举：

更新完配置纪元以后，从节点会向集群发起广播选举的消息（CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST），要求所有收到这条消息，并且具有投票权的主节点进行投票。每个从节点在一个纪元中只能发起一次选举。

（4）选举投票：

如果一个主节点具有投票权，并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，表示这个主节点支持从节点成为新的主节点。每个参与选举的从节点都会接收CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，并根据自己收到了多少条这种消息来统计自己获得了多少主节点的支持。

如果超过(N/2 + 1)数量的master节点都投票给了某个从节点，那么选举通过，这个从节点可以切换成master，如果在 cluster-node-timeout*2 的时间内从节点没有获得足够数量的票数，本次选举作废，更新配置纪元，并进行第二轮选举，直到选出新的主节点为止。

> 在第(1)步排序领先的从节点通常会获得更多的票，因为它触发选举的时间更早一些，获得票的机会更大

2.3、替换主节点：

当满足投票条件的从节点被选出来以后，会触发替换主节点的操作。删除原主节点负责的槽数据，把这些槽数据添加到自己节点上，并且广播让其他的节点都知道这件事情，新的主节点诞生了。

（1）被选中的从节点执行SLAVEOF NO ONE命令，使其成为新的主节点

（2）新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己

（3）新的主节点对集群进行广播PONG消息，告知其他节点已经成为新的主节点

（4）新的主节点开始接收和处理槽相关的请求

> 备注：如果集群中某个节点的master和slave节点都宕机了，那么集群就会进入fail状态，因为集群的slot映射不完整。如果集群超过半数以上的master挂掉，无论是否有slave，集群都会进入fail状态。

# Redis集群的搭建：

该部分可以参考这篇文章：[https://juejin.cn/post/6922690589347545102#heading-1](https://juejin.cn/post/6922690589347545102#heading-1)

Redis集群的搭建可以分为以下几个部分：

1、启动节点：将节点以集群模式启动，读取或者生成集群配置文件，此时节点是独立的。

2、节点握手：节点通过gossip协议通信，将独立的节点连成网络，主要使用meet命令。

3、槽指派：将16384个槽位分配给主节点，以达到分片保存数据库键值对的效果。

# Redis集群的运维：

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

（1）数据倾斜：

- 节点和槽分配不均
- 不同槽对应键数量差异过大
- 集合对象包含大量元素
- 内存相关配置不一致

（2）请求倾斜：

合理设计键，热点大集合对象做拆分或者使用hmget代替hgetall避免整体读取

**5、集群读写分离：**

集群模式下读写分离成本比较高，直接扩展主节点数量来提高集群性能是更好的选择。

[Redis主从复制原理](https://www.cnblogs.com/hepingqingfeng/p/7263782.html)

# 其他内容  

## lua
### 命令  
- EVAL  
	- 主从服务器都会执行相同的Lua脚本  
- SCRIPT FLUSH  
	- 主从服务器都会重置自己的Lua环境，并清空自己的脚本字典  
- SCRIPT LOAD  
	- 主从服务器都会载入相同的Lua脚本  
- EVALSHA  
	- 无法命令传播  
		- 原因  
			- 可能主从服务器的lua_scripts字典不相同，主服务器能够执行的，从服务器可能执行不了，出现脚本未找到错误；出现在从服务器开始复制主服务器之前，主服务器已执行过某些脚本的场景  
			- 多个从服务器之间载入Lua脚本的情况也可能各有不同，一个从服务器成功，不代表其他的从服务器也能成功；出现在不同从服务器开始复制主服务器的时间不同的场景  
		- 如何复制  
			- 主服务执行EVALSHA之后，将判断此命令给定的校验和是否存在于repl_scriptcache_dict字典中，来决定传播EVAL还是EVALSHA：  
				- 若存在，则传播EVALSHA；  
				- 若不存在，则转换成等价的EVAL再传播，然后将此校验和添加到主服务器的repl_scriptcache_dict字典中  

# 发布与订阅

Redis提供了发布订阅功能，可以用于消息的传输 Redis的发布订阅机制包括三个部分，publisher，subscriber和Channel

![](https://secure2.wostatic.cn/static/tMiPVF36VqgWRN3UzuE8a/image.png?auth_key=1716829395-dPiexBivqvffJHyytHn5q7-0-b5130cb574b83be3a4fde057eba7b607)

发布者和订阅者都是Redis客户端，Channel则为Redis服务器端。 发布者将消息发送到某个的频道，订阅了这个频道的订阅者就能接收到这条消息。

## 频道/模式的订阅与退订

- subscribe：订阅 `subscribe channel1 channel2 ..`
    
    Redis客户端1订阅频道1和频道2
    

```bash
127.0.0.1:6379> subscribe ch1  ch2
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "ch1"
3) (integer) 1
1) "subscribe"
2) "ch2"
3) (integer) 2
```

- publish：发布消息`publish channel message`
    
    Redis客户端2将消息发布在频道1和频道2上
    

```bash
127.0.0.1:6379> publish ch1 hello
(integer) 1
127.0.0.1:6379> publish ch2 world
(integer) 1
```

```
Redis客户端1接收到频道1和频道2的消息
```

```bash
1) "message"
2) "ch1"
3) "hello"
1) "message"
2) "ch2"
3) "world"
```

- unsubscribe：退订
    
    channel Redis客户端1退订频道1
    

```bash
127.0.0.1:6379> unsubscribe ch1
1) "unsubscribe"
2) "ch1"
3) (integer) 0
```

- psubscribe ：模式匹配 `psubscribe +模式`
    
    Redis客户端1订阅所有以ch开头的频道
    

```bash
127.0.0.1:6379> psubscribe ch*
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "ch*"
3) (integer) 1
```

```
Redis客户端2发布信息在频道5上
```

```bash
127.0.0.1:6379> publish ch5 helloworld
(integer) 1
```

```
Redis客户端1收到频道5的信息
```

```bash
1) "pmessage"
2) "ch*"
3) "ch5"
4) "helloworld"
```

- punsubscribe 退订模式

```bash
127.0.0.1:6379>  punsubscribe ch*
1) "punsubscribe"
2) "ch*"
3) (integer) 0
```

## 发布订阅的机制

订阅某个频道或模式：

- 客户端(client)：
    
    属性为pubsub_channels，该属性表明了该客户端订阅的所有频道
    
    属性为pubsub_patterns，该属性表示该客户端订阅的所有模式
    
- 服务器端(RedisServer)：
    
    属性为pubsub_channels，该服务器端中的所有频道以及订阅了这个频道的客户端
    
    属性为pubsub_patterns，该服务器端中的所有模式和订阅了这些模式的客户端
    

```c
typedef struct redisClient {
    ...
    dict *pubsub_channels; //该client订阅的channels，以channel为key用dict的方式组织 
    list *pubsub_patterns; //该client订阅的pattern，以list的方式组织
    ...
} redisClient;

struct redisServer {
    ...
    dict *pubsub_channels;  //redis server进程中维护的channel dict，它以channel 为key，订阅channel的client list为value
    //redis server进程中维护的pattern list
    list *pubsub_patterns;
    int notify_keyspace_events;
    ...
};
```

当客户端向某个频道发送消息时，Redis首先在redisServer中的pubsub_channels中找出键为该频道的结点，遍历该结点的值，即遍历订阅了该频道的所有客户端，将消息发送给这些客户端。

然后，遍历结构体redisServer中的pubsub_patterns，找出包含该频道的模式的结点，将消息发送给订阅了该模式的客户端。

## 使用场景

在Redis哨兵模式中，哨兵通过发布与订阅的方式与Redis主服务器和Redis从服务器进行通信。这个我们将在后面的章节中详细讲解。

Redisson是一个分布式锁框架，在Redisson分布式锁释放的时候，是使用发布与订阅的方式通知的， 这个我们将在后面的章节中详细讲解。

# 事务

所谓事务(Transaction) ，是指作为单个逻辑工作单元执行的一系列操作

## ACID回顾

- Atomicity(原子性)：构成事务的的所有操作必须是一个逻辑单元，要么全部执行，要么全部不执行。
- Consistency(一致性)：数据库在事务执行前后状态都必须是稳定的或者是一致的。
- Isolation(隔离性)：事务之间不会相互影响。
- Durability(持久性)：事务执行成功后必须全部写入磁盘。

## Redis事务

- Redis的事务是通过multi、exec、discard和watch这四个命令来完成的。
- Redis的单个命令都是原子性的，所以这里需要确保事务性的对象是命令集合。
- Redis将命令集合序列化并确保处于同一事务的命令集合连续且不被打断的执行
- Redis不支持回滚操作

## 事务命令

- multi：用于标记事务块的开始,Redis会将后续的命令逐个放入队列中，然后使用exec原子化地执行这个命令队列
- exec：执行命令队列
- discard：清除命令队列
- watch：监视key
- unwatch：清除监视key

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

# Lua脚本

lua是一种轻量小巧的**脚本语言**，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。

Lua应用场景：游戏开发、独立应用脚本、Web应用脚本、扩展和数据库插件。

OpenRestry一个可伸缩的基于Nginx的Web平台，是在nginx之上集成了lua模块的第三方服务器

OpenResty是一个通过Lua扩展Nginx实现的可伸缩的Web平台，内部集成了大量精良的Lua库、第三方 模块以及大多数的依赖项。 用于方便地搭建能够处理超高并发(日活千万级别)、扩展性极高的动态Web应用、Web服务和动态网 关。 功能和nginx类似，就是由于支持lua动态脚本，所以更加灵活，可以实现鉴权、限流、分流、日志记 录、灰度发布等功能。 OpenResty通过Lua脚本扩展nginx功能，可提供负载均衡、请求路由、安全认证、服务鉴权、流量控制与日志监控等服务。

类似的还有Kong(Api Gateway)、tengine(阿里)

## 创建并修改lua环境

- 下载
    
    地址：[http://www.lua.org/download.html](http://www.lua.org/download.html) 可以本地下载上传到linux，也可以使用curl命令在linux系统中进行在线下载
    

```bash
curl -R -O http://www.lua.org/ftp/lua-5.3.5.tar.gz
```

- 安装

```bash
yum -y install readline-devel ncurses-devel tar -zxvf lua-5.3.5.tar.gz
#在src目录下
make linux
#或
make install
```

```
如果报错，说找不到readline/readline.h, 可以通过yum命令安装
```

```bash
yum -y install readline-devel ncurses-devel
```

```
安装完以后再
```

```bash
make linux  / make install
```

```
最后，直接输入 lua命令即可进入lua的控制台 
```

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

- script参数：是一段Lua脚本程序，它会被运行在Redis服务器上下文中，这段脚本不必(也不应该)定义为一个Lua函数。
- numkeys参数：用于指定键名参数的个数。
- key [key ...]参数：从EVAL的第三个参数开始算起，使用了numkeys个键(key)，表示在脚本中 所用到的那些Redis键(key)，这些键名参数可以在Lua中通过全局变量KEYS数组，用1为基址的形 式访问( KEYS[1] ， KEYS[2] ，以此类推)。
- arg [arg ...]参数：可以在Lua中通过全局变量ARGV数组访问，访问的形式和KEYS变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。

```bash
eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
```

### lua脚本中调用Redis命令

- redis.call()：
    - 返回值就是redis命令执行的返回值
    - 如果出错，则返回错误信息，不继续执行
- redis.pcall()：
    - 返回值就是redis命令执行的返回值
    - 如果出错，则记录错误信息，继续执行
- 注意事项
    - 在脚本中，使用return语句将返回值返回给客户端，如果没有return，则返回nil

```bash
eval "return redis.call('set',KEYS[1],ARGV[1])" 1 n1 zhaoyun
```

### EVALSHA

EVAL 命令要求你在每次执行脚本的时候都发送一次脚本主体(script body)。

Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本，不过在很多场合，付出无谓的带宽来传送脚本主体并不是最佳选择。

为了减少带宽的消耗， Redis 实现了 EVALSHA 命令，它的作用和 EVAL 一样，都用于对脚本求值，但它接受的第一个参数不是脚本，而是脚本的 SHA1 校验和(sum)

#### SCRIPT命令

- SCRIPT FLUSH：清除所有脚本缓存
- SCRIPT EXISTS：根据给定的脚本校验和，检查指定的脚本是否存在于脚本缓存
- SCRIPT LOAD：将一个脚本装入脚本缓存，返回SHA1摘要，但并不立即运行它

```bash
192.168.24.131:6380> script load "return redis.call('set',KEYS[1],ARGV[1])"
"c686f316aaf1eb01d5a4de1b0b63cd233010e63d"
192.168.24.131:6380> evalsha c686f316aaf1eb01d5a4de1b0b63cd233010e63d 1 n2
zhangfei
OK
192.168.24.131:6380> get n2
```

- SCRIPT KILL：杀死当前正在运行的脚本

## 脚本管理命令实现

使用redis-cli直接执行lua脚本。

test.lua脚本内容

```lua
return redis.call('set',KEYS[1],ARGV[1])

```

```Bash
./redis-cli -h 127.0.0.1 -p 6379 --eval test.lua name:6 , 'caocao' #，两边有空格
```

list.lua

脚本内容

```lua
local key=KEYS[1]
local list=redis.call("lrange",key,0,-1);
return list;

```

```Bash
./redis-cli --eval list.lua list
```

利用Redis整合Lua，主要是为了性能以及事务的原子性。因为redis帮我们提供的事务功能太差。

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

# 慢查询日志

我们都知道MySQL有慢查询日志，Redis也有慢查询日志，可用于监视和优化查询

## 慢查询设置

在redis.conf中可以配置和慢查询日志相关的选项:

```bash
#执行时间超过多少微秒的命令请求会被记录到日志上 0 :全记录 <0 不记录 
slowlog-log-slower-than 10000
#slowlog-max-len 存储慢查询日志条数
slowlog-max-len 128
```

Redis使用列表存储慢查询日志，采用队列方式(FIFO)

config set的方式可以临时设置，redis重启后就无效

- config set slowlog-log-slower-than 微秒
- config set slowlog-max-len 条数

查看日志：slowlog get [n]

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

## 慢查询记录的保存

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

## 慢查询日志的阅览&删除

初始化日志列表

```c
void slowlogInit(void) {
    server.slowlog = listCreate(); /* 创建一个list列表 */ 
    server.slowlog_entry_id = 0;     /* 日志ID从0开始 */ 
    listSetFreeMethod(server.slowlog,slowlogFreeEntry);     /* 指定慢查询日志list空间的释放方法 */ 
}
```

获得慢查询日志记录

slowlog get [n]

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

查看日志数量的 slowlog len

```c
def SLOWLOG_LEN():
    // slowlog 链表的长度就是慢查询日志的条目数量 
    return len(redisServer.slowlog)
```

清除日志 slowlog reset

```C
def SLOWLOG_RESET():
// 遍历服务器中的所有慢查询日志
for log in redisServer.slowlog:
    // 删除日志 
    deleteLog(log)
```

## 添加日志实现

在每次执行命令的之前和之后， 程序都会记录微秒格式的当前 UNIX 时间戳， 这两个时间戳之间的差 就是服务器执行命令所耗费的时长， 服务器会将这个时长作为参数之一传给slowlogPushEntryIfNeeded 函数， 而slowlogPushEntryIfNeeded 函数则负责检查是否需要为这次执行的命令创建慢查询日志

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

slowlogPushEntryIfNeeded 函数的作用有两个:

1. 检查命令的执行时长是否超过 slowlog-log-slower-than 选项所设置的时间， 如果是的话， 就为命令创建一个新的日志， 并将新日志添加到 slowlog 链表的表头。
2. 检查慢查询日志的长度是否超过 slowlog-max-len 选项所设置的长度， 如果是的话， 那么将多出来的日志从 slowlog 链表中删除掉。

## 慢查询定位&处理

使用slowlog get 可以获得执行较慢的redis命令，针对该命令可以进行优化:

1. 尽量使用短的key，对于value有些也可精简，能使用int就int。
2. 避免使用keys *、hgetall等全量操作。
3. 减少大key的存取，打散为小key
4. 将rdb改为aof模式，rdb fork 子进程，数据量过大主进程阻塞，redis大幅下降，关闭持久化 ， (适合于数据量较小) 改aof 命令式
5. 想要一次添加多条数据的时候可以使用管道
6. 尽可能地使用哈希存储
7. 尽量限制下redis使用的内存大小，这样可以避免redis使用swap分区或者出现OOM错误，避免内存与硬盘的swap

# 监视器

Redis客户端通过执行MONITOR命令可以将自己变为一个监视器，实时地接受并打印出服务器当前处理 的命令请求的相关信息。

此时，当其他客户端向服务器发送一条命令请求时，服务器除了会处理这条命令请求之外，还会将这条命令请求的信息发送给所有监视器。

![](https://secure2.wostatic.cn/static/dm9Qm8jkKgvjoVRyqwhem/image.png?auth_key=1716829397-q6rG6ZKQKXGm9LF4Xoni6L-0-5d442ed7814a45ea0dfc5387cae06807)

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

call 主要调用了 replicationFeedMonitors ，这个函数的作用就是将命令打包为协议，发送给监视器。

## Redis监控平台

- **Grafana** 是一个开箱即用的可视化工具，具有功能齐全的度量仪表盘和图形编辑器，有灵活丰富的图形化选项，可以混合多种风格，支持多个数据源特点。
- **Prometheus**是一个开源的服务监控系统，它通过HTTP协议从远程的机器收集数据并存储在本地的时序数据库上。
- r**edis_exporter**为Prometheus提供了redis指标的导出，配合Prometheus以及grafana进行可视化及监控。

![](https://secure2.wostatic.cn/static/qWhSdkueAWmLCCsCi9y16a/image.png?auth_key=1716829400-wLtjguSGireMAu7M9g1RnJ-0-d532266399549d3574fb047aaadf9bd0)

## 热点缓存问题

如果一个缓存设置了过期问题，那么在高并发的环境下就有可能出现热点缓存问题，更有可能导致缓存雪崩。

**解决方式：**

1、使用双重检测锁解决热点缓存问题


## 缓存预热

缓存预热就是系统启动前,提前将相关的缓存数据直接加载到缓存系统。避免在用户请求的时候,先查询数据库,然后再将数据缓存的问题!用户直接查询实现被预热的缓存数据。

加载缓存思路:

- 数据量不大，可以在项目启动的时候自动进行加载
- 利用定时任务刷新缓存，将数据库的数据刷新到缓存中

### 主从不一致的问题？

1、redis的的确是弱一致性，异步的同步

2、锁不能用主从，要用单实例/分片集群/redlock →redisson

3、在配置中提供必须有多少个Client连接能同步，你可以配置同步因子；趋向于强一致性

4、wait 2 0 小心

5、3，4点有点违背redis储中


# 缓存的读写模式

缓存有三种读写模式

## Cache Aside Pattern(常用)

Cache Aside Pattern(旁路缓存)，是最经典的缓存+数据库读写模式。 读的时候，先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应。

![](

更新的时候，先更新数据库，然后再删除缓存。


**为什么是删除缓存，而不是更新缓存呢? **

1. 缓存的值是一个结构：hash、list，更新数据需要遍历
    
    先遍历(耗时)后修改
    
2. 懒加载，使用的时候才更新缓存
    
    使用的时候才从DB中加载 也可以采用异步的方式填充缓存,开启一个线程 定时将DB的数据刷到缓存中
    

**高并发脏读的三种情况 **

1. 先更新数据库，再更新缓存 update与commit之间，更新缓存，commit失败，则DB与缓存数据不一致
    
2. 先删除缓存，再更新数据库 update与commit之间，有新的读，缓存空，读DB数据到缓存，数据是旧的数据，commit后 DB为新数据 ,则DB与缓存数据不一致
    
3. 先更新数据库，再删除缓存(推荐) update与commit之间，有新的读，缓存空，读DB数据到缓存 数据是旧的数据 commit后 DB为新数据，则DB与缓存数据不一致
    
    采用延时双删策略
    

## Read/Write Through Pattern

应用程序只操作缓存，缓存操作数据库。

Read-Through(穿透读模式/直读模式)：应用程序读缓存，缓存没有，由缓存回源到数据库，并写入缓存。(guavacache)

Write-Through(穿透写模式/直写模式)：应用程序写缓存，缓存写数据库。 该种模式需要提供数据库的handler，开发较为复杂。

## Write Behind Caching Pattern

应用程序只更新缓存。 缓存通过异步的方式将数据批量或合并后更新到DB中 不能时时同步，甚至会丢数据

# 数据分区算法

## 概念

分布式数据存储中，数据是分布式在不同的服务器上的，那么每条数据应该存储到哪台服务器？取的时候又应该去哪台服务器去取？分布式数据存储算法就是解决此类问题的算法

## 种类

### hash算法

#### 过程

1. 客户端开始操作数据
2. 服务器对数据的key进行hash计算，得到一个数字
3. 服务器对得到的数字与服务器数量做取余计算，得到服务器的编号
4. 服务器在相应的服务器上进行操作

hash算法的数据存储过程图解：

![](https://secure2.wostatic.cn/static/pKaQRcM9dSsGiusH9e8C2f/image.png?auth_key=1716829063-uoLseP6r16bh3jid2Eff3d-0-b81f5208538928a6f80f83239720147d)

![](https://secure2.wostatic.cn/static/mi99MNwZbL4ufrZpxPA45E/image.png?auth_key=1716829063-6i2zsSHqfpn3J5MmmKmxrM-0-4fa9e0ad0bf980c099595bbbc6ac197f)

#### 缺点和使用现状

可能出现某个服务器上的热点数据特别多，导致该服务器出现性能瓶颈，如果某个服务器出现故障，会导致大部分数据的hash错乱，导致大部分数据读写失效或错误。由于hash算法的上述缺点，现在很少有分布式数据存储使用该算法

### 原始一致性hash算法

#### 过程

1. 将服务器使用Hash函数进行一个哈希（一致性Hash算法将整个Hash空间组织成一个虚拟的圆环，Hash函数的值空间为0 ~ 2^32 - 1(一个32位无符号整型)），分布在一个圆环上，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台服务器就确定在了哈希环的一个位置上。
2. 客户端开始进行数据操作
3. 将数据Key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针查找，遇到的服务器就是其应该定位到的服务器。
4. 用得到的hash值在圆环对应的各个点上去对比，得到数据在圆环上的落点
5. 服务器顺时针寻找距离该落点最近的一个服务器节点
6. 服务器在相应的服务器上进行操作

服务器分布样式图解：

![](https://secure2.wostatic.cn/static/adgHTn4qGystLjDCMQR5Vm/image.png?auth_key=1716829063-vb41Y6ikFt5PjxriKNK9g4-0-c8d8d206c7bbdb14db672eb990315e3d)

#### 一致性hash算法的优点

如果某个服务器节点出现故障，只会导致该节点对应区间的数据出现错误，其他节点的数据仍然保存完整。

#### 一致性hash算法的缺点

因为实体服务器节点少量相比哈希环分片数据很少，这种特性决定了一致性哈希的数据倾斜，由于数量少导致服务节点分布不均，造成机器负载失衡。

## 基于虚拟节点的一致性hash算法

基于虚拟节点的一致性hash算法指的是在一致性hash算法的基础上，在每个虚拟节对应的区间上增加若干个其他节点的虚拟节点，这样就能最大限度的解决热点数据导致的服务器数据分布不均的问题。

基于虚拟节点的一致性hash算法图解：

![](https://secure2.wostatic.cn/static/sMUvZdX2cQCsiSBUJfx2QU/image.png?auth_key=1716829063-oSYhvZsVRkCcemXJ9HagFT-0-6c69ad5e8006abae05a67e6ab668ce04)

## Redis Hash Slot算法

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

# Redis的Scan和Keys命令
# 背景

我们有一个类似用户中心，其中有百万级别用户以user_id + id号为key存放在redis中。有一个需求是将user_为前缀进行匹配查询进行key的匹配，就在进行这个的操作命令的时候出现服务卡顿和redis 有部分链接超时。最后排查出来的问题所在就是keys的时候查出来的key太多导致的问题。具体原因那就从他这个命令的原理看起 最后的解决方案是：使用scan命令

# Keys

## 简介

通过简单的正则就可以进行模糊匹配，没有分页，没有游标。就是暴力查找遍历。 好处就是方便，坏处应有仅有，Redis执行命令是单线程的，那就是如果说我这个线程查询的内容过多，导致查询时间很长就会出现其他线程的阻塞，或者超时的问题。查询的时间复杂度是O（n）

# Scan

## 简介

1. scan 复杂度为O（n）可带游标进行分步进行查询，不会阻塞线程
2. 可以进行模糊匹配和keys一样，只不过每一次都要带上一次返回的游标，可以使用limit限制最大条数，有可能少但是不会超过（[http://doc.redisfans.com/key/scan.html#scan）](http://doc.redisfans.com/key/scan.html#scan%EF%BC%89)
3. 每次根据游标返回的数据有可能为空也有可能为多个。只要返回的游标不为0，就不代表数据没有了。
4. 但是问题还是有的，就是有可能返回重复的key值这个得我们应用程序进行一次去重，可以使用set进行存储或者Map因为他两的数据结构本身就是不可以重复的。

## SCAN 内部探究

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



# [Redisson](https://redisson.org/)

[Redisson](https://redisson.org/)是架设在[Redis](http://redis.cn/)基础上的一个Java驻内存数据网格（In-Memory Data Grid）。简化了分布式环境中程序相互之间的协作。

[https://github.com/redisson/redisson/wiki/目录](https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95)

Redisson是架设在Redis基础上的一个Java驻内存数据网格(In-Memory Data Grid)。 Redisson在基于NIO的Netty框架上，生产环境使用分布式锁。

# 使用

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

# Redisson分布式锁的实现原理

![](https://secure2.wostatic.cn/static/3jDbCrYXVMrD4NgozZKpLT/image.png?auth_key=1716829222-214mhHXJbiagGBSVAQFB3w-0-25dc8675134d764a7bf105441b948c88)

## 加锁机制

如果该客户端面对的是一个redis cluster集群，他首先会根据hash节点选择一台机器。 发送lua脚本到redis服务器上，脚本如下:

```lua
if (redis.call('exists',KEYS[1])==0) 
then 
  --看有没有锁
  redis.call('hset',KEYS[1],ARGV[2],1) ; 
  --无锁 加锁
  redis.call('pexpire',KEYS[1],ARGV[1]) ; 
  return nil; 
end ;
if (redis.call('hexists',KEYS[1],ARGV[2]) ==1 ) 
then 
  --我加的锁 
  redis.call('hincrby',KEYS[1],ARGV[2],1) ; 
  --重入锁 
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
    

如果执行lock.unlock()，就可以释放分布式锁，此时的业务逻辑也是非常简单的。 其实说白了，就是每次都对myLock数据结构中的那个加锁次数减1。 如果发现加锁次数是0了，说明这个客户端已经不再持有锁了，此时就会用: `del myLock`命令，从redis里删除这个key。 然后呢，另外的客户端2就可以尝试完成加锁了。

# 分布式锁特性

## 互斥性

任意时刻，只能有一个客户端获取锁，不能同时有两个客户端获取到锁。

## 同一性

锁只能被持有该锁的客户端删除，不能由其它客户端删除。

## 可重入性

持有某个锁的客户端可继续对该锁加锁，实现锁的续租

## 容错性

锁失效后(超过生命周期)自动释放锁(key失效)，其他客户端可以继续获得该锁，防止死锁。

# 分布式锁的实际应用

- 数据并发竞争
    
    利用分布式锁可以将处理串行化，前面已经讲过了。
    
- 防止库存超卖
    
    ![](https://secure2.wostatic.cn/static/tWkbhYRALuQTzUgvnwgTBg/image.png?auth_key=1716829222-dUFULiuuMhBuzHeeBbFCvt-0-f48aeb43063d9a144c25f3de41142597)
    
    订单1下单前会先查看库存，库存为10，所以下单5本可以成功; 订单2下单前会先查看库存，库存为10，所以下单8本可以成功;
    
    订单1和订单2 同时操作，共下单13本，但库存只有10本，显然库存不够了，这种情况称为库存超卖。 可以采用分布式锁解决这个问题。
    
    ![](https://secure2.wostatic.cn/static/bDwQVtFtG7Tnbch15ziZvA/image.png?auth_key=1716829223-eunPVncBEBgqaC4rVkERZf-0-f633340bfe81139a7c7d3944b2186d68)
    
    订单1和订单2都从Redis中获得分布式锁(setnx)，谁能获得锁谁进行下单操作，这样就把订单系统下单 的顺序串行化了，就不会出现超卖的情况了。伪码如下:
    

```java
//加锁并设置有效期 
if(redis.lock("RDL",200)){
    //判断库存
    if (orderNum<getCount()){ 
        //加锁成功 ,可以下单 
        order(5);
        //释放锁 
        redis,unlock("RDL");
    }
}
```

```
注意此种方法会降低处理效率，这样不适合秒杀的场景，秒杀可以使用CAS和Redis队列的方式。 
```



# 通信协议

Redis是单进程单线程的。 应用系统和Redis通过Redis协议(RESP)进行交互。

# 请求响应模式

![](https://secure2.wostatic.cn/static/b4BxPMEGGxf2nSoLpAaWys/image.png?auth_key=1716829337-6teSD6SdX3WPRTNyUEZUsg-0-7af03c46f607dcd99e9801433de2a1a7)

## 串行的请求响应模式(ping-pong)

串行化是最简单模式，客户端与服务器端建立长连接

连接通过心跳机制检测(ping-pong) ack应答 客户端发送请求，服务端响应，客户端收到响应后，再发起第二个请求，服务器端再响应。

![](https://secure2.wostatic.cn/static/m38aN8PKRsQQFxnFUjT6h4/image.png?auth_key=1716829337-oZbYLCzKpNnVrVDg85i6y7-0-0e843aa8559c441809b68d506e1e75b8)

telnet和redis-cli 发出的命令都属于该种模式

特点:

- 有问有答
- 耗时在网络传输命令
- 性能较低

## 双工的请求响应模式(pipeline)

![](https://secure2.wostatic.cn/static/82CPydxhLH5FV3VGVsP8vF/image.png?auth_key=1716829337-gWxnzh7FKun8ox5PosJyed-0-1ce05866f96174f77aace04e5ca7824d)

- pipeline的作用是将一批命令进行打包，然后发送给服务器，服务器执行完按顺序打包返回。
- 通过pipeline，一次pipeline(n条命令)=一次网络时间 + n次命令时间

通过Jedis可以很方便的使用pipeline

```java
Jedis redis = new Jedis("192.168.1.111", 6379); 
//授权密码 对应redis.conf的requirepass密码 
redis.auth("12345678");
Pipeline pipe = jedis.pipelined();
for (int i = 0; i <50000; i++) {
    pipe.set("key_"+String.valueOf(i),String.valueOf(i));
}
//将封装后的PIPE一次性发给redis 
pipe.sync();
```

## 原子化的批量请求响应模式(事务)

Redis可以利用事务机制批量执行命令。后面会详细讲解。

## 发布订阅模式(pub/sub)

发布订阅模式是：一个客户端触发，多个客户端被动接收，通过服务器中转。后面会详细讲解。

## 脚本化的批量执行(lua)

客户端向服务器端提交一个lua脚本，服务器端执行该脚本。后面会详细讲解。

# 请求数据格式

Redis客户端与服务器交互采用序列化协议(RESP)。 请求以字符串数组的形式来表示要执行命令的参数，Redis使用命令特有(command-specific)数据类型作为回复。

Redis通信协议的主要特点有:

客户端和服务器通过 TCP 连接来进行数据交互， 服务器默认的端口号为 6379 。 客户端和服务器发送的命令或数据一律以 \r\n (CRLF)结尾。

在这个协议中， 所有发送至 Redis 服务器的参数都是二进制安全(binary safe)的。 简单，高效，易读。

## 内联格式

可以使用telnet给Redis发送命令，首字符为Redis命令名的字符，格式为 str1 str2 str3...

```bash
[root@localhost bin]# telnet 127.0.0.1 6379
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
ping
+PONG
exists name
:1
```

## 规范格式(redis-cli) RESP

1. 间隔符号，在Linux下是\r\n，在Windows下是\n
2. 简单字符串 Simple Strings, 以 "+"加号 开头
3. 错误 Errors, 以"-"减号 开头
4. 整数型 Integer， 以 ":" 冒号开头
5. 大字符串类型 Bulk Strings, 以 "$"美元符号开头，长度限制512M
6. 数组类型 Arrays，以 "*"星号开头 用SET命令来举例说明RESP协议的格式。

```bash
redis> SET mykey Hello
"OK"
```

实际发送的请求数据:

```text
*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$5\r\nHello\r\n
*3
$3
SET
$5
mykey
$5
Hello
```

实际收到的响应数据:

```text
+OK\r\n
```

# 命令处理流程

整个流程包括：服务器启动监听、接收命令请求并解析、执行命令请求、返回命令回复等。

![](https://secure2.wostatic.cn/static/b3HricxfzhteUt7JbEb3pE/image.png?auth_key=1716829337-6hZYLYeHMHL43F8yWdydjF-0-a1ca3fade0c2affda3cca88d7d0e879c)

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
    
    ![](https://secure2.wostatic.cn/static/vFYBMTRx6LgsURXGJuyKyB/image.png?auth_key=1716829337-zTcKw7TAZAtEY2yMvoJg7-0-2d8c079668a4283ecf542a67c4911087)
    
    大致分三个部分:
    
    - 调用 lookupCommand 方法获得对应的 redisCommand （是set还是get还是别的命令）
    - 检测当前 Redis 是否可以执行该命令
    - 是不是集群？可能会move
    - 是不是到了最大内存限制，牵扯到缓存淘汰
    - 是不是master，slave不允许写
    - lua和事务判断
    - 调用 call 方法真正执行命令

# 协议响应格式

- 状态回复
    
    对于状态，回复的第一个字节是“+”
    

```text
"+OK"
```

- 错误回复
    
    对于错误，回复的第一个字节是“ - ”
    

```text
1. -ERR unknown command 'foobar'
2. -WRONGTYPE Operation against a key holding the wrong kind of value
```

- 整数回复
    
    对于整数，回复的第一个字节是“:”
    

```text
":6"
```

- 批量回复
    
    对于批量字符串，回复的第一个字节是“$”
    

```text
"$6 foobar"
```

- 多条批量回复
    
    对于多条批量回复(数组)，回复的第一个字节是“*”
    

```text
"*3"
```

# 协议解析及处理

包括协议解析、调用命令、返回结果。

## 协议解析

用户在Redis客户端键入命令后，Redis-cli会把命令转化为RESP协议格式，然后发送给服务器。服务器再对协议进行解析，分为三个步骤

1. 解析命令请求参数数量
    
    命令请求参数数量的协议格式为"*N\r\n" ,其中N就是数量，比如
    

```bash
127.0.0.1:6379> set name:1 zhaoyun
```

```
我们打开aof文件可以看到协议内容
```

```text
*3(/r/n)
$3(/r/n)
set(/r/n)
$7(/r/n)
name:10(/r/n)
$7(/r/n)
zhaoyun(/r/n)
```

```
首字符必须是“*”，使用"\r"定位到行尾，之间的数就是参数数量了。
```

2. 循环解析请求参数
    
    首字符必须是""，使用"/r"定位到行尾，之间的数是参数的长度，从/n后到下一个""之间就是参数的值了
    
    循环解析直到没有"$"。
    

## 协议执行

协议的执行包括命令的调用和返回结果

- 判断参数个数和取出的参数是否一致
    
- RedisServer解析完命令后,会调用函数processCommand处理该命令请求
    
- quit校验，如果是“quit”命令，直接返回并关闭客户端
    
- 命令语法校验，执行lookupCommand，查找命令(set)，如果不存在则返回:“unknown command”错误。
    
- 参数数目校验，参数数目和解析出来的参数个数要匹配，如果不匹配则返回:“wrong number of arguments”错误。
    
- 此外还有权限校验，最大内存校验，集群校验，持久化校验等等。
    
- 校验成功后，会调用call函数执行命令，并记录命令执行时间和调用次数。
    
- 如果执行命令时间过长还要记录慢查询日志。
    
- 执行命令后返回结果的类型不同则协议格式也不同，分为5类:状态回复、错误回复、整数回复、批量 回复、多条批量回复。
    

# 事件处理机制

Redis服务器是典型的事件驱动系统。

Redis将事件分为两大类：文件事件和时间事件。

# 文件事件

文件事件即Socket的读写事件，也就是IO事件。 客户端的连接、命令请求、数据回复、连接断开

### socket

套接字(socket)是一个抽象层，应用程序可以通过它发送或接收数据。

### Reactor

Redis事件处理机制采用单线程的Reactor模式，属于I/O多路复用的一种常见模式。 IO多路复用( I/O multiplexing )指的通过单个线程管理多个Socket。

Reactor pattern(反应器设计模式)是一种为处理并发服务请求，并将请求提交到一个或者多个服务处理程序的事件设计模式。

Reactor模式是事件驱动的有一个或多个并发输入源(文件事件)。有一个Service Handler，有多个Request Handlers。

这个Service Handler会同步的将输入的请求(Event)多路复用的分发给相应的Request Handler

![](https://secure2.wostatic.cn/static/q3svttSYJV8mbnfu9suqkv/image.png?auth_key=1716829338-2N7HtHFTNR6joSBkRiVNR5-0-f4f77fee27dc55e9e923f8ad839efd6d)

![](https://secure2.wostatic.cn/static/qcKmW5eD16mvnMr6mpYQE9/image.png?auth_key=1716829339-iYsGxoCrBtMPV6yyTNSzrS-0-912bc7df82c1d3341ad1d6f20a5e5ac4)

- Handle：I/O操作的基本文件句柄，在linux下就是fd(文件描述符)
- Synchronous Event Demultiplexer：同步事件分离器，阻塞等待Handles中的事件发生。(系统)
- Reactor：事件分派器，负责事件的注册，删除以及对所有注册到事件分派器的事件进行监控， 当事件发生时会调用Event Handler接口来处理事件。
- Event Handler： 事件处理器接口，这里需要Concrete Event Handler来实现该接口
- Concrete Event Handler：真实的事件处理器，通常都是绑定了一个handle，实现对可读事件 进行读 取或对可写事件进行写入的操作。

![](https://secure2.wostatic.cn/static/63SD6imMvRiaf6tqNBiHsK/image.png?auth_key=1716829338-ua7QWVWKoxL4K4BxXVEnAs-0-d336414f5c4ad6b910156d12f16a07cd)

1. 主程序向事件分派器(Reactor)注册要监听的事件
2. Reactor调用OS提供的事件处理分离器，监听事件(wait)
3. 当有事件产生时，Reactor将事件派给相应的处理器来处理 handle_event()

4种IO多路复用模型与选择 `select`，`poll`，`epoll`、`kqueue`都是IO多路复用的机制。

I/O多路复用就是通过一种机制，一个进程可以监视多个描述符(socket)，一旦某个描述符就绪(一 般是读就绪或者写就绪)，能够通知程序进行相应的读写操作。

select、poll、epoll

## 文件事件分派器

在redis中，对于文件事件的处理采用了Reactor模型。采用的是epoll的实现方式。

![](https://secure2.wostatic.cn/static/raoSnsaTgVexEg6Zoj2xGq/image.png?auth_key=1716829338-evUG5MYubrhFCvZ163mxb7-0-e050b016257d8354629143731f27fcd4)

Redis在主循环中统一处理文件事件和时间事件，信号事件则由专门的handler来处理。

主循环

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

## 事件处理器

### 连接处理函数 acceptTCPHandler

当客户端向 Redis 建立 socket时，aeEventLoop 会调用 acceptTcpHandler 处理函数，服务器会为每个链接创建一个 Client 对象，并创建相应文件事件来监听socket的可读事件，并指定事件处理函数。

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

### 请求处理函数 readQueryFromClient

当客户端通过 socket 发送来数据后，Redis 会调用 readQueryFromClient 方法,readQueryFromClient 方法会调用 read 方法从 socket 中读取数据到输入缓冲区中，然后判断其大小是否大于系统设置的 client_max_querybuf_len，如果大于，则向 Redis返回错误信息，并关闭 client。

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

### 命令回复处理器 sendReplyToClient

sendReplyToClient函数是Redis的命令回复处理器，这个处理器负责将服务器执行命令后得到的命令回复通过套接字返回给客户端。

1、将outbuf内容写入到套接字描述符并传输到客户端

2、aeDeleteFileEvent 用于删除文件写事件

# 时间事件

时间事件分为定时事件与周期事件:

一个时间事件主要由以下三个属性组成:

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

## serverCron

时间事件的最主要的应用是在redis服务器需要对自身的资源与配置进行定期的调整，从而确保服务器的长久运行，这些操作由redis.c中的serverCron函数实现。该时间事件主要进行以下操作:

1. 更新redis服务器各类统计信息，包括时间、内存占用、数据库占用等情况。
    
2. 清理数据库中的过期键值对。
    
3. 关闭和清理连接失败的客户端。
    
4. 尝试进行aof和rdb持久化操作。
    
5. 如果服务器是主服务器，会定期将数据向从服务器做同步操作。
    
6. 如果处于集群模式，对集群定期进行同步与连接测试操作。
    

redis服务器开启后，就会周期性执行此函数，直到redis服务器关闭为止。默认每秒执行10次，平均100毫秒执行一次，可以在redis配置文件的hz选项，调整该函数每秒执行的次数。

### server.hz

serverCron在一秒内执行的次数 ， 在redis/conf中可以配置

```bash
hz 100
```

比如：server.hz是100，也就是servreCron的执行间隔是10ms

### run_with_period

```c
#define run_with_period(_ms_) \
if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))
```

定时任务执行都是在10毫秒的基础上定时处理自己的任务(run_with_period(ms))，即调用 run_with_period(ms)[ms是指多长时间执行一次，单位是毫秒]来确定自己是否需要执行。

返回1表示执行。

假如有一些任务需要每500ms执行一次，就可以在serverCron中用run_with_period(500)把每500ms需要执行一次的工作控制起来。

## 定时事件

定时事件：让一段程序在指定的时间之后执行一次

aeTimeProc(时间处理器)的返回值是AE_NOMORE

该事件在达到后删除，之后不会再重复。

## 周期性事件

周期性事件：让一段程序每隔指定时间就执行一次

aeTimeProc(时间处理器)的返回值不是AE_NOMORE

当一个时间事件到达后，服务器会根据时间处理器的返回值，对时间事件的 when 属性进行更新，让这 个事件在一段时间后再次达到。

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

### 初始化

Redis 服务端在其初始化函数 initServer 中，会创建事件管理器 aeEventLoop 对象。

函数 aeCreateEventLoop 将创建一个事件管理器，主要是初始化 aeEventLoop 的各个属性值，比如events 、 fired 、 timeEventHead 和 apidata :

- 首先创建 aeEventLoop 对象。、
- 初始化注册的文件事件表、就绪文件事件表。 指针指向注册的文件事件表、 指针指 向就绪文件事件表。表的内容在后面添加具体事件时进行初变更。
- 初始化时间事件列表，设置 timeEventHead 和 timeEventNextId 属性。
- 调用 aeApiCreate 函数创建 epoll 实例，并初始化 apidata 。

### stop

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

## 时间事件: timeEventHead, beforesleep, aftersleep

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

## aeMain

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

## aeProcessEvent

首先计算距离当前时间最近的时间事件，以此计算一个超时时间;

然后调用 aeApiPoll 函数去等待底层的I/O多路复用事件就绪;

aeApiPoll 函数返回之后，会处理所有已经产生文件事件和已经达到的时间事件。

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



# 多级缓存

缓存的设计要分多个层次，在不同的层次上选择不同的缓存，包括JVM缓存、文件缓存和Redis缓存。

## JVM缓存

JVM缓存就是本地缓存，设计在应用服务器中(tomcat)。 通常可以采用Ehcache和Guava Cache，在互联网应用中，由于要处理高并发，通常选择Guava Cache。

适用本地(JVM)缓存的场景:

1. 对性能有非常高的要求。
2. 不经常变化
3. 占用内存不大
4. 有访问整个集合的需求
5. 数据不要求实时一致

## 文件缓存

这里的文件缓存是基于http协议的文件缓存，一般放在nginx中。

因为静态文件(比如css，js， 图片)中，很多都是不经常更新的。nginx使用proxy_cache将用户的请求缓存到本地一个目录。下一个相同请求可以直接调取缓存文件，就不用去请求服务器了。

```nginx
server {
    listen 80 default_server;
    server_name  localhost;
    root /mnt/blog/;
    
    location / {
    } 
    #要缓存文件的后缀，可以在以下设置。
    location ~ .*\.(gif|jpg|png|css|js)(.*) { 
        proxy_pass http://ip地址:90;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_cache cache_one;
        proxy_cache_valid 200 302 24h;
        proxy_cache_valid 301 30d;
        proxy_cache_valid any 5m;
        expires 90d;
        add_header wall  "hello lagou.";
    } 
}


```

## Redis缓存

分布式缓存，采用主从+哨兵或RedisCluster的方式缓存数据库的数据。

在实际开发中，比如作为数据库使用，作为Mybatis的二级缓存使用

# 缓存大小

GuavaCache的缓存设置方式:

```java
CacheBuilder.newBuilder().maximumSize(num) // 超过num会按照LRU算法来移除缓存
```

Nginx的缓存设置方式:

```nginx
http { 
   ...
   proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m use_temp_path=off;
   server {
       proxy_cache mycache;
       location / {
          proxy_pass http://localhost:8000;
       }
    } 
}
```

Redis缓存设置:

```.properties
maxmemory=num # 最大缓存量 一般为内存的3/4 
maxmemory-policy allkeys lru #
```

# 缓存淘汰策略的选择

- allkeys-lru : 在不确定时一般采用策略。
- volatile-lru : 比allkeys-lru性能差
- allkeys-random : 希望请求符合平均分布(每个元素以相同的概率被访问)
- 自己控制：volatile-ttl
- 禁止驱逐，用作DB，不设置maxmemory

# key数量

官方说Redis单例能处理key：2.5亿个

一个key或是value大小最大是512M

# 读写峰值

Redis采用的是基于内存的采用的是单进程单线程模型的 KV 数据库，由C语言编写，官方提供的数据是可以达到110000+的QPS(每秒内查询次数)。80000的写

# 命中率

- 命中：可以直接通过缓存获取到需要的数据。
- 不命中：无法直接通过缓存获取到想要的数据，需要再次查询数据库或者执行其它的操作。原因可能是由于缓存中根本不存在，或者缓存已经过期。

通常来讲，缓存的命中率越高则表示使用缓存的收益越高，应用的性能越好(响应时间越短、吞吐量越高)，抗并发的能力越强。 由此可见，在高并发的互联网系统中，缓存的命中率是至关重要的指标。

通过info命令可以监控服务器状态

```bash
127.0.0.1:6379> info
# Server
redis_version:5.0.5
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:e188a39ce7a16352
redis_mode:standalone
os:Linux 3.10.0-229.el7.x86_64 x86_64 arch_bits:64
#缓存命中
keyspace_hits:1000
#缓存未命中 
keyspace_misses:20 
used_memory:433264648 
expired_keys:1333536 
evicted_keys:1547380
```

命中率=1000/1000+20=83% 一个缓存失效机制，和过期时间设计良好的系统，命中率可以做到95%以上。

影响缓存命中率的因素:

1. 缓存的数量越少命中率越高，比如缓存单个对象的命中率要高于缓存集合
2. 过期时间越长命中率越高
3. 缓存越大缓存的对象越多，则命中的越多

# 过期策略

Redis的过期策略是定时删除+惰性删除，这个前面已经讲了。

# 性能监控指标

利用info命令就可以了解Redis的状态了，主要监控指标有:

```.properties
connected_clients:68 #连接的客户端数量 
used_memory_rss_human:847.62M #系统给redis分配的内存 
used_memory_peak_human:794.42M #内存使用的峰值大小 
total_connections_received:619104 #服务器已接受的连接请求数量 
instantaneous_ops_per_sec:1159 #服务器每秒钟执行的命令数量 qps 
instantaneous_input_kbps:55.85 #redis网络入口kps 
instantaneous_output_kbps:3553.89 #redis网络出口kps 
rejected_connections:0 #因为最大客户端数量限制而被拒绝的连接请求数量 
expired_keys:0 #因为过期而被自动删除的数据库键数量
evicted_keys:0 #因为最大内存容量限制而被驱逐(evict)的键数量 
keyspace_hits:0 #查找数据库键成功的次数
keyspace_misses:0 #查找数据库键失败的次数
```

Redis监控平台: grafana、prometheus以及redis_exporter。

# 缓存预热

缓存预热就是系统启动前,提前将相关的缓存数据直接加载到缓存系统。避免在用户请求的时候,先查询数据库,然后再将数据缓存的问题!用户直接查询实现被预热的缓存数据。

加载缓存思路:

- 数据量不大，可以在项目启动的时候自动进行加载
- 利用定时任务刷新缓存，将数据库的数据刷新到缓存中

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

# Hot Key

当有大量的请求(几十万)访问某个Redis某个key时，由于流量集中达到网络上限，从而导致这个redis的 服务器宕机。造成缓存击穿，接下来对这个key的访问将直接访问数据库造成数据库崩溃，或者访问数据库回填Redis再访问Redis，继续崩溃。

![](https://secure2.wostatic.cn/static/kSiLGK3QoDYXmgfPKwabgD/image.png?auth_key=1716829496-fSwuV2AgjXu6ZY9Rmv77tY-0-b67ae03502865a1807631ff393cb424a)

## 如何发现热key

1. 预估热key，比如秒杀的商品、火爆的新闻等
2. 在客户端进行统计，实现简单，加一行代码即可
3. 如果是Proxy，比如Codis，可以在Proxy端收集
4. 利用Redis自带的命令，monitor、hotkeys。但是执行缓慢(不要用)
5. 利用基于大数据领域的流式计算技术来进行实时数据访问次数的统计，比如 Storm、Spark Streaming、Flink，这些技术都是可以的。发现热点数据后可以写到zookeeper中

![](https://secure2.wostatic.cn/static/vAYjWtdvTrf6bs1Mn327yo/image.png?auth_key=1716829496-hSW79zr3PDF3HtZgN5sbfX-0-aeff68b85f2ae18200de826ed942d816)

## 如何处理热Key:

1. 变分布式缓存为本地缓存
    
    发现热key后，把缓存数据取出后，直接加载到本地缓存中。可以采用Ehcache、Guava Cache都可以，这样系统在访问热key数据时就可以直接访问自己的缓存了。(数据不要求实时一致)
    
2. 在每个Redis主节点上备份热key数据，这样在读取时可以采用随机读取的方式，将访问压力负载到每个Redis上。
    
3. 利用对热点数据访问的限流熔断保护措施
    
    每个系统实例每秒最多请求缓存集群读操作不超过 400 次，一超过就可以熔断掉，不让请求缓存集群，直接返回一个空白信息，然后用户稍后会自行再次重新刷新页面之类的。(首页不行，系统友好性差)。通过系统层自己直接加限流熔断保护措施，可以很好的保护后面的缓存集群。
    

# Big Key

大key指的是存储的值(Value)非常大，常见场景:

- 热门话题下的讨论 大V的粉丝列表
- 序列化后的图片
- 没有及时处理的垃圾数据
- .....

## 大key的影响:

- 大key会大量占用内存，在集群中无法均衡 （倾斜）
- Redis的性能下降，主从复制异常
- 在主动删除或过期删除时会操作时间过长而引起服务阻塞

## 如何发现大key:

1. redis-cli --bigkeys命令。可以找到某个实例5种数据类型(String、hash、list、set、zset)的最大 key。但如果Redis 的key比较多，执行该命令会比较慢
2. 获取生产Redis的rdb文件，通过rdbtools分析rdb生成csv文件，再导入MySQL或其他数据库中进行分析统计，根据size_in_bytes统计bigkey

## 大key的处理:

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

### 缓存与数据库一致性

#### 缓存更新策略

- 利用Redis的缓存淘汰策略被动更新 LRU 、LFU
    
- 利用TTL被动更新
    
- 在更新数据库时主动更新 (先更数据库再删缓存----延时双删)
    
    异步更新 定时任务 数据不保证时时一致 不穿DB
    

#### 不同策略之间的优缺点

|策略|一致性|维护成本|
|---|---|---|
|利用Redis的缓存淘汰策略被动更新|最差|最低|
|利用TTL被动更新|较差|较低|
|在更新数据库时主动更新|较强|最高|

#### 与Mybatis整合

可以使用Redis做Mybatis的二级缓存，在分布式环境下可以使用。 框架采用springboot+Mybatis+Redis。

1. 在pom.xml中添加Redis依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2. 在application.yml中添加Redis配置

```yaml
#开发配置 
spring:
#数据源配置 
  datasource:
    url: jdbc:mysql://192.168.127.128:3306/test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
  redis:
    host: 192.168.127.128
    port: 6379
    jedis:
      pool:
        min-idle: 0
        max-idle: 8
        max-active: 8
        max-wait: -1ms
  #公共配置与profiles选择无关 
  mybatis:
    typeAliasesPackage: com.lagou.rcache.entity
    mapperLocations: classpath:mapper/*.xml
```

3. 缓存实现
    
    ApplicationContextHolder 用于注入RedisTemplate
    

```java
@Component
public class ApplicationContextHolder implements ApplicationContextAware {
    private static ApplicationContext ctx;
    @Override
    //向工具类注入applicationContext
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        ctx = applicationContext; //ctx就是注入的applicationContext 
    }
    
    //外部调用ctx
    public static ApplicationContext getCtx() {
        return ctx; 
    }
    
    public static <T> T getBean(Class<T> tClass) {
        return ctx.getBean(tClass);
    }
    
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) {
        return (T) ctx.getBean(name);
    }
}
```

```
RedisCache 使用redis实现mybatis二级缓存
```

```java
/**
 * 使用redis实现mybatis二级缓存 
 */
public class RedisCache implements Cache {
    //缓存对象唯一标识
    private final String id; //orm的框架都是按对象的方式缓存，而每个对象都需要一个唯一标识.
    //用于事务性缓存操作的读写锁
    private static ReadWriteLock readWriteLock = new ReentrantReadWriteLock(); //处理事务性缓存中做的
    //操作数据缓存的--跟着线程走的
    private RedisTemplate redisTemplate; //Redis的模板负责将缓存对象写到redis服务器里面去
    //缓存对象的是失效时间，30分钟
    private static final long EXPRIRE_TIME_IN_MINUT = 30;
    //构造方法---把对象唯一标识传进来 
    public RedisCache(String id) {
        if (id == null) {
            throw new IllegalArgumentException("缓存对象id是不能为空的");
        }
        this.id = id;
    }
    
    @Override
    public String getId() {
        return this.id;
    }
    //给模板对象RedisTemplate赋值，并传出去 
    private RedisTemplate getRedisTemplate() {
        if (redisTemplate == null) { 
            //每个连接池的连接都要获得RedisTemplate 
            redisTemplate = ApplicationContextHolder.getBean("redisTemplate");
        }
        return redisTemplate;
    }
    /*
    保存缓存对象的方法
    */
    @Override
    public void putObject(Object key, Object value) {
        try {
            RedisTemplate redisTemplate = getRedisTemplate(); 
            //使用redisTemplate得到值操作对象
            ValueOperations operation = redisTemplate.opsForValue(); 
            //使用值操作对象operation设置缓存对象
            operation.set(key, value, EXPRIRE_TIME_IN_MINUT, TimeUnit.MINUTES); //TimeUnit.MINUTES系统当前时间的分钟数
            System.out.println("缓存对象保存成功"); 
        } catch (Throwable t) {
            System.out.println("缓存对象保存失败" + t); 
        }
    }
    /*
      获取缓存对象的方法
    */
    @Override
    public Object getObject(Object key) {
        try {
            RedisTemplate redisTemplate = getRedisTemplate(); 
            ValueOperations operations = redisTemplate.opsForValue(); 
            Object result = operations.get(key); System.out.println("获取缓存对象");
            return result;
            
        } catch (Throwable t) { 
            System.out.println("缓存对象获取失败" + t); 
            return null;
        } 
    }
    /*
    删除缓存对象

    */
    @Override
    public Object  (Object key) {
        try {
            RedisTemplate redisTemplate = getRedisTemplate(); 
            redisTemplate.delete(key); 
            System.out.println("删除缓存对象成功!");
        } catch (Throwable t) { 
            System.out.println("删除缓存对象失败!" + t);
        }
        return null;
    }
    /*
    清空缓存对象
当缓存的对象更新了的化，就执行此方法
    */
    @Override
    public void clear() {
        RedisTemplate redisTemplate = getRedisTemplate(); //回调函数
        redisTemplate.execute((RedisCallback) collection -> {
            collection.flushDb();
            return null;
        });
        System.out.println("清空缓存对象成功!"); 
    }
    //可选实现的方法 
    @Override
    public int getSize() {
        return 0; 
    }
    
    @Override
    public ReadWriteLock getReadWriteLock() {
        return readWriteLock;
    } 
}

```

4. 在mapper中增加二级缓存开启(默认不开启)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.lagou.rcache.dao.UserDao" >
    <cache type="com.lagou.rcache.utils.RedisCache" />
    <resultMap id="BaseResultMap" type="com.lagou.rcache.entity.TUser" >

   <id column="id" property="id" jdbcType="INTEGER" />
        <result column="name" property="name" jdbcType="VARCHAR" />
        <result column="address" property="address" jdbcType="VARCHAR" />
    </resultMap>
    <sql id="Base_Column_List" >
        id, name, address
    </sql>
    <select id="selectUser" resultMap="BaseResultMap">
        select
        <include refid="Base_Column_List" />
        from tuser
    </select>
</mapper> 
```

5. 在启动时允许缓存

```java
@SpringBootApplication
@MapperScan("com.lagou.rcache.dao")
@EnableCaching
public class RcacheApplication {
    public static void main(String[] args) {
        SpringApplication.run(RcacheApplication.class, args);
    } 
}
```

6. 运行结果: 控制台:
    
    ![](https://secure2.wostatic.cn/static/da2RPVUQVvL8eZ2k3tPD1g/image.png?auth_key=1716829497-nQg2ywDbJ72vixPkKX62Zm-0-e2ef4155b71cfae20b3d965c8279c417)
    
7. Redis客户端:
    

```bash
127.0.0.1:6379> keys *
1) "\xac\xed\x00\x05sr\x00
org.apache.ibatis.cache.CacheKey\x0f\xe9\xd5\xb4\xcd3\xa8\x82\x02\x00\x05J\x00\b
checksumI\x00\x05countI\x00\bhashcodeI\x00\nmultiplierL\x00\nupdateListt\x00\x10
Ljava/util/List;xp\x00\x00\x00\x00\x87\xd8u%\x00\x00\x00\x05\xb0\xf6RU\x00\x00\x
00%sr\x00\x13java.util.ArrayListx\x81\xd2\x1d\x99\xc7a\x9d\x03\x00\x01I\x00\x04s
izexp\x00\x00\x00\x05w\x04\x00\x00\x00\x05t\x00'com.lagou.rcache.dao.UserDao.sel
ectUsersr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00
\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00x
p\x00\x00\x00\x00sq\x00~\x00\x06\x7f\xff\xff\xfft\x00Cselect\n         \n
id, name, address\n     \n        from tusert\x00\x15SqlSessionFactoryBeanx"
```

# 分布式锁

分布式锁

# 分布式集群架构中的session分离

略

# Redis的Key的设计

1. 用:分割
2. 把表名转换为key前缀, 比如: user:
3. 第二段放置主键值
4. 第三段放置列名

传统的session是由tomcat自己进行维护和管理，但是对于集群或分布式环境，不同的tomcat管理各自 的session，很难进行session共享，通过传统的模式进行session共享，会造成session对象在各个 tomcat之间，通过网络和Io进行复制，极大的影响了系统的性能。

可以将登录成功后的Session信息，存放在Redis中，这样多个服务器(Tomcat)可以共享Session信息。 利用spring-session-data-redis(SpringSession)，可以实现基于redis来实现的session分离。这个

知识点在讲Spring的时候可以讲过了，这里就不再赘述了。

# 阿里Redis使用手册

本文主要介绍在使用阿里云Redis的开发规范，从下面几个方面进行说明。

- 键值设计
- 命令使用
- 客户端使用

## 键值设计

1. key名设计
    
    - 可读性和可管理性 以业务名(或数据库名)为前缀(防止key冲突)，用冒号分隔，比如业务名:表名:id
        
        ![](https://secure2.wostatic.cn/static/ccQn9G5a5uFrcGKUd6i5er/image.png?auth_key=1716829497-oW9ZVFf7RKLbX68fLW86eE-0-471e53e784adb7d5ce1ee5d5320f05e7)
        
    - 简洁性
        
        保证语义的前提下，控制key的长度，当key较多时，内存占用也不容忽视，例如:
        
        ![](https://secure2.wostatic.cn/static/4PoEnSMfrhSiKtEFtysPE/image.png?auth_key=1716829497-3wurrohsegfxh2ghwrjLC5-0-15570c91b6483550e274cf5dd6c486a9)
        
    - 不要包含特殊字符。
        
        反例:包含空格、换行、单双引号以及其他转义字符
        
2. value设计
    
    拒绝bigkey 防止网卡流量、慢查询，string类型控制在10KB以内，hash、list、set、zset元素个数不要超过5000。 反例:一个包含200万个元素的list。 拆解非字符串的bigkey，不要使用del删除，使用hscan、sscan、zscan方式渐进式删除，同时要注意防止 bigkey过期时间自动删除问题(例如一个200万的zset设置1小时过期，会触发del操作，造成阻塞，而且该操作不会不出现在慢查询中(latency可查))，查找方法和删除方法选择适合的数据类型 例如:实体类型(要合理控制和使用数据结构内存编码优化配置,例如ziplist，但也要注意节省内存和性能之间的平衡) 反例:
    
    ![](https://secure2.wostatic.cn/static/o7Gc4PQeQNYhJvwwMMemRn/image.png?auth_key=1716829497-gtZk671k5kajhPff7hBeMb-0-a1144855fcccad5424c3598397517615)
    
    正例:
    
    ![](https://secure2.wostatic.cn/static/ax3Hp6voQ2XLBbXr2FEcnb/image.png?auth_key=1716829497-pVEc4ZRa7qspnvJcGT2AKD-0-0c9186d847813b24a83c937a9cc2c9a8)
    
3. 控制key的生命周期
    
    redis不是垃圾桶，建议使用expire设置过期时间(条件允许可以打散过期时间，防止集中过期)，不过期的数据重点关注idletime。
    

## 命令使用

1. O(N)命令关注N的数量
    
    例如hgetall、lrange、smembers、zrange、sinter等并非不能使用，但是需要明确N的值。有遍历 需求可以使用hscan、sscan、zscan代替。
    
2. 禁用命令
    
    禁止线上使用keys、flushall、flushdb等，通过redis的rename机制禁掉命令，或者使用scan的方式渐进式处理。
    
3. 合理使用select
    
    redis的多数据库较弱，使用数字进行区分，很多客户端支持较差，同时多业务用多数据库实际还是单线程处理，会有干扰。
    
4. 使用批量操作提高效率
    
    - 原生命令：例如mget、mset。
    - 非原生命令：可以使用pipeline提高效率。 但要注意控制一次批量操作的元素个数(例如500以内，实际也和元素字节数有关)。
    - 注意两者不同:
        1. 原生是原子操作，pipeline是非原子操作。
        2. pipeline可以打包不同的命令，原生做不到
        3. pipeline需要客户端和服务端同时支持。
5. 不建议过多使用Redis事务功能
    
    Redis的事务功能较弱(不支持回滚)，而且集群版本(自研和官方)要求一次事务操作的key必须在一个slot 上(可以使用hashtag功能解决)
    
6. Redis集群版本在使用Lua上有特殊要求
    
    1. 所有key都应该由 KEYS 数组来传递，redis.call/pcall 里面调用的redis命令，key的位置，必须是 KEYS array, 否则直接返回error，"-ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS arrayrn"
    2. 所有key，必须在1个slot上，否则直接返回error, "-ERR eval/evalsha command keys must in same slotrn"
7. monitor命令 必要情况下使用monitor命令时，要注意不要长时间使用。
    

## 客户端使用

1. 避免多个应用使用一个Redis实例
    
    不相干的业务拆分，公共数据做服务化。
    
2. 使用连接池
    
    可以有效控制连接，同时提高效率，标准使用方式:
    
    ![](https://secure2.wostatic.cn/static/uXsk3eEQDqjgZnWWKbnpeG/image.png?auth_key=1716829497-csuA426NtJPgSiMYLV3xuF-0-c6aa1d5741613dedfbc779ae7ba65013)
    
3. 熔断功能
    
    高并发下建议客户端添加熔断功能(例如netflix hystrix)
    
4. 合理的加密
    
    设置合理的密码，如有必要可以使用SSL加密访问(阿里云Redis支持)
    
5. 淘汰策略
    
    根据自身业务类型，选好maxmemory-policy(最大内存淘汰策略)，设置好过期时间。
    
    默认策略是volatile-lru，即超过最大内存后，在过期键中使用lru算法进行key的剔除，保证不过期数据 不被删除，但是可能会出现OOM问题。
    
    其他策略如下:
    
    - allkeys-lru:根据LRU算法删除键，不管数据有没有设置超时属性，直到腾出足够空间为止。
    - allkeys-random:随机删除所有键，直到腾出足够空间为止。
    - volatile-random:随机删除过期键，直到腾出足够空间为止。
    - volatile-ttl:根据键值对象的ttl属性，删除最近将要过期数据。如果没有，回退到noeviction策略。
    - noeviction:不会剔除任何数据，拒绝所有写入操作并返回客户端错误信息"(error) OOM command not allowed when used memory"，此时Redis只响应读操作。

## 相关工具

1. 数据同步
    
    redis间数据同步可以使用:redis-port
    
2. big key搜索
    
    redis大key搜索工具
    
3. 热点key寻找 内部实现使用monitor，所以建议短时间使用facebook的redis-faina 阿里云Redis已经在内核层面解决热点key问题
    

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