# Redis的数据结构
* string
* list
* set
* sortedset
* hash
* bitmap
* geo
* stream
## string

> Redis的string能表达3种值的类型：字符串、整数、浮点数，例如 100.01 是个六位的string

应用场景：

1. key和命令是字符串
2. 普通的赋值
3. incr用于乐观锁 incr:递增数字，可用于实现乐观锁 watch(事务)
4. setnx用于分布式锁 当value不存在时采用赋值，可用于实现分布式锁

## list

>list列表类型可以存储有序、可重复的元素，获取头部或尾部附近的记录是极快的，list的元素个数最多为2^32-1个(40亿)

应用场景：

1. 作为栈或队列使用
2. 可用于各种列表，比如用户列表、商品列表、评论列表等。

## set

>无序、唯一元素，集合中最大的成员数为 2^32 - 1

应用场景：

适用于不能重复的且不需要顺序的数据结构，比如：关注的用户，还可以通过spop进行随机抽奖

## sortedset

>也称为ZSet，有序集合元素本身是无序不重复的，每个元素关联一个分数(score) 可按分数排序，分数可重复

应用场景：

由于可以按照分值排序，所以适用于各种排行榜。比如:点击排行榜、销量排行榜、关注排行榜等。

## hash

>Redis hash 是一个 string 类型的 field 和 value 的映射表，它提供了字段和字段值的映射。 每个 hash 可以存储 2^32 - 1 键值对(40多亿)。

![](Redis-hash.jpg)

应用场景：

对象的存储 ，表数据的映射

## bitmap

>bitmap是进行位操作的通过一个bit位来表示某个元素对应的值或者状态，其中的key就是对应元素本身。 bitmap本身会极大的节省储存空间。

应用场景:

1. 用户每月签到，用户id为key ， 日期作为偏移量 1表示签到
2. 统计活跃用户, 日期为key，用户id为偏移量 1表示活跃
3. 查询用户在线状态， 日期为key，用户id为偏移量 1表示在线

## geo

>geo是Redis用来处理位置信息的。在Redis3.2中正式使用。主要是利用了Z阶曲线、Base32编码和geohash算法

应用场景:

1. 记录地理位置
2. 计算距离
3. 查找"附近的人"

## stream

>stream是Redis5.0后新增的数据结构，用于可持久化的消息队列。 几乎满足了消息队列具备的全部内容，包括：消息ID的序列化生成、消息遍历、消息的阻塞和非阻塞读取、消息的分组消费、未完成消息的处理、消息队列监控。每个Stream都有唯一的名称，它就是Redis的key，首次使用 xadd 指令追加消息时自动创建。

应用场景:

消息队列的使用


# 底层数据结构

Redis作为Key-Value存储系统，数据结构如下:

![](Redis数据结构.jpg)

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

* `id`：id是数据库序号，为0-15(默认Redis有16个数据库)
* `dict`：存储数据库所有的key-value
* `expires`：存储key的过期时间
## RedisObject结构

Value是一个对象，包含字符串对象，列表对象，哈希对象，集合对象和有序集合对象

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

* **4位type**：type 字段表示对象的类型，占 4 位;

	`REDIS_STRING(字符串)`、`REDIS_LIST (列表)`、`REDIS_HASH(哈希)`、`REDIS_SET(集合)`、`REDIS_ZSET(有序集合)`。

	当我们执行 type 命令时，便是通过读取 `RedisObject` 的 `type` 字段获得对象的类型。

	```bash
	127.0.0.1:6379> type a1
	string
	```

* **4位encoding**：encoding 表示对象的内部编码，占 4 位

	每个对象有不同的实现编码
	
	Redis 可以根据不同的使用场景来为对象设置不同的编码，大大提高了 Redis 的灵活性和效率。 通过`object encoding` 命令，可以查看对象采用的编码方式
	
	```bash
	127.0.0.1:6379>  object encoding a1
	"int"
	```

* **24位LRU**

	lru 记录的是对象最后一次被命令程序访问的时间，( 4.0 版本占 24 位，2.6 版本占 22 位)。 高16位存储一个分钟数级别的时间戳，低8位存储访问计数(lfu : 最近访问次数)  
	* 淘汰策略lru---> 高16位： 最后被访问的时间  
	* 淘汰策略lfu--->低8位：最近访问次数

* **refcount**

	refcount 记录的是该对象被引用的次数，类型为整型。
	
	refcount 的作用，主要在于对象的引用计数和内存回收。 当对象的refcount 大于 1时，称为共享对象
	
	Redis 为了节省内存，当有一些对象重复出现时，新的程序不会创建新的对象，而是仍然使用原来的对象。

