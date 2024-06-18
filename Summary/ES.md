#flashcards 

## ES的工作流程
### 读流程
1. 客户端请求集群协调节点  
2. 协调节点计算数据所在的主分片和所有副本的位置  
3. 负载均衡轮询所有分片  
4. 请求转发给具体节点  
5. 节点数据返回协调节点  
6. 协调节点返回客户端
### 写流程
1. 客户端请求集群协调节点  
2. 协调节点将请求转发给指定节点  
    >协调节点默认使用文档 ID 参与计算（也支持通过 routing），以便为路由提供合适的分片：`shard = hash(document_id) % (num_of_primary_shards)`  
3. 主分片将数据写入  
    - 当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 Memory Buffer，然后定时（默认是每隔 1 秒）写入到 Filesystem Cache，这个从 Memory Buffer 到 Filesystem Cache 的过程就叫做 [refresh](#refresh&nbsp;操作)；
    - 当然在某些情况下，存在 Memory Buffer 和 Filesystem Cache 的数据可能会丢失， ES 是通过[ translog的机制](#translog)来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中，当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 [flush](#flush)；  
    - 在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync 将创建一个新的提交点，并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。  
    - flush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512M）时；  
4. 主分片将数据发送给副本  
    - 一致性参数设置  
        - one  
            - 只需要主分片成功写入  
        - all  
            - 所有主分片和副本都要写入成功  
        - quorum  
            - 超过半数分片写入  
5. 副本保存数据然后返回  
6. 主分片返回  
7. 协调节点返回客户端
### 更新和删除文档的流程
- 删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更；  
- 磁盘上的每个段都有一个相应的.del 文件。当删除请求发送后，文档并没有真的被删除，而是在.del文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当[段合并](#段合并)时，在.del 文件中被标记为删除的文档将不会被写入新段。  
- 在新的文档被创建时， Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉。
### 搜索流程
搜索被执行成一个两阶段过程，我们称之为 Query Then Fetch；  
- 在初始查询阶段时，查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。 每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列。 PS：在搜索的时候是会查询Filesystem Cache 的，但是有部分数据还在 Memory Buffer，所以搜索是近实时的。  
- 每个分片返回各自优先队列中 所有文档的 ID 和排序值 给协调节点，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。  
- 接下来就是取回阶段， 协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每个分片加载并丰富文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了，协调节点返回结果给客户端。  
- Query Then Fetch 的搜索类型在文档相关性打分的时候参考的是本分片的数据，这样在文档数量较少的时候可能不够准确， DFS Query Then Fetch 增加了一个预查询的处理，询问 Term 和 Document frequency，这个评分更准确，但是性能会变差。
## 为什么说ES是近实时搜索? ::
### refresh 操作

主分片先将数据写入 Memory Buffer，然后定时（默认每隔1s）将 Memory Buffer 中的数据写入一个新的 segment 文件中，并进入 Filesystem Cache（同时清空 Memory Buffer），这个过程就叫做 refresh；

每个 Segment 文件实际上是一些倒排索引的集合， 只有经历了 refresh 操作之后，这些数据才能变成可检索的。  

**ES 的近实时性：当数据存在 Memory Buffer 时是搜索不到的，只有数据被 refresh 到 Filesystem Cache 之后才能被搜索到，而 refresh 是每秒一次， 所以称 es 是近实时的，或者可以通过手动调用 es 的 api 触发一次 refresh 操作，让数据马上可以被搜索到；**  

Memory Buffer，也称为 Indexing Buffer，这个区域默认的内存大小是 10% heap size。
### translog

由于 Memory Buffer 和 Filesystem Cache 都是基于内存，假设服务器宕机，那么数据就会丢失，所以 ES 通过 translog 日志文件来保证数据的可靠性，在数据写入 memory buffer 的同时，将数据写入 translog 日志文件中，在机器宕机重启时，es 会从磁盘中读取 translog 日志文件中最后一个提交点 commit point 之后的数据，恢复到 Memory Buffer 和 Filesystem cache 中去。  

ES 数据丢失的问题：translog 也是先写入 Filesystem cache，然后默认每隔 5 秒刷一次到磁盘中，所以默认情况下，可能有 5 秒的数据会仅仅停留在 memory buffer 或者 translog 文件的 Filesystem cache中，而不在磁盘上，如果此时机器宕机，会丢失 5 秒钟的数据。也可以将 translog 设置成每次写操作必须是直接 fsync 到磁盘，但是性能会差很多。
### flush

不断重复上面的步骤，translog 会变得越来越大， translog 文件默认每30分钟或者阈值超过 512M ，就会触发 flush 操作，将 memory buffer 中所有的数据写入新的 Segment 文件中， 并将内存中所有的 Segment 文件全部落盘，最后清空 translog 事务日志。  
1. 将 memory buffer 中的数据 refresh 到 Filesystem Cache 中的一个新的 segment 文件中去，然后清空 Memory Buffer；  
2. 创建一个新的 commit point（提交点），同时强行将 Filesystem Cache 中目前所有的数据都 fsync 到磁盘文件中；  
3. 删除旧的 translog 日志文件并创建一个新的 translog 日志文件，此时 flush 操作完成  
        
flush 操作主要通过以下几个参数控制  
- index.translog.flush_threshold_period  
	>每隔多长时间执行一次flush，默认30m  
- index.translog.flush_threshold_size 
	>当事务日志大小到达此预设值，则执行flush，默认512mb  
- index.translog.flush_threshold_ops  
	>当事务日志累积到多少条数据后flush一次

### 段合并  
- 将多个小 segment 文件合并成一个 segment，在合并时被标识为 deleted 的 doc（或被更新文档的旧版本）不会被写入到新的 segment 中。  
- 合并完成后，然后将新的 segment 文件 flush 写入磁盘；然后创建一个新的 commit point 文件，标识所有新的 segment 文件，并排除掉旧的 segement 和已经被合并的小 segment；  
- 然后打开新 segment 文件用于搜索使用，等所有的检索请求都从小的 segment 转到 大 segment 上以后，删除旧的 segment 文件，这时候，索引里 segment 数量就下降了。
## ES如何保证读写一致？::
- 可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖，由应用层来处理具体的冲突；  
    
- 另外对于写操作，一致性级别支持 quorum/one/all，默认为 quorum，即只有当大多数分片可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，分片将会在一个不同的节点上重建。  
    
- 对于读操作，可以设置 replication 为 sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置 replication 为 async 时，也可以通过设置搜索请求参数_preference 为 primary 来查询主分片，确保文档是最新版本。
- 
## 什么是倒排索引？::
倒排索引（Inverted Index）是一种数据结构，通常用于快速查找文档中包含特定词语的情况。它将词汇表中的每个词与包含该词的文档列表相关联。

传统的索引结构是将文档编号与包含的词语映射起来。但在倒排索引中，我们反转了这种映射关系，是将词语与其关联的文档编号做映射。这样做的好处是，当我们需要查找某个词语时，可以快速获取到包含该词的文档列表，而不需要遍历所有的文档。

倒排索引主要用于全文搜索引擎中，通过倒排索引，搜索引擎可以快速找到包含用户查询关键词的文档，从而提供相关的搜索结果。

## master 选举流程  

- Elasticsearch的选主是ZenDiscovery模块负责的，主要包含Ping（节点之间通过这个RPC来发现彼此）和Unicast（单播模块包含-一个主机列表以控制哪些节点需要ping通）这两部分。  

- 对所有可以成为master的节点（node master: true）根据nodeId字典排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。  
	

- 如果对某个节点的投票数达到一定的值（可以成为master节点数n/2+1）并且该节点自己也选举自己，那这个节点就是master。否则重新选举一直到满足上述条件。