#flashcards 

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
### linkedlist
### hashtable
### skiplist
### intset
### ziplist
### quicklist

# key的过期策略
- 惰性删除  
    >碰到过期键时才进行删除操作  
- 定期删除  
    >每隔一段时间， 主动查找并删除过期键

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
# 集群