* **ptr**

	ptr 指针指向具体的数据，比如:set hello world，ptr 指向包含字符串 world 的 SDS。

## 底层基本数据结构

| `REDIS_ENCODING_INT`        | `long` 类型的整数        |
| --------------------------- | ------------------- |
| `REDIS_ENCODING_EMBSTR`     | `embstr` 编码的简单动态字符串 |
| `REDIS_ENCODING_RAW`        | 简单动态字符串             |
| `REDIS_ENCODING_HT`         | 字典                  |
| `REDIS_ENCODING_LINKEDLIST` | 双端链表                |
| `REDIS_ENCODING_ZIPLIST`    | 压缩列表                |
| `REDIS_ENCODING_INTSET`     | 整数集合                |
| `REDIS_ENCODING_SKIPLIST`   | 跳跃表和字典              |
| ...                         |                     |
 
### 字符串对象

Redis 使用了 SDS(Simple Dynamic String)。用于存储字符串和整型数据。SDS的底层实现有三种：int、embstr、raw。

![](reids-sds.png)

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

> buf[] 的长度=len+free+1

**SDS的优势:**

1. SDS 在 C语言字符串的基础上加入了 free 和 len 字段，获取字符串长度:SDS 是 O(1)，C语言获取字符串长度是 O(n)。
2. SDS 由于记录了长度，在可能造成缓冲区溢出时会自动重新分配内存，杜绝了缓冲区溢出。
3. 可以存取二进制数据，以字符串长度len来作为结束标识  
    C: \0 空字符串 二进制数据包括空字符串，所以没有办法存取二进制数据  
    SDS : 存非二进制 \0作为判断 ，存二进制: 字符串长度作为判断

**使用场景:**

SDS的主要应用在：存储字符串和整型数据、存储key、AOF缓冲区和用户输入缓冲。

### 跳表(重点)

跳表是有序集合(sorted-set)的底层实现，效率高，实现简单。

>跳跃表的基本思想：将有序链表中的部分节点分层，每一层都是一个有序链表。

#### 查找

在查找时优先从最高层开始向后查找，当到达某个节点时，如果next节点值大于要查找的值或next指针 指向null，则从当前节点下降一层继续向后查找。

举例:

![](跳表1.jpg)

查找元素9，按道理我们需要从头结点开始遍历，一共遍历8个结点才能找到元素9。

第一次分层: 遍历5次找到元素9(红色的线为查找路径)

![](跳表2.jpg)

第二次分层: 遍历4次找到元素9

![](跳表3.jpg)

第三层分层: 遍历4次找到元素9

![](跳表4.jpg)

这种数据结构，就是跳跃表，它具有二分查找的功能。 插入与删除上面例子中，9个结点，一共4层，是理想的跳跃表。

通过抛硬币(概率1/2)的方式来决定新插入结点跨越的层数：正面：插入上层，背面：不插入，达到1/2概率(计算次数)

#### 删除

找到指定元素并删除每层的该元素即可

#### 跳跃表特点

* 每层都是一个有序链表
* 查找次数近似于层数(1/2)
* 底层包含所有元素
* 空间复杂度 O(n) 扩充了一倍

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

![](redis跳表结构.webp)

**跳跃表的优势:**

1. 可以快速查找到需要的节点
2. 可以在O(1)的时间复杂度下，快速获得跳跃表的头节点、尾结点、长度和高度。

**应用场景：**

有序集合的实现

### 字典(重点+难点)

字典dict又称散列表(hash)，是用来存储键值对的一种数据结构。Redis整个数据库是用字典来存储的。(K-V结构)。对Redis进行CURD操作其实就是对字典中的数据进行CURD操作。

**数组**：用来存储数据的容器，采用头指针+偏移量的方式能够以O(1)的时间复杂度定位到数据所在的内存地址。

**Hash函数**：作用是把任意长度的输入通过散列算法转换成固定类型、固定长度的散列值。 hash函数可以把Redis里的key:包括字符串、整数、浮点数统一转换成整数。

**数组下标**=hash(key)%数组容量(hash值%数组容量得到的余数)

#### **Hash冲突**

不同的key经过计算后出现数组下标一致，称为Hash冲突。 采用单链表在相同的下标位置处存储原始key和value 当根据key找Value时，找到数组下标，遍历单链表可以找出key相同的value

![](redis-hash冲突.webp)

#### Redis字典的实现

Redis字典实现包括：字典(dict)、Hash表(dictht)、Hash表节点(dictEntry)。

![](redis字典实现类图.webp)

##### **Hash表**

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

![](redis-hash表.webp)

