
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

如果报错，说找不到readline/readline.h, 可以通过yum命令安装

```bash
yum -y install readline-devel ncurses-devel
```

安装完以后再

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