##### dict字典

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

#### Redis字典数据结构:

![](redis完整字典结构.webp)

#### 字典扩容rehash

字典达到存储上限，需要rehash(扩容) 扩容流程:

![](redis字典扩容.webp)

说明:

1. 初次申请默认容量为4个dictEntry，非初次申请为当前hash表容量的一倍。
2. rehashidx=0表示要进行rehash操作。
3. 新增加的数据在新的hash表`h[1]`
4. 修改、删除、查询在老hash表`h[0]`、新hash表`h[1]`中(rehash中)
5. 将老的hash表`h[0]`的数据重新计算索引值后全部迁移到新的hash表`h[1]`中，这个过程称为 rehash。

![](rehash.jpg)
#### 渐进式rehash

当数据量巨大时rehash的过程是非常缓慢的，所以需要进行优化。 服务器忙，则只对一个节点进行rehash，服务器闲，可批量rehash(100节点)
![](渐进式.jpg)  

#### **字典应用场景**

1. 主数据库的K-V数据存储
2. 散列表对象(hash)
3. 哨兵模式中的主从节点管理

### 压缩列表

> 压缩列表(ziplist)是由一系列特殊编码的连续内存块组成的顺序型数据结构，是一个字节数组，可以包含多个节点(entry)。每个节点可以保存一个字节数组或一个整数。

压缩列表的数据结构如下:

![](redis压缩列表的数据结构.webp)

- `zlbytes`：压缩列表的字节长度
- `zltail`：压缩列表尾元素相对于压缩列表起始地址的偏移量
- `zllen`：压缩列表的元素个数
- `entry1..entryX`：压缩列表的各个节点
- `zlend`：压缩列表的结尾，占一个字节，恒为0xFF(255)

entryX元素的编码结构：

![](entryX数据结构.webp)
    
- `previous_entry_length`：前一个元素的字节长度
- `encoding`：表示当前元素的编码
- `content`：数据内容

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

**应用场景:**

* sorted-set和hash元素个数少且是小整数或短字符串(直接使用) 
* list用快速链表(quicklist)数据结构存储，而快速链表是双向列表与压缩列表的组合。(间接使用)

### 整数集合

整数集合(intset)是一个有序的(整数升序)、存储整数的连续存储结构。当Redis集合类型的元素都是整数并且都处在64位有符号整数范围内(2^64)，使用该结构体存储。

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

![](inset结构图.webp)

```c
typedef struct intset{
    uint32_t encoding;  //编码方式
    uint32_t length;   //集合包含的元素数量
    int8_t contents[]; //保存元素的数组 
}intset;
```

**应用场景:**

可以保存类型为int16_t、int32_t 或者int64_t 的整数值，并且保证集合中不会出现重复元素。

### 快速列表(重要)

快速列表(quicklist)是Redis底层重要的数据结构。是列表的底层实现。(在Redis3.2之前，Redis采用双向链表(adlist)和压缩列表(ziplist)实现。)在Redis3.2以后结合adlist和ziplist的优势Redis设计出了quicklist。

```bash
127.0.0.1:6379> lpush list:001 1 2 5 4 3
(integer) 5
127.0.0.1:6379> object encoding list:001
"quicklist"
```

#### 双向列表(adlist)

![](adlist.webp)

双向链表优势:

1. 双向：链表具有前置节点和后置节点的引用，获取这两个节点时间复杂度都为O(1)。
2. 普通链表(单链表)：节点类保留下一节点的引用。链表类只保留头节点的引用，只能从头节点插入删除
3. 无环:表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL,对链表的访问都是以 NULL 结束。
    环状:头的前一个节点指向尾节点
4. 带链表长度计数器：通过 len 属性获取链表长度的时间复杂度为 O(1)。
5. 多态：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。

#### 快速列表

quicklist是一个双向链表，链表中的每个节点时一个ziplist结构。quicklist中的每个节点ziplist都能够存储多个数据元素。

![](quicklist.webp)

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

### 流对象

stream主要由:消息、生产者、消费者和消费组构成。

![](redis流对象.webp)

Redis Stream的底层主要使用了listpack(紧凑列表)和Rax树(基数树)。

#### listpack

listpack表示一个字符串列表的序列化，listpack可用于存储字符串或整数。用于存储stream的消息内容。

![](listpack.webp)

#### Rax树

Rax 是一个有序字典树 (基数树 Radix Tree)，按照 key 的字典序排列，支持快速地定位、插入和删除操作。

![](rax树.webp)

Rax 被用在 Redis Stream 结构里面用于存储消息队列，在 Stream 里面消息 ID 的前缀是时间戳 + 序号，这样的消息可以理解为时间序列消息。使用 Rax 结构进行存储就可以快速地根据消息 ID 定位到具体的消息，然后继续遍历指定消息之后的所有消息。

![](rax结构.webp)

应用场景: stream的底层实现

## encoding

* encoding 表示对象的内部编码，占 4 位。
* Redis通过 encoding 属性为对象设置不同的编码
* 对于少的和小的数据，Redis采用小的和压缩的存储方式

### **string**

string的编码是int、raw、embstr

- int：REDIS_ENCODING_INT(int类型的整数)

```bash
127.0.0.1:6379> set n1 123
OK
127.0.0.1:6379> object encoding n1
"int"
```

- embstr：
	- REDIS_ENCODING_EMBSTR(编码的简单动态字符串) 
	- 小字符串 长度小于44个字节

```bash
127.0.0.1:6379> set name:001 zhangfei
OK
127.0.0.1:6379> object encoding name:001
"embstr"
```

- raw
    * REDIS_ENCODING_RAW (简单动态字符串) 
    * 大字符串 长度大于44个字节

```bash
127.0.0.1:6379> set address:001
asdasdasdasdasdasdsadasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdas
dasdasdas
OK
127.0.0.1:6379> object encoding address:001
"raw"
```

### **list**

* 列表的编码是quicklist。 
* REDIS_ENCODING_QUICKLIST(快速列表)

```bash
127.0.0.1:6379> lpush list:001 1 2 5 4 3
(integer) 5
127.0.0.1:6379> object encoding list:001
"quicklist"
```

### **hash**

散列的编码是字典dict和压缩列表ziplist

 * dict
	* REDIS_ENCODING_HT(字典) 
	* 当散列表元素的个数比较多或元素不是小整数或短字符串时。

```bash
127.0.0.1:6379>  hmset user:003
username111111111111111111111111111111111111111111111111111111111111111111111111
11111111111111111111111111111111  zhangfei password 111 num
2300000000000000000000000000000000000000000000000000
OK
127.0.0.1:6379> object encoding user:003
"hashtable"
```

* ziplist
	* REDIS_ENCODING_ZIPLIST(压缩列表) 
	* 当散列表元素的个数比较少，且元素都是小整数或短字符串时。

```bash
127.0.0.1:6379> hmset user:001  username zhangfei password 111 age 23 sex M
OK
127.0.0.1:6379> object encoding user:001
"ziplist"
```

### **set**

集合的编码是整形集合intset和字典dict

- intset
	- REDIS_ENCODING_INTSET(整数集合) 
	- 当Redis集合类型的元素都是整数并且都处在64位有符号整数范围内(<18446744073709551616)

```bash
127.0.0.1:6379> sadd set:001 1  3 5 6 2
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
```

- dict
	- REDIS_ENCODING_HT(字典) 
	- 当Redis集合类型的元素都是整数并且都处在64位有符号整数范围外(>18446744073709551616)

```bash
127.0.0.1:6379> sadd set:004 1 100000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
```

### **zset**

有序集合的编码是压缩列表和跳跃表+字典

- ziplist
	- REDIS_ENCODING_ZIPLIST(压缩列表) 
	- 当元素的个数比较少，且元素都是小整数或短字符串时。

```bash
127.0.0.1:6379> zadd hit:1 100 item1 20 item2 45 item3
(integer) 3
127.0.0.1:6379> object encoding hit:1
"ziplist"
```

- skiplist + dict
	- REDIS_ENCODING_SKIPLIST(跳跃表+字典) 
	- 当元素的个数比较多或元素不是小整数或短字符串时。

```bash
127.0.0.1:6379>  zadd hit:2 100
item1111111111111111111111111111111111111111111111111111111111111111111111111111
1111111111111111111111111111111111 20 item2 45 item3
(integer) 3
127.0.0.1:6379>  object encoding hit:2
"skiplist"
```


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

## Redis集群的搭建:

该部分可以参考这篇文章：[https://juejin.cn/post/6922690589347545102#heading-1](https://juejin.cn/post/6922690589347545102#heading-1)

Redis集群的搭建可以分为以下几个部分：

1. 启动节点：将节点以集群模式启动，读取或者生成集群配置文件，此时节点是独立的。
2. 节点握手：节点通过gossip协议通信，将独立的节点连成网络，主要使用meet命令。
3. 槽指派：将16384个槽位分配给主节点，以达到分片保存数据库键值对的效果。

# Redis集群的运维:

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

## 集群的故障检测与故障转恢复机制：

### **集群的故障检测：**

Redis集群的故障检测是基于gossip协议的，集群中的每个节点都会定期地向集群中的其他节点发送PING消息，以此交换各个节点状态信息，检测各个节点状态：在线状态、疑似下线状态PFAIL、已下线状态FAIL。

（1）主观下线（pfail）：当节点A检测到与节点B的通讯时间超过了cluster-node-timeout 的时候，就会更新本地节点状态，把节点B更新为主观下线。

> 主观下线并不能代表某个节点真的下线了，有可能是节点A与节点B之间的网络断开了，但是其他的节点依旧可以和节点B进行通讯。

（2）客观下线：

由于集群内的节点会不断地与其他节点进行通讯，下线信息也会通过 Gossip 消息传遍所有节点，因此集群内的节点会不断收到下线报告。

当半数以上的主节点标记了节点B是主观下线时，便会触发客观下线的流程（该流程只针对主节点，如果是从节点就会忽略）。将主观下线的报告保存到本地的 ClusterNode 的结构fail_reports链表中，并且对主观下线报告的时效性进行检查，如果超过 cluster-node-timeout*2 的时间，就忽略这个报告，否则就记录报告内容，将其标记为客观下线。

接着向集群广播一条主节点B的Fail 消息，所有收到消息的节点都会标记节点B为客观下线。

### **集群地故障恢复：**

当故障节点下线后，如果是持有槽的主节点则需要在其从节点中找出一个替换它，从而保证高可用。此时下线主节点的所有从节点都担负着恢复义务，这些从节点会定时监测主节点是否进入客观下线状态，如果是，则触发故障恢复流程。故障恢复也就是选举一个节点充当新的master，选举的过程是基于Raft协议选举方式来实现的。
* 从节点过滤：
* 投票选举：
* 替换主节点：

#### 从节点过滤：

检查每个slave节点与master节点断开连接的时间，如果超过了cluster-node-timeout * cluster-slave-validity-factor，那么就没有资格切换成master

#### 投票选举：

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

#### 替换主节点：

当满足投票条件的从节点被选出来以后，会触发替换主节点的操作。删除原主节点负责的槽数据，把这些槽数据添加到自己节点上，并且广播让其他的节点都知道这件事情，新的主节点诞生了。

（1）被选中的从节点执行SLAVEOF NO ONE命令，使其成为新的主节点

（2）新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己

（3）新的主节点对集群进行广播PONG消息，告知其他节点已经成为新的主节点

（4）新的主节点开始接收和处理槽相关的请求

> 备注：如果集群中某个节点的master和slave节点都宕机了，那么集群就会进入fail状态，因为集群的slot映射不完整。如果集群超过半数以上的master挂掉，无论是否有slave，集群都会进入fail状态。

## Redis主从、哨兵、集群比较
todo
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


# 发布与订阅

Redis提供了发布订阅功能，可以用于消息的传输 Redis的发布订阅机制包括三个部分，`publisher`，`subscriber` 和 `Channel`

![](redis发布订阅.png)

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

Redis客户端1接收到频道1和频道2的消息

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

- psubscribe ：模式匹配 `psubscribe + 模式`
    
    Redis客户端1订阅所有以ch开头的频道
	
	```bash
	127.0.0.1:6379> psubscribe ch*
	Reading messages... (press Ctrl-C to quit)
	1) "psubscribe"
	2) "ch*"
	3) (integer) 1
```

Redis客户端2发布信息在频道5上

```bash
127.0.0.1:6379> publish ch5 helloworld
(integer) 1
```

Redis客户端1收到频道5的信息

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
    
    属性为`pubsub_channels`，该属性表明了该客户端订阅的所有频道
    
    属性为`pubsub_patterns`，该属性表示该客户端订阅的所有模式
    
- 服务器端(RedisServer)：
    
    属性为`pubsub_channels`，该服务器端中的所有频道以及订阅了这个频道的客户端
    
    属性为`pubsub_patterns`，该服务器端中的所有模式和订阅了这些模式的客户端
    

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

当客户端向某个频道发送消息时，Redis首先在redisServer中的`pubsub_channels`中找出键为该频道的节点，遍历该节点的值，即遍历订阅了该频道的所有客户端，将消息发送给这些客户端。

然后，遍历结构体redisServer中的`pubsub_patterns`，找出包含该频道的模式的节点，将消息发送给订阅了该模式的客户端。

## 使用场景

* 在Redis哨兵模式中，哨兵通过发布与订阅的方式与Redis主服务器和Redis从服务器进行通信。
* Redisson是一个分布式锁框架，在Redisson分布式锁释放的时候，是使用发布与订阅的方式通知的。

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
	```text
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

#  与Mybatis整合做二级缓存
略。