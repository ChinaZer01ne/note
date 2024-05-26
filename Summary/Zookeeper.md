# 什么是ZooKeeper

Zookeeper是⼀个开源的分布式协调服务，其设计⽬标是将那些复杂的且容易出错的分布式⼀致性服务封装起来，构成⼀个⾼效可靠的原语集，并以⼀些简单的接⼝提供给⽤户使⽤。 zookeeper是⼀个典型的分布式数据⼀致性的解决⽅案，分布式应⽤程序可以基于它实现诸如数据订阅/发布、负载均衡、命名服务、集群管理、分布式锁和分布式队列等功能

## 基本概念

### 集群⻆⾊
通常在分布式系统中，构成⼀个集群的每⼀台机器都有⾃⼰的⻆⾊，最典型的集群就是Master/Slave模式（主备模式），此情况下把所有能够处理写操作的机器称为Master机器，把所有通过异步复制⽅式获取最新数据，并提供读服务的机器为Slave机器。
    
⽽在Zookeeper中，这些概念被颠覆了。它没有沿⽤传递的Master/Slave概念，⽽是引⼊了**Leader**、 **Follower**、 **Observer**三种⻆⾊。 Zookeeper集群中的所有机器通过Leader选举来选定⼀台被称为Leader的机器， Leader服务器为客户端提供读和写服务，除Leader外，其他机器包括Follower和Observer,Follower和Observer都能提供读服务，唯⼀的区别在于Observer不参与Leader选举过程，不参与写操作的过半写成功策略，因此Observer可以在不影响写性能的情况下提升集群的性能。
    
### 会话（session）
Session指客户端会话， ⼀个客户端连接是指客户端和服务端之间的⼀个TCP⻓连接， Zookeeper对外的服务端⼝默认为2181，客户端启动的时候，⾸先会与服务器建⽴⼀个TCP连接，从第⼀次连接建⽴开始，客户端会话的⽣命周期也开始了，通过这个连接，客户端能够⼼跳检测与服务器保持有效的会话，也能够向Zookeeper服务器发送请求并接受响应，同时还能够通过该连接接受来⾃服务器的Watch事件通知。
    
### 数据节点（Znode）

是指数据模型中的数据单元，我们称之为数据节点——ZNode。 ZooKeeper将所有数据存储在内存中，数据模型是⼀棵树（ZNode Tree），由斜杠（/）进⾏分割的路径，就是⼀个Znode，例如/app/path1。每个ZNode上都会保存⾃⼰的数据内容，同时还会保存⼀系列属性信息。
    
### 版本
刚刚我们提到， Zookeeper的每个Znode上都会存储数据，对于每个ZNode， Zookeeper都会为其维护⼀个叫作Stat的数据结构， Stat记录了这个ZNode的三个数据版本，分别是`version`（当前ZNode的版本）、 `cversion`（当前ZNode⼦节点的版本）、 `aversion`（当前ZNode的ACL版本）。
    
### Watcher（事件监听器）
    
Wathcer（事件监听器），是Zookeeper中⼀个很重要的特性， Zookeeper允许⽤户在指定节点上注册⼀些Watcher，并且在⼀些特定事件触发的时候， Zookeeper服务端会将事件通知到感兴趣的客户端，该机制是Zookeeper实现分布式协调服务的重要特性
    
### ACL
Zookeeper采⽤ACL（Access Control Lists）策略来进⾏权限控制，其定义了如下五种权限：
- CREATE：创建⼦节点的权限。
- READ：获取节点数据和⼦节点列表的权限。
- WRITE：更新节点数据的权限。
- DELETE：删除⼦节点的权限。
- ADMIN：设置节点ACL的权限。
    其中需要注意的是， CREATE和DELETE这两种权限都是针对⼦节点的权限控制

## 系统模型

### Znode

在ZooKeeper中，数据信息被保存在⼀个个数据节点上，这些节点被称为znode。 ZNode 是Zookeeper 中最⼩数据单位，在 ZNode 下⾯⼜可以再挂 ZNode，这样⼀层层下去就形成了⼀个层次化命名空间 ZNode 树，我们称为 ZNode Tree，它采⽤了类似⽂件系统的层级树状结构进⾏管理。⻅下图

![](znode.png)

#### 类型

- **持久性节点**
- **临时性节点**
- **顺序性节点**

在开发中在创建节点的时候通过组合可以⽣成以下四种节点类型：持久节点、持久顺序节点、临时节点、临时顺序节点。不同类型的节点则会有不同的⽣命周期

* **持久节点**： 是Zookeeper中最常⻅的⼀种节点类型，所谓持久节点，就是指节点被创建后会***⼀直存在***服务器，直到删除操作主动清除

* **持久顺序节点**： 就是***有顺序的持久节点***，节点特性和持久节点是⼀样的，只是额外特性表现在顺序上。顺序特性实质是在创建节点的时候，会在节点名后⾯加上⼀个数字后缀，来表示其顺序。

* **临时节点**： 就是***会被⾃动清理掉的节点***，它的⽣命周期和客户端会话绑在⼀起，客户端会话结束，节点会被删除掉。与持久性节点不同的是，临时节点不能创建⼦节点。

* **临时顺序节点**： 就是***有顺序的临时节点***，和持久顺序节点相同，在其创建的时候会在名字后⾯加上数字后缀

#### 事务ID

在ZooKeeper中，事务是指能够改变ZooKeeper服务器状态的操作，我们也称之为事务操作或更新操作，⼀般包括数据节点创建与删除、数据节点内容更新等操作。对于每⼀个事务请求， ZooKeeper都会为其分配⼀个全局唯⼀的事务ID，⽤ ZXID 来表示，通常是⼀个 64 位的数字。**每⼀个 ZXID 对应⼀次更新操作**，从这些ZXID中可以间接地识别出ZooKeeper处理这些更新操作请求的全局顺序

#### Znode状态信息

![](zk状态信息.png)

整个 ZNode 节点内容包括两部分：
* 节点数据内容
* 节点状态信息

图中quota 是数据内容，其他的属于状态信息。那么这些状态信息都有什么含义呢？

* cZxid 就是 Create ZXID，表示节点被创建时的事务ID。
* ctime 就是 Create Time，表示节点创建时间。
* mZxid 就是 Modified ZXID，表示节点最后⼀次被修改时的事务ID。
* mtime 就是 Modified Time，表示节点最后⼀次被修改的时间。
* pZxid 表示该节点的⼦节点列表最后⼀次被修改时的事务 ID。只有⼦节点列表变更才会更新 pZxid，⼦节点内容变更不会更新。
* cversion 表示⼦节点的版本号。
* dataVersion 表示内容版本号。
* aclVersion 标识acl版本
* ephemeralOwner 表示创建该临时节点时的会话 sessionID，如果是持久性节点那么值为 0 dataLength 表示数据⻓度。
* numChildren 表示直系⼦节点数

### Watcher

Zookeeper使⽤Watcher机制实现分布式数据的发布/订阅功能。

在 ZooKeeper 中，引⼊了 Watcher 机制来实现这种分布式的通知功能。 ZooKeeper 允许客户端向服务端注册⼀个 Watcher 监听，当服务端的⼀些指定事件触发了这个 Watcher，那么就会向指定客户端发送⼀个事件通知来实现分布式的通知功能。

整个Watcher注册与通知过程如图所示。

![](zk的watch机制.png)

Zookeeper的Watcher机制主要包括三部分。
* 客户端线程
* 客户端WatcherManager
*  Zookeeper服务器

具体⼯作流程为：**客户端在向Zookeeper服务器注册的同时，会将Watcher对象存储在客户端的WatcherManager当中。当Zookeeper服务器触发Watcher事件后，会向客户端发送通知，客户端线程从WatcherManager中取出对应的Watcher对象来执⾏回调逻辑**。

### ACL

Zookeeper作为⼀个分布式协调框架，其内部存储了分布式系统运⾏时状态的元数据，这些元数据会直接影响基于Zookeeper进⾏构造的分布式系统的运⾏状态，因此，如何保障系统中数据的安全，从⽽避免因误操作所带来的数据随意变更⽽导致的数据库异常⼗分重要，在Zookeeper中，提供了⼀套完善的ACL（Access Control List）权限控制机制来保障数据的安全。

我们可以从三个⽅⾯来理解ACL机制： 
* **权限模式（Scheme）**
* **授权对象（ID）**
* **权限（Permission）** 
  
通常使⽤"`scheme : id : permission`"来标识⼀个有效的ACL信息。

#### **权限模式： Scheme**

权限模式⽤来确定权限验证过程中使⽤的检验策略，有如下四种模式：
1. IP
	IP模式就是通过IP地址粒度来进⾏权限控制，如"ip:192.168.0.110"表示权限控制针对该IP地址，同时IP模式可以⽀持按照⽹段⽅式进⾏配置，如"ip:192.168.0.1/24"表示针对`192.168.0.*`这个⽹段进⾏权限控制。
2. Digest
	Digest是最常⽤的权限控制模式，要更符合我们对权限控制的认识，其使⽤"username:password"形式的权限标识来进⾏权限配置，便于区分不同应⽤来进⾏权限控制。 当我们通过“username:password”形式配置了权限标识后， Zookeeper会先后对其进⾏SHA-1加密和BASE64编码。
3. World
	World是⼀种最开放的权限控制模式，这种权限控制⽅式⼏乎没有任何作⽤，数据节点的访问权限对所有⽤户开放，即所有⽤户都可以在不进⾏任何权限校验的情况下操作ZooKeeper上的数据。另外， World模式也可以看作是⼀种特殊的Digest模式，它只有⼀个权限标识，即“world：anyone”。
4. Super
	Super模式，顾名思义就是超级⽤户的意思，也是⼀种特殊的Digest模式。在Super模式下，超级⽤户可以对任意ZooKeeper上的数据节点进⾏任何操作。
        
#### **授权对象： ID**
    
授权对象指的是权限赋予的⽤户或⼀个指定实体，例如 IP 地址或是机器等。在不同的权限模式下，授权对象是不同的，表中列出了各个权限模式和授权对象之间的对应关系。

| 权限模式   | 授权对象                                                                        |
| ------ | --------------------------------------------------------------------------- |
| IP     | 通常是⼀个IP地址或IP段：例如： 192.168.10.110 或192.168.10.1/24                           |
| Digest | ⾃定义，通常是username:BASE64(SHA-1(username:password))例如： zm:sdfndsllndlksfn7c=   |
| world  | ，所有客户端都拥有指定的权限。world 下只有一个 id 选项，就是 anyone，通常组合写法为`world:anyone:permissons` |
| Super  | 超级⽤户                                                                        |
| auth   | 只有经过认证的用户才拥有指定的权限。通常组合写法为`auth:user:password:permissons`                    |

#### **权限：Permission**

权限就是指那些通过权限检查后可以被允许执⾏的操作。在ZooKeeper中，所有对数据的操作权限分为

以下五⼤类：
- CREATE（C）：数据节点的创建权限，允许授权对象在该数据节点下创建⼦节点。
- DELETE（D）：⼦节点的删除权限，允许授权对象删除该数据节点的⼦节点。
- READ（R）：数据节点的读取权限，允许授权对象访问该数据节点并读取其数据内容或⼦节点列表等。
- WRITE（W）：数据节点的更新权限，允许授权对象对该数据节点进⾏更新操作。 ·
- ADMIN（A）：数据节点的管理权限，允许授权对象对该数据节点进⾏ ACL 相关的设置操作。

# Zookeeper应⽤场景

ZooKeeper是⼀个典型的发布/订阅模式的分布式数据管理与协调框架，我们可以使⽤它来进⾏分布式数据的发布与订阅。另⼀⽅⾯，通过对ZooKeeper中丰富的数据节点类型进⾏交叉使⽤，配合Watcher事件通知机制，可以⾮常⽅便地构建⼀系列分布式应⽤中都会涉及的核⼼功能，如数据发布/订阅、命名服务、集群管理、 Master选举、分布式锁和分布式队列等。那接下来就针对这些典型的分布式应⽤场景来做下介绍

## 配置中心

数据发布/订阅（Publish/Subscribe）系统，即所谓的配置中⼼，顾名思义就是发布者将数据发布到ZooKeeper的⼀个或⼀系列节点上，供订阅者进⾏数据订阅，进⽽达到动态获取数据的⽬的，实现配置信息的集中式管理和数据的动态更新。

发布/订阅系统⼀般有两种设计模式，分别是推（Push） 模式和拉（Pull） 模式。在推模式中，服务端主动将数据更新发送给所有订阅的客户端；⽽拉模式则是由客户端主动发起请求来获取最新数据，通常客户端都采⽤定时进⾏轮询拉取的⽅式。

**ZooKeeper 采⽤的是推拉相结合的⽅式：客户端向服务端注册⾃⼰需要关注的节点，⼀旦该节点的数据发⽣变更，那么服务端就会向相应的客户端发送Watcher事件通知，客户端接收到这个消息通知之后，需要主动到服务端获取最新的数据。**

如果将配置信息存放到ZooKeeper上进⾏集中管理，那么通常情况下，应⽤在启动的时候都会主动到ZooKeeper服务端上进⾏⼀次配置信息的获取，同时，在指定节点上注册⼀个Watcher监听，这样⼀来，但凡配置信息发⽣变更，服务端都会实时通知到所有订阅的客户端，从⽽达到实时获取最新配置信息的⽬的。

在我们平常的应⽤系统开发中，经常会碰到这样的需求：系统中需要使⽤⼀些通⽤的配置信息，例如机器列表信息、运⾏时的开关配置、数据库配置信息等。这些全局配置信息通常具备以下3个特性。

- 数据量通常⽐较⼩。
- 数据内容在运⾏时会发⽣动态变化。
- 集群中各机器共享，配置⼀致。

对于这类配置信息，⼀般的做法通常可以选择将其存储在本地配置⽂件或是内存变量中。⽆论采⽤哪种⽅式，其实都可以简单地实现配置管理，在集群机器规模不⼤、配置变更不是特别频繁的情况下，⽆论刚刚提到的哪种⽅式，都能够⾮常⽅便地解决配置管理的问题。但是，⼀旦机器规模变⼤，且配置信息变更越来越频繁后，我们发现依靠现有的这两种⽅式解决配置管理就变得越来越困难了。我们既希望能够快速地做到全局配置信息的变更，同时希望变更成本⾜够⼩，因此我们必须寻求⼀种更为分布式化的解决⽅案。

### 配置存储

在进⾏配置管理之前，⾸先我们需要将初始化配置信息存储到Zookeeper上去

### 配置获取

集群中每台机器在启动初始化阶段，⾸先会从上⾯提到的ZooKeeper配置节点上读取数据库信息，同时，客户端还需要在该配置节点上注册⼀个数据变更的 Watcher监听，⼀旦发⽣节点数据变更，所有订阅的客户端都能够获取到数据变更通知。

### 配置变更

在系统运⾏过程中，可能会出现需要进⾏数据库切换的情况，这个时候就需要进⾏配置变更。借助ZooKeeper，我们只需要对ZooKeeper上配置节点的内容进⾏更新， ZooKeeper就能够帮我们将数据变更的通知发送到各个客户端，每个客户端在接收到这个变更通知后，就可以重新进⾏最新数据的获取。

## 命名服务

命名服务（Name Service）也是分布式系统中⽐较常⻅的⼀类场景，是分布式系统最基本的公共服务之⼀。在分布式系统中，被命名的实体通常可以是集群中的机器、提供的服务地址或远程对象等——这些我们都可以统称它们为名字（Name），其中较为常⻅的就是⼀些分布式服务框架（如RPC、 RMI）中的服务地址列表，通过使⽤命名服务，客户端应⽤能够根据指定名字来获取资源的实体、服务地址和提供者的信息等。

ZooKeeper 提供的命名服务功能能够帮助应⽤系统通过⼀个资源引⽤的⽅式来实现对资源的定位与使⽤。另外，⼴义上命名服务的资源定位都不是真正意义的实体资源——在分布式环境中，上层应⽤仅仅需要⼀个全局唯⼀的名字，类似于数据库中的唯⼀主键。

### Zookeeper命名服务

接下来，我们结合⼀个分布式任务调度系统来看看如何使⽤ZooKeeper来实现这类全局唯⼀ID的⽣成。 之前我们已经提到，通过调⽤ZooKeeper节点创建的API接⼝可以创建⼀个**顺序节点**，并且在API返回值中会返回这个节点的完整名字。利⽤这个特性，我们就可以借助ZooKeeper来⽣成全局唯⼀的ID 了，如下图：

![](zk顺序节点命名服务.png)


说明，对于⼀个任务列表的主键，使⽤ZooKeeper⽣成唯⼀ID的基本步骤：

1. 所有客户端都会根据⾃⼰的任务类型，在指定类型的任务下⾯通过调⽤`create()`接⼝来创建⼀个顺序节点，例如创建“job-”节点。
2. 节点创建完毕后， `create()`接⼝会返回⼀个完整的节点名，例如“job-0000000003”。
3. 客户端拿到这个返回值后，拼接上 type 类型，例如“type2-job-0000000003”，这就可以作为⼀个全局唯⼀的ID了。

在ZooKeeper中，每⼀个数据节点都能够维护⼀份顺序序列，当客户端对其创建⼀个顺序⼦节点的时候 ZooKeeper 会⾃动以后缀的形式在其⼦节点上添加⼀个序号，在这个场景中就是利⽤了ZooKeeper的这个特性。

## 集群管理

在⽇常开发和运维过程中，我们经常会有类似于如下的需求：

- 如何快速的统计出当前⽣产环境下⼀共有多少台机器
- 如何快速的获取到机器上下线的情况
- 如何实时监控集群中每台主机的运⾏时状态

### 传统集群管理

在传统的基于Agent的分布式集群管理体系中，都是通过在集群中的每台机器上部署⼀个 Agent，由这个 Agent 负责主动向指定的⼀个监控中⼼系统（监控中⼼系统负责将所有数据进⾏集中处理，形成⼀系列报表，并负责实时报警，以下简称“监控中⼼”）汇报⾃⼰所在机器的状态。在集群规模适中的场景下，这确实是⼀种在⽣产实践中⼴泛使⽤的解决⽅案，能够快速有效地实现分布式环境集群监控，但是⼀旦系统的业务场景增多，集群规模变⼤之后，该解决⽅案的弊端也就显现出来了。

#### ⼤规模升级困难

以客户端形式存在的 Agent，在⼤规模使⽤后，⼀旦遇上需要⼤规模升级的情况，就⾮常麻烦，在升级成本和升级进度的控制上⾯临巨⼤的挑战。

#### 统⼀的Agent⽆法满⾜多样的需求

对于机器的CPU使⽤率、负载（Load）、内存使⽤率、⽹络吞吐以及磁盘容量等机器基本的物理状态，使⽤统⼀的Agent来进⾏监控或许都可以满⾜。但是，如果需要深⼊应⽤内部，对⼀些业务状态进⾏监控，例如，在⼀个分布式消息中间件中，希望监控到每个消费者对消息的消费状态；或者在⼀个分布式任务调度系统中，需要对每个机器上任务的执⾏情况进⾏监控。很显然，对于这些业务耦合紧密的监控需求，不适合由⼀个统⼀的Agent来提供。

#### 编程语⾔多样性

随着越来越多编程语⾔的出现，各种异构系统层出不穷。如果使⽤传统的Agent⽅式，那么需要提供各种语⾔的 Agent 客户端。另⼀⽅⾯， “监控中⼼”在对异构系统的数据进⾏整合上⾯临巨⼤挑战。

### Zookeeper的两⼤特性

1. 客户端如果对Zookeeper的数据节点注册Watcher监听，那么当该数据节点的内容或是其⼦节点列表发⽣变更时， Zookeeper服务器就会向订阅的客户端发送变更通知。
2. 对在Zookeeper上创建的临时节点，⼀旦客户端与服务器之间的会话失效，那么临时节点也会被⾃动删除

利⽤其两⼤特性，可以实现集群机器存活监控系统，若监控系统在/clusterServers节点上注册⼀个Watcher监听，那么但凡进⾏动态添加机器的操作，就会在/clusterServers节点下创建⼀个临时节点： `/clusterServers/[Hostname]`，这样，监控系统就能够实时监测机器的变动情况。

下⾯通过分布式⽇志收集系统这个典型应⽤来学习Zookeeper如何实现集群管理。

### 分布式⽇志收集系统

分布式⽇志收集系统的核⼼⼯作就是收集分布在不同机器上的系统⽇志，在这⾥我们重点来看分布式⽇志系统（以下简称“⽇志系统”）的收集器模块。

在⼀个典型的⽇志系统的架构设计中，整个⽇志系统会把所有需要收集的⽇志机器（我们以“⽇志源机器”代表此类机器）分为多个组别，每个组别对应⼀个收集器，这个收集器其实就是⼀个后台机器（我们以“收集器机器”代表此类机器），⽤于收集⽇志

对于⼤规模的分布式⽇志收集系统场景，通常需要解决两个问题：

- 变化的⽇志源机器
    
    在⽣产环境中，伴随着机器的变动，每个应⽤的机器⼏乎每天都是在变化的（机器硬件问题、扩容、机房迁移或是⽹络问题等都会导致⼀个应⽤的机器变化），也就是说每个组别中的⽇志源机器通常是在不断变化的
    
- 变化的收集器机器
    
    ⽇志收集系统⾃身也会有机器的变更或扩容，于是会出现新的收集器机器加⼊或是⽼的收集器机器退出的情况
    

⽆论是⽇志源机器还是收集器机器的变更，最终都可以归结为如何快速、合理、动态地为每个收集器分配对应的⽇志源机器。这也成为了整个⽇志系统正确稳定运转的前提，也是⽇志收集过程中最⼤的技术挑战之⼀，在这种情况下，我们就可以引⼊zookeeper了，下⾯我们就来看ZooKeeper在这个场景中的使⽤。

使⽤Zookeeper的场景步骤如下

#### ① 注册收集器机器

使⽤ZooKeeper来进⾏⽇志系统收集器的注册，典型做法是在ZooKeeper上创建⼀个节点作为收集器的根节点，例如/logs/collector（下⽂我们以“收集器节点”代表该数据节点），每个收集器机器在启动的时候，都会在收集器节点下创建⾃⼰的节点，例如`/logs/collector/[Hostname]`

![](分布式日志收集.png)

#### ② 任务分发

待所有收集器机器都创建好⾃⼰对应的节点后，系统根据收集器节点下⼦节点的个数，将所有⽇志源机器分成对应的若⼲组，然后将分组后的机器列表分别写到这些收集器机器创建的⼦节点（例如/logs/collector/host1）上去。这样⼀来，每个收集器机器都能够从⾃⼰对应的收集器节点上获取⽇志源机器列表，进⽽开始进⾏⽇志收集⼯作。

#### ③ 状态汇报

完成收集器机器的注册以及任务分发后，我们还要考虑到这些机器随时都有挂掉的可能。因此，针对这个问题，我们需要有⼀个收集器的状态汇报机制：每个收集器机器在创建完⾃⼰的专属节点后，还需要在对应的⼦节点上创建⼀个状态⼦节点，例如`/logs/collector/host1/status`，每个收集器机器都需要定期向该节点写⼊⾃⼰的状态信息。我们可以把这种策略看作是⼀种⼼跳检测机制，通常收集器机器都会在这个节点中写⼊⽇志收集进度信息。⽇志系统根据该状态⼦节点的最后更新时间来判断对应的收集器机器是否存活。

#### ④ 动态分配

如果收集器机器挂掉或是扩容了，就需要动态地进⾏收集任务的分配。在运⾏过程中，⽇志系统始终关注着`/logs/collector`这个节点下所有⼦节点的变更，⼀旦检测到有收集器机器停⽌汇报或是有新的收集器机器加⼊，就要开始进⾏任务的重新分配。⽆论是针对收集器机器停⽌汇报还是新机器加⼊的情况，⽇志系统都需要将之前分配给该收集器的所有任务进⾏转移。为了解决这个问题，通常有两种做法：

- 全局动态分配
    
    这是⼀种简单粗暴的做法，在出现收集器机器挂掉或是新机器加⼊的时候，⽇志系统需要根据新的收集器机器列表，⽴即对所有的⽇志源机器重新进⾏⼀次分组，然后将其分配给剩下的收集器机器。
    
- 局部动态分配
    
    全局动态分配⽅式虽然策略简单，但是存在⼀个问题：⼀个或部分收集器机器的变更，就会导致全局动态任务的分配，影响⾯⽐较⼤，因此⻛险也就⽐较⼤。所谓局部动态分配，顾名思义就是在⼩范围内进⾏任务的动态分配。在这种策略中，每个收集器机器在汇报⾃⼰⽇志收集状态的同时，也会把⾃⼰的负载汇报上去。请注意，这⾥提到的负载并不仅仅只是简单地指机器CPU负载（Load），⽽是⼀个对当前收集器任务执⾏的综合评估，这个评估算法和ZooKeeper本身并没有太⼤的关系，这⾥不再赘述。
    

在这种策略中，如果⼀个收集器机器挂了，那么⽇志系统就会把之前分配给这个机器的任务重新分配到那些负载较低的机器上去。同样，如果有新的收集器机器加⼊，会从那些负载⾼的机器上转移部分任务给这个新加⼊的机器。

上述步骤已经完整的说明了整个⽇志收集系统的⼯作流程，其中有两点注意事项：

* 节点类型
  在`/logs/collector`节点下创建临时节点可以很好的判断机器是否存活，但是，若机器挂了，其节点会被删除，记录在节点上的⽇志源机器列表也被清除，所以需要选择持久节点来标识每⼀台机器，同时在节点下分别创建`/logs/collector/[Hostname]/status`节点来表征每⼀个收集器机器的状态，这样，既能实现对所有机器的监控，同时机器挂掉后，依然能够将分配任务还原。
  
* ⽇志系统节点监听
  若采⽤Watcher机制，那么通知的消息量的⽹络开销⾮常⼤，需要采⽤⽇志系统主动轮询收集器节点的策略，这样可以节省⽹络流量，但是存在⼀定的延时。

## Master选举

Master选举是⼀个在分布式系统中⾮常常⻅的应⽤场景。分布式最核⼼的特性就是能够将具有独⽴计算能⼒的系统单元部署在不同的机器上，构成⼀个完整的分布式系统。⽽与此同时，实际场景中往往也需要在这些分布在不同机器上的独⽴系统单元中选出⼀个所谓的“⽼⼤”，在计算机中，我们称之为Master。

在分布式系统中， Master往往⽤来协调集群中其他系统单元，具有对分布式系统状态变更的决定权。例如，在⼀些读写分离的应⽤场景中，客户端的写请求往往是由 Master来处理的；⽽在另⼀些场景中，Master则常常负责处理⼀些复杂的逻辑，并将处理结果同步给集群中其他系统单元。 Master选举可以说是ZooKeeper最典型的应⽤场景了，接下来，我们就结合“⼀种海量数据处理与共享模型”这个具体例⼦来看看 ZooKeeper在集群Master选举中的应⽤场景。

在分布式环境中，经常会碰到这样的应⽤场景：集群中的所有系统单元需要对前端业务提供数据，⽐如⼀个商品 ID，或者是⼀个⽹站轮播⼴告的⼴告 ID（通常出现在⼀些⼴告投放系统中）等，⽽这些商品ID或是⼴告ID往往需要从⼀系列的海量数据处理中计算得到——这通常是⼀个⾮常耗费 I/O 和 CPU资源的过程。鉴于该计算过程的复杂性，如果让集群中的所有机器都执⾏这个计算逻辑的话，那么将耗费⾮常多的资源。⼀种⽐较好的⽅法就是只让集群中的部分，甚⾄只让其中的⼀台机器去处理数据计算，⼀旦计算出数据结果，就可以共享给整个集群中的其他所有客户端机器，这样可以⼤⼤减少重复劳动，提升性能。 这⾥我们以⼀个简单的⼴告投放系统后台场景为例来讲解这个模型。

![](master选举案例.png)

整个系统⼤体上可以分成客户端集群、分布式缓存系统、海量数据处理总线和 ZooKeeper四个部分

⾸先我们来看整个系统的运⾏机制。图中的Client集群每天定时会通过ZooKeeper来实现Master选举。选举产⽣Master客户端之后，这个Master就会负责进⾏⼀系列的海量数据处理，最终计算得到⼀个数据结果，并将其放置在⼀个内存/数据库中。同时， Master还需要通知集群中其他所有的客户端从这个内存/数据库中共享计算结果。

接下去，我们将重点来看 Master 选举的过程，⾸先来明确下 Master 选举的需求：在集群的所有机器中选举出⼀台机器作为Master。针对这个需求，通常情况下，我们可以选择常⻅的关系型数据库中的主键特性来实现：集群中的所有机器都向数据库中插⼊⼀条相同主键 ID 的记录，数据库会帮助我们⾃动进⾏主键冲突检查，也就是说，所有进⾏插⼊操作的客户端机器中，只有⼀台机器能够成功——那么，我们就认为向数据库中成功插⼊数据的客户端机器成为Master。

借助数据库的这种⽅案确实可⾏，依靠关系型数据库的主键特性能够很好地保证在集群中选举出唯⼀的⼀个Master。但是我们需要考虑的另⼀个问题是，如果当前选举出的Master挂了，那么该如何处理？谁来告诉我Master挂了呢？显然，关系型数据库没法通知我们这个事件。那么，如果使⽤ZooKeeper是否可以做到这⼀点呢？ 那在之前，我们介绍了ZooKeeper创建节点的API接⼝，其中⼀个重要特性便是：利⽤ZooKeeper的强⼀致性，能够很好保证在分布式⾼并发情况下节点的创建⼀定能够保证全局唯⼀性，即ZooKeeper将会保证客户端⽆法重复创建⼀个已经存在的数据节点。也就是说，如果同时有多个客户端请求创建同⼀个节点，那么最终⼀定只有⼀个客户端请求能够创建成功。利⽤这个特性，就能 很容易地在分布式环境中进⾏Master选举了。

![](zk实现master选举.png)

在这个系统中，⾸先会在 ZooKeeper 上创建⼀个⽇期节点，例如“2020-11-11”

客户端集群每天都会定时往ZooKeeper 上创建⼀个临时节点，例如/master_election/2020-11-11/binding。在这个过程中，只有⼀个客户端能够成功创建这个节点，那么这个客户端所在的机器就成为了Master。同时，其他没有在ZooKeeper上成功创建节点的客户端，都会在节点/master_election/2020-11-11 上注册⼀个⼦节点变更的 Watcher，⽤于监控当前的 Master 机器是否存活，⼀旦发现当前的 Master 挂了，那么其余的客户端将会重新进⾏Master选举。

从上⾯的讲解中，我们可以看到，如果仅仅只是想实现Master选举的话，那么其实只需要有⼀个能够保证数据唯⼀性的组件即可，例如关系型数据库的主键模型就是⾮常不错的选择。但是，如果希望能够快速地进⾏集群 Master 动态选举，那么就可以基于 ZooKeeper来实现。

## 分布式锁

分布式锁是控制分布式系统之间同步访问共享资源的⼀种⽅式。

下⾯我们来看看使⽤ZooKeeper如何实现分布式锁，这⾥主要讲解排他锁和共享锁两类分布式锁。

### 排他锁

**排他锁的核⼼是如何保证当前有且仅有⼀个事务获得锁，并且锁被释放后，所有正在等待获取锁的事务都能够被通知到**。

下⾯我们就来看看如何借助ZooKeeper实现排他锁：

#### 定义锁

在通常的Java开发编程中，有两种常⻅的⽅式可以⽤来定义锁，分别是synchronized机制和JDK5提供的 ReentrantLock。然⽽，在ZooKeeper中，没有类似于这样的API可以直接使⽤，⽽是通过 ZooKeeper上的数据节点来表示⼀个锁，例如`/exclusive_lock/lock`节点就可以被定义为⼀个锁，如图：

![](zk排他锁.png)

#### 获取锁

在需要获取排他锁时，所有的客户端都会试图通过调⽤ `create()`接⼝，在`/exclusive_lock`节点下创建**临时⼦节点**`/exclusive_lock/lock`。在前⾯，我们也介绍了， ZooKeeper 会保证在所有的客户端中，最终只有⼀个客户端能够创建成功，那么就可以认为该客户端获取了锁。同时，所有没有获取到锁的客户端就需要到`/exclusive_lock` 节点上注册⼀个⼦节点变更的Watcher监听，以便实时监听到lock节点的变更情况。

> **先创建节点，没有创建成功就注册监听**

#### 释放锁

在“定义锁”部分，我们已经提到， `/exclusive_lock/lock` 是⼀个临时节点，因此在以下两种情况下，都有可能释放锁。

- 当前获取锁的客户端机器发⽣宕机，那么ZooKeeper上的这个临时节点就会被移除。
- 正常执⾏完业务逻辑后，客户端就会主动将⾃⼰创建的临时节点删除。

⽆论在什么情况下移除了lock节点， ZooKeeper都会通知所有在`/exclusive_lock`节点上注册了⼦节点变更Watcher监听的客户端。这些客户端在接收到通知后，再次重新发起分布式锁获取，即重复“获取锁”过程。整个排他锁的获取和释放流程，如下图：

![](zk排他锁流程.png)

### 共享锁

**共享锁和排他锁最根本的区别在于，加上排他锁后，数据对象只对⼀个事务可⻅，⽽加上共享锁后，数据对所有事务都可⻅。**

下⾯我们就来看看如何借助ZooKeeper来实现共享锁。

#### 定义锁

和排他锁⼀样，同样是通过 ZooKeeper 上的数据节点来表示⼀个锁，是⼀个类似于“`/shared_lock/[Hostname]-请求类型-序号`”**的临时顺序节点**，例如`/shared_lock/host1-R-0000000001`，那么，这个节点就代表了⼀个共享锁，如图所示：

![](zk共享锁.png)

####  获取锁

在需要获取共享锁时，所有客户端都会到`/shared_lock` 这个节点下⾯创建⼀个临时顺序节点，如果当前是读请求，那么就创建例如`/shared_lock/host1-R-0000000001`的节点；如果是写请求，那么就创建例如`/shared_lock/host2-W-0000000002`的节点。

判断读写顺序

通过Zookeeper来确定分布式读写顺序，⼤致分为四步

1. 创建完节点后，获取`/shared_lock`节点下所有⼦节点，并对该节点变更注册监听。
2. 确定⾃⼰的节点序号在所有⼦节点中的顺序。
3. 对于读请求：若没有⽐⾃⼰序号⼩的⼦节点或所有⽐⾃⼰序号⼩的⼦节点都是读请求，那么表明⾃⼰已经成功获取到共享锁，同时开始执⾏读取逻辑，若有写请求，则需要等待。对于写请求：若⾃⼰不是序号最⼩的⼦节点，那么需要等待。
4. 接收到Watcher通知后，重复步骤1

#### 释放锁

其释放锁的流程与独占锁⼀致。

### ⽺群效应

上⾯讲解的这个共享锁实现，⼤体上能够满⾜⼀般的分布式集群竞争锁的需求，并且性能都还可以——这⾥说的⼀般场景是指集群规模不是特别⼤，⼀般是在10台机器以内。但是如果机器规模扩⼤之后，会有什么问题呢？我们着重来看上⾯“判断读写顺序”过程的步骤3，结合下⾯的图，看看实际运⾏中的情况。

![](zk锁的羊群效应.png)

针对如上图所示的情况进⾏分析

1. host1⾸先进⾏读操作，完成后将节点`/shared_lock/host1-R-00000001`删除。
2. 余下4台机器均收到这个节点移除的通知，然后重新从`/shared_lock`节点上获取⼀份新的⼦节点列表。
3. 每台机器判断⾃⼰的读写顺序，其中host2检测到⾃⼰序号最⼩，于是进⾏写操作，余下的机器则继续等待。
4. 继续...

可以看到， host1客户端在移除⾃⼰的共享锁后， **Zookeeper发送了⼦节点更变Watcher通知给所有机器，然⽽除了给host2产⽣影响外，对其他机器没有任何作⽤**。⼤量的Watcher通知和⼦节点列表获取两个操作会重复运⾏，这样不仅会对zookeeper服务器造成巨⼤的性能影响影响和⽹络开销，更为严重的是，如果同⼀时间有多个节点对应的客户端完成事务或是事务中断引起节点消失， **ZooKeeper服务器就会在短时间内向其余客户端发送⼤量的事件通知，这就是所谓的⽺群效应。**

上⾯这个ZooKeeper分布式共享锁实现中出现⽺群效应的根源在于，没有找准客户端真正的关注点。我们再来回顾⼀下上⾯的分布式锁竞争过程，它的核⼼逻辑在于：判断⾃⼰是否是所有⼦节点中序号最⼩的。于是，很容易可以联想到，**每个节点对应的客户端只需要关注⽐⾃⼰序号⼩的那个相关节点的变更情况就可以了——⽽不需要关注全局的⼦列表变更情况。** 可以有如下改动来避免⽺群效应。

改进后的分布式锁实现：

⾸先，我们需要肯定的⼀点是，上⾯提到的共享锁实现，从整体思路上来说完全正确。这⾥主要的改动在于：每个锁竞争者，只需要关注/shared_lock节点下序号⽐⾃⼰⼩的那个节点是否存在即可，具体实现如下。

1. 客户端调⽤create接⼝常⻅类似于`/shared_lock/[Hostname]-请求类型-序号`的临时顺序节点。
2. 客户端调⽤getChildren接⼝获取所有已经创建的⼦节点列表（不注册任何Watcher）。
3. 如果⽆法获取共享锁，就调⽤exist接⼝来对⽐⾃⼰⼩的节点注册Watcher。对于读请求：向⽐⾃⼰序号⼩的最后⼀个写请求节点注册Watcher监听。对于写请求：向⽐⾃⼰序号⼩的最后⼀个节点注册Watcher监听。
4. 等待Watcher通知，继续进⼊步骤2。

此⽅案改动主要在于：每个锁竞争者，只需要关注/shared_lock节点下序号⽐⾃⼰⼩的那个节点是否存在即可。

![](改进版zk锁.png)

注意 相信很多同学都会觉得改进后的分布式锁实现相对来说⽐较麻烦。确实如此，如同在多线程并发编程实践中，我们会去尽量缩⼩锁的范围——对于分布式锁实现的改进其实也是同样的思路。那么对于开发⼈员来说，是否必须按照改进后的思路来设计实现⾃⼰的分布式锁呢？答案是否定的。在具体的实际开发过程中，我们提倡根据具体的业务场景和集群规模来选择适合⾃⼰的分布式锁实现：在集群规模不⼤、⽹络资源丰富的情况下，第⼀种分布式锁实现⽅式是简单实⽤的选择；⽽如果集群规模达到⼀定程度，并且希望能够精细化地控制分布式锁机制，那么就可以试试改进版的分布式锁实现。

## 分布式队列

分布式队列可以简单分为两⼤类： ⼀种是常规的FIFO先⼊先出队列模型，还有⼀种是等待队列元素聚集后统⼀安排处理执⾏的Barrier模型。

### FIFO先⼊先出

FIFO（First Input First Output，先⼊先出）， FIFO 队列是⼀种⾮常典型且应⽤⼴泛的按序执⾏的队列模型：先进⼊队列的请求操作先完成后，才会开始处理后⾯的请求。

使⽤ZooKeeper实现FIFO队列，和之前提到的共享锁的实现⾮常类似。 FIFO队列就类似于⼀个全写的共享锁模型，⼤体的设计思路其实⾮常简单：所有客户端都会到`/queue_fifo` 这个节点下⾯创建⼀个临时顺序节点，例如如`/queue_fifo/host1-00000001`。

![](zk-fifo.png)

创建完节点后，根据如下4个步骤来确定执⾏顺序。

1. 通过调⽤getChildren接⼝来获取/queue_fifo节点的所有⼦节点，即获取队列中所有的元素。
2. 确定⾃⼰的节点序号在所有⼦节点中的顺序。
3. 如果⾃⼰的序号不是最⼩，那么需要等待，同时向⽐⾃⼰序号⼩的最后⼀个节点注册Watcher监听。
4. 接收到Watcher通知后，重复步骤1。

![](zk-fifo实现.png)

### Barrier：分布式屏障

Barrier在分布式系统中特指系统之间的⼀个协调条件，规定了⼀个队列的元素必须都集聚后才能统⼀进⾏安排，否则⼀直等待。这往往出现在那些⼤规模分布式并⾏计算的应⽤。

场景上：最终的合并计算需要基于很多并⾏计算的⼦结果来进⾏。这些队列其实是在 FIFO 队列的基础上进⾏了增强，⼤致的设计思想如下：开始时， /queue_barrier 节点是⼀个已经存在的默认节点，并且将其节点的数据内容赋值为⼀个数字n来代表Barrier值，例如n=10表示只有当/queue_barrier节点下的⼦节点个数达到10后，才会打开Barrier。之后，所有的客户端都会到/queue_barrie节点下创建⼀个临时节点，例如/queue_barrier/host1，如图所示。

![](zk分布式屏障.png)

创建完节点后，按照如下步骤执⾏。

1. 通过调⽤getData接⼝获取/queue_barrier节点的数据内容： 10。
2. 通过调⽤getChildren接⼝获取/queue_barrier节点下的所有⼦节点，同时注册对⼦节点变更的Watcher监听。
3. 统计⼦节点的个数。
4. 如果⼦节点个数还不⾜10个，那么需要等待。
5. 接受到Wacher通知后，重复步骤2

![](zk分布式屏障实现.png)

# Zookeeper是如何确保读请求是安全的（线性一致）？

## 为什么读请求不安全？

如果你有一个读请求，例如get请求，把它发给某一个副本而不是Leader。如果我们这么做了，对于写请求没有什么帮助，是我们将大量的读请求的负担从Leader移走了。现在对于读请求来说，有了很大的提升，因为现在，添加越多的服务器，我们可以支持越多的客户端读请求，因为我们将客户端的读请求分担到了不同的副本上。

> 读请求放到子节点上，这个节点并不一定是完整的Leader数据

所以，现在的问题是，如果我们直接将客户端的请求发送给副本，我们能得到预期的结果吗？ 是的，实时性是这里需要考虑的问题。Zookeeper作为一个类似于Raft的系统，如果客户端将请求发送给一个随机的副本，那个副本中肯定有一份Log的拷贝，这个拷贝随着Leader的执行而变化。假设这个副本有一个key-value表，当它收到一个读X的请求，在key-value表中会有X的某个数据，这个副本可以用这个数据返回给客户端。

所以，功能上来说，副本拥有可以响应来自客户端读请求的所有数据。这里的问题是，没有理由可以相信，除了Leader以外的任何一个副本的数据是最新（up to date）的。

这里有很多原因导致副本没有最新的数据，其中一个原因是，这个副本可能不在Leader所在的过半服务器中。对于Raft来说，Leader只会等待它所在的过半服务器中的其他follower对于Leader发送的AppendEntries消息的返回，之后Leader才会commit消息，并进行下一个操作。所以，如果这个副本不在过半服务器中，它或许永远也看不到写请求。又或许网络丢包了，这个副本永远没有收到这个写请求。所以，有可能Leader和过半服务器可以看见前三个请求，但是这个副本只能看见前两个请求，而错过了请求C。所以从这个副本读数据可能读到一个旧的数据。

即使这个副本看到了相应的Log条目，它可能收不到commit消息。Zookeeper的Zab与Raft非常相似，它先发出Log条目，之后，当Leader收到了过半服务器的回复，Leader会发送commit消息。然后这个副本可能没有收到这个commit消息。

最坏的情况是，我之前已经说过，这个副本可能与Leader不在一个网络分区，或者与Leader完全没有通信，作为follower，完全没有方法知道它与Leader已经失联了，并且不能收到任何消息了（心跳呢？）。

所以，如果这里不做任何改变，并且我们想构建一个线性一致的系统，尽管在性能上很有吸引力，我们不能将读请求发送给副本，在一个线性一致系统中，不允许提供旧的数据。所以，Zookeeper这里是怎么办的？

如果你看Zookeeper论文的表2，Zookeeper的读性能随着服务器数量的增加而显著的增加。所以，很明显，Zookeeper这里有一些修改使得读请求可以由其他的服务器，其他的副本来处理。那么Zookeeper是如何确保这里的读请求是安全的（线性一致）？

对的，实际上，Zookeeper并不要求返回最新的写入数据。Zookeeper的方式是，放弃线性一致性。它对于这里问题的解决方法是，不提供线性一致的读。所以，因此，Zookeeper也不用为读请求提供最新的数据。它有自己有关一致性的定义，而这个定义不是线性一致的，因此允许为读请求返回旧的数据。所以，Zookeeper这里声明，自己最开始就不支持线性一致性，来解决这里的技术问题。如果不提供这个能力，那么（为读请求返回旧数据）就不是一个bug。这实际上是一种经典的解决性能和强一致之间矛盾的方法，也就是不提供强一致。

> Zookeeper 不提供线性一致性

## 如果系统不提供线性一致性，那么系统是否还可用？

客户端发送了一个读请求，但是并没有得到当前的正确数据，也就是最新的数据，那我们为什么要相信这个系统是可用的？我们接下来看一下这个问题。

在这之前，还有问题吗？Zookeeper的确允许客户端将读请求发送给任意副本，并由副本根据自己的状态来响应读请求。副本的Log可能并没有拥有最新的条目，所以尽管系统中可能有一些更新的数据，这个副本可能还是会返回旧的数据。

# Zookeeper的一致性保证

Zookeeper的确有一些一致性的保证，用来帮助那些使用基于Zookeeper开发应用程序的人，来理解他们的应用程序，以及理解当他们运行程序时，会发生什么。与线性一致一样，这些保证与序列有关。Zookeeper有两个主要的保证，它们在论文的2.3有提及。

第一个是，**写请求是线性一致**的。

现在，你可以发现，它（Zookeeper）对于线性一致的定义与我的不太一样，因为Zookeeper只考虑写，不考虑读。这里的意思是，尽管客户端可以并发的发送写请求，然后Zookeeper表现的就像以某种顺序，一次只执行一个写请求，并且也符合写请求的实际时间。所以如果一个写请求在另一个写请求开始前就结束了，那么Zookeeper实际上也会先执行第一个写请求，再执行第二个写请求。所以，这里不包括读请求，单独看写请求是线性一致的。Zookeeper并不是一个严格的读写系统。写请求通常也会跟着读请求。对于这种混合的读写请求，任何更改状态的操作相比其他更改状态的操作，都是线性一致的。 Zookeeper的另一个保证是，任何一个客户端的请求，都会按照客户端指定的顺序来执行，论文里称之为FIFO（First In First Out）客户端序列。

这里的意思是，如果一个特定的客户端发送了一个写请求之后是一个读请求或者任意请求，那么首先，所有的写请求会以这个客户端发送的相对顺序，加入到所有客户端的写请求中（满足保证1）。所以，如果一个客户端说，先完成这个写操作，再完成另一个写操作，之后是第三个写操作，那么在最终整体的写请求的序列中，可以看到这个客户端的写请求以相同顺序出现（虽然可能不是相邻的）。所以，对于写请求，最终会以客户端确定的顺序执行。

这里实际上是服务端需要考虑的问题，因为客户端是可以发送异步的写请求，也就是说客户端可以发送多个写请求给Zookeeper Leader节点，而不用等任何一个请求完成。Zookeeper论文并没有明确说明，但是可以假设，为了让Leader可以实际的按照客户端确定的顺序执行写请求，我设想，客户端实际上会对它的写请求打上序号，表明它先执行这个，再执行这个，第三个是这个，而Zookeeper Leader节点会遵从这个顺序。这里由于有这些异步的写请求变得非常有意思。 对于读请求，这里会更加复杂一些。我之前说过，读请求不需要经过Leader，只有写请求经过Leader，读请求只会到达某个副本。所以，读请求只能看到那个副本的Log对应的状态。对于读请求，我们应该这么考虑FIFO客户端序列，客户端会以某种顺序读某个数据，之后读第二个数据，之后是第三个数据，对于那个副本上的Log来说，每一个读请求必然要在Log的某个特定的点执行，或者说每个读请求都可以在Log一个特定的点观察到对应的状态。

然后，后续的读请求，必须要在不早于当前读请求对应的Log点执行。也就是一个客户端发起了两个读请求，如果第一个读请求在Log中的一个位置执行，那么第二个读请求只允许在第一个读请求对应的位置或者更后的位置执行。第二个读请求不允许看到之前的状态，第二个读请求至少要看到第一个读请求的状态。这是一个极其重要的事实，我们会用它来实现正确的Zookeeper应用程序。

这里特别有意思的是，如果一个客户端正在与一个副本交互，客户端发送了一些读请求给这个副本，之后这个副本故障了，客户端需要将读请求发送给另一个副本。这时，尽管客户端切换到了一个新的副本，FIFO客户端序列仍然有效。所以这意味着，如果你知道在故障前，客户端在一个副本执行了一个读请求并看到了对应于Log中这个点的状态，

当客户端切换到了一个新的副本并且发起了另一个读请求，假设之前的读请求在这里执行，那么尽管客户端切换到了一个新的副本，客户端的在新的副本的读请求，必须在Log这个点或者之后的点执行。

这里工作的原理是，**每个Log条目都会被Leader打上zxid的标签，这些标签就是Log对应的条目号。任何时候一个副本回复一个客户端的读请求，首先这个读请求是在Log的某个特定点执行的，其次回复里面会带上zxid，对应的就是Log中执行点的前一条Log条目号。客户端会记住最高的zxid，当客户端发出一个请求到一个相同或者不同的副本时，它会在它的请求中带上这个最高的zxid。这样，其他的副本就知道，应该至少在Log中这个点或者之后执行这个读请求。这里有个有趣的场景，如果第二个副本并没有最新的Log，当它从客户端收到一个请求，客户端说，上一次我的读请求在其他副本Log的这个位置执行，那么在获取到对应这个位置的Log之前，这个副本不能响应客户端请求。**

我不是很清楚这里具体怎么工作，但是要么副本阻塞了对于客户端的响应，要么副本拒绝了客户端的读请求并说：我并不了解这些信息，去问问其他的副本，或者过会再来问我。

最终，如果这个副本连上了Leader，它会更新上最新的Log，到那个时候，这个副本就可以响应读请求了。好的，所以读请求都是有序的，它们的顺序与时间正相关。

更进一步，FIFO客户端请求序列是对一个客户端的所有读请求，写请求生效。所以，如果我发送一个写请求给Leader，在Leader commit这个请求之前需要消耗一些时间，所以我现在给Leader发了一个写请求，而Leader还没有处理完它，或者commit它。之后，我发送了一个读请求给某个副本。这个读请求需要暂缓一下，以确保FIFO客户端请求序列。读请求需要暂缓，直到这个副本发现之前的写请求已经执行了。这是FIFO客户端请求序列的必然结果，（对于某个特定的客户端）读写请求是线性一致的。

> zookeeper写请求是线性一致的，读请求是对某个客户端线性一致的。

最明显的理解这种行为的方式是，如果一个客户端写了一份数据，例如向Leader发送了一个写请求，之后立即读同一份数据，并将读请求发送给了某一个副本，那么客户端需要看到自己刚刚写入的值。如果我写了某个变量为17，那么我之后读这个变量，返回的不是17，这会很奇怪，这表明系统并没有执行我的请求。因为如果执行了的话，写请求应该在读请求之前执行。所以，副本必然有一些有意思的行为来暂缓客户端，比如当客户端发送一个读请求说，我上一次发送给Leader的写请求对应了zxid是多少，这个副本必须等到自己看到对应zxid的写请求再执行读请求。

## 从Zookeeper读到的数据不能保证是最新的？

从一个副本读取的或许不是最新的数据，所以Leader或许已经向过半服务器发送了C，并commit了，过半服务器也执行了这个请求。但是这个副本并不在Leader的过半服务器中，所以或许这个副本没有最新的数据。这就是Zookeeper的工作方式，它并不保证我们可以看到最新的数据。Zookeeper可以保证读写有序，但是只针对一个客户端来说。所以，如果我发送了一个写请求，之后我读取相同的数据，Zookeeper系统可以保证读请求可以读到我之前写入的数据。但是，如果你发送了一个写请求，之后我读取相同的数据，并没有保证说我可以看到你写入的数据。这就是Zookeeper可以根据副本的数量加速读请求的基础。

## 那么Zookeeper究竟是不是线性一致呢？

我认为Zookeeper不是线性一致的，但是又不是完全的非线性一致。首先，所有客户端发送的请求以一个特定的序列执行，所以，某种意义上来说，所有的**写请求是线性一致**的。同时，每一个客户端的所有请求或许也可以认为是线性一致的。尽管我不是很确定，Zookeeper的一致性保证的第二条可以理解为，**单个客户端的请求是线性一致的**。

# Zookeeper的同步操作

总的来说，Zookeeper的一致性保证没有线性一致那么好。尽管它们有一些难以理解，并且需要一些额外共识，例如，读请求可能会返回旧数据，而这在一个线性一致系统不可能发生，但是，这些保证已经足够好了，好到可以用来直观解释很多基于Zookeeper的系统。

Zookeeper有一个操作类型是sync，它本质上就是一个写请求。假设我知道你最近写了一些数据，并且我想读出你写入的数据，所以现在的场景是，我想读出Zookeeper中最新的数据。这个时候，我可以发送一个sync请求，它的效果相当于一个写请求，

所以它最终会出现在所有副本的Log中，尽管我只关心与我交互的副本，因为我需要从那个副本读出数据。接下来，在发送读请求时，我（客户端）告诉副本，在看到我上一次sync请求之前，不要返回我的读请求。

如果这里把sync看成是一个写请求，这里实际上符合了FIFO客户端请求序列，因为读请求必须至少要看到同一个客户端前一个写请求对应的状态。所以，如果我发送了一个sync请求之后，又发送了一个读请求。Zookeeper必须要向我返回至少是我发送的sync请求对应的状态。

不管怎么样，如果我需要读最新的数据，我需要发送一个sync请求，之后再发送读请求。这个读请求可以保证看到sync对应的状态，所以可以合理的认为是最新的。但是同时也要认识到，这是一个代价很高的操作，因为我们现在将一个廉价的读操作转换成了一个耗费Leader时间的sync操作。所以，如果不是必须的，那还是不要这么做。

# 就绪文件

在论文中有几个例子场景，通过Zookeeper的一致性保证可以很简答的解释它们。 首先我想介绍的是论文中2.3有关Ready file的一些设计（这里的file对应的就是论文里的znode，Zookeeper以文件目录的形式管理数据，所以每一个数据点也可以认为是一个file）。

我们假设有另外一个分布式系统，这个分布式有一个Master节点，而Master节点在Zookeeper中维护了一个配置，这个配置对应了一些file（也就是znode）。通过这个配置，描述了有关分布式系统的一些信息，例如所有worker的IP地址，或者当前谁是Master。所以，现在Master在更新这个配置，同时，或许有大量的客户端需要读取相应的配置，并且需要发现配置的每一次变化。所以，现在的问题是，尽管配置被分割成了多个file，我们还能有原子效果的更新吗？ 为什么要有原子效果的更新呢？因为只有这样，其他的客户端才能读出完整更新的配置，而不是读出更新了一半的配置。这是人们使用Zookeeper管理配置文件时的一个经典场景。

我们这里直接拷贝论文中的2.3节的内容。假设Master做了一系列写请求来更新配置，那么我们的分布式系统中的Master会以这种顺序执行写请求。首先我们假设有一些Ready file，就是以Ready为名字的file。如果Ready file存在，那么允许读这个配置。如果Ready file不存在，那么说明配置正在更新过程中，我们不应该读取配置。所以，如果Master要更新配置，那么第一件事情是删除Ready file。之后它会更新各个保存了配置的Zookeeper file（也就是znode），这里或许有很多的file。当所有组成配置的file都更新完成之后，Master会再次创建Ready file。目前为止，这里的语句都很直观，这里只有写请求，没有读请求，而Zookeeper中写请求可以确保以线性顺序执行。

为了确保这里的执行顺序，Master以某种方式为这些请求打上了tag，表明了对于这些写请求期望的执行顺序。之后Zookeeper Leader需要按照这个顺序将这些写请求加到多副本的Log中。

接下来，所有的副本会履行自己的职责，按照这里的顺序一条条执行请求。它们也会删除（自己的）Ready file，之后执行这两个写请求，最后再次创建（自己的）Ready file。所以，这里是写请求，顺序还是很直观的。

对于读请求，需要更多的思考。假设我们有一些worker节点需要读取当前的配置。我们可以假设Worker节点首先会检查Ready file是否存在。如果不存在，那么Worker节点会过一会再重试。所以，我们假设Ready file存在，并且是经历过一次重新创建。

这里的意思是，左边的都是发送给Leader的写请求，右边是一个发送给某一个与客户端交互的副本的读请求。之后，如果文件存在，那么客户端会接下来读f1和f2。

这里，有关FIFO客户端序列中有意思的地方是，如果判断Ready file的确存在，那么也是从与客户端交互的那个副本得出的判断。所以，这里通过读请求发现Ready file存在，可以说明那个副本看到了Ready file的重新创建这个请求（由Leader同步过来的）。

同时，因为后续的读请求永远不会在更早的log条目号执行，必须在更晚的Log条目号执行，所以，对于与客户端交互的副本来说，如果它的log中包含了这条创建Ready file的log，那么意味着接下来客户端的读请求只会在log中更后面的位置执行（下图中横线位置）。

所以，如果客户端看见了Ready file，那么副本接下来执行的读请求，会在Ready file重新创建的位置之后执行。这意味着，Zookeeper可以保证这些读请求看到之前对于配置的全部更新。所以，尽管Zookeeper不是完全的线性一致，但是由于写请求是线性一致的，并且读请求是随着时间在Log中单调向前的，我们还是可以得到合理的结果。

# API

[https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-09-more-replication-craq/9.1-zookeeper-api](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-09-more-replication-craq/9.1-zookeeper-api)

# 服务器启动

服务端整体架构图

![](zk架构图.png)

Zookeeper服务器的启动，⼤致可以分为以下五个步骤

1. 配置⽂件解析
2. 初始化数据管理器
3. 初始化⽹络I/O管理器
4. 数据恢复
5. 对外服务

## 单机版服务器启动

单机版服务器的启动其流程图如下

![](单机zk启动流程.png)

上图的过程可以分为预启动和初始化过程。

1. 预启动
    1. 统⼀由`QuorumPeerMain`作为启动类。⽆论单机或集群，在`zkServer.cmd`和`zkServer.sh`中都配置了`QuorumPeerMain`作为启动⼊⼝类。
    2. 解析配置⽂件`zoo.cfg`。 `zoo.cfg`配置运⾏时的基本参数，如`tickTime`、 `dataDir`、`clientPort`等参数。
    3. 创建并启动历史⽂件清理器`DatadirCleanupManager`。对事务⽇志和快照数据⽂件进⾏定时清理。
    4. 判断当前是集群模式还是单机模式启动。若是单机模式，则委托给`ZooKeeperServerMain`进⾏启动。
    5. 再次进⾏配置⽂件`zoo.cfg`的解析。
    6. 创建服务器实例`ZooKeeperServer`。 Zookeeper服务器⾸先会进⾏服务器实例的创建，然后对该服务器实例进⾏初始化，包括连接器、内存数据库、请求处理器等组件的初始化。
2. 初始化
    1. 创建服务器统计器`ServerStats`。 `ServerStats`是Zookeeper服务器运⾏时的统计器。
    2. 创建Zookeeper数据管理器`FileTxnSnapLog`。 `FileTxnSnapLog`是Zookeeper上层服务器和底层数据存储之间的对接层，提供了⼀系列操作数据⽂件的接⼝，如事务⽇志⽂件和快照数据⽂件。 Zookeeper根据`zoo.cfg`⽂件中解析出的快照数据⽬录`dataDir`和事务⽇志⽬录`dataLogDir`来创建 `FileTxnSnapLog`。
    3. 设置服务器`tickTime`和会话超时时间限制。
    4. 创建`ServerCnxnFactory`。通过配置系统属性`zookeper.serverCnxnFactory`来指定使⽤Zookeeper⾃⼰实现的NIO还是使⽤Netty框架作为Zookeeper服务端⽹络连接⼯⼚。
    5. 初始化`ServerCnxnFactory`。 Zookeeper会初始化Thread作为`ServerCnxnFactory`的主线程，然后再初始化NIO服务器。
    6. 启动`ServerCnxnFactory`主线程。进⼊Thread的run⽅法，此时服务端还不能处理客户端请求。
    7. 恢复本地数据。启动时，需要从本地快照数据⽂件和事务⽇志⽂件进⾏数据恢复。
    8. 创建并启动会话管理器。 Zookeeper会创建会话管理器SessionTracker进⾏会话管理。
    9. 初始化Zookeeper的请求处理链。 Zookeeper请求处理⽅式为责任链模式的实现。会有多个请求处理器依次处理⼀个客户端请求，在服务器启动时，会将这些请求处理器串联成⼀个请求处理链。
    10. 注册JMX服务。 Zookeeper会将服务器运⾏时的⼀些信息以JMX的⽅式暴露给外部。
    11. 注册Zookeeper服务器实例。将Zookeeper服务器实例注册给ServerCnxnFactory，之后Zookeeper就可以对外提供服务。

⾄此，单机版的Zookeeper服务器启动完毕。

## 集群服务器启动

单机和集群服务器的启动在很多地⽅是⼀致的，其流程图如下：

![](集群zk启动流程.png)

上图的过程可以分为**预启动**、**初始化**、 **Leader选举**、 Leader与Follower启动期交互、 Leader与Follower启动等过程

### 预启动

统⼀由QuorumPeerMain作为启动类。
    1. 解析配置⽂件zoo.cfg。
    2. 创建并启动历史⽂件清理器DatadirCleanupFactory。
    3. 判断当前是集群模式还是单机模式的启动。在集群模式中，在zoo.cfg⽂件中配置了多个服务器地址，可以选择集群启动。

### 初始化

1. 创建ServerCnxnFactory。
2. 初始化ServerCnxnFactory。
3. 创建Zookeeper数据管理器FileTxnSnapLog。
4. 创建QuorumPeer实例。 Quorum是集群模式下特有的对象，是Zookeeper服务器实例（ZooKeeperServer）的托管者， QuorumPeer代表了集群中的⼀台机器，在运⾏期间，QuorumPeer会不断检测当前服务器实例的运⾏状态，同时根据情况发起Leader选举。
5. 创建内存数据库ZKDatabase。 ZKDatabase负责管理ZooKeeper的所有会话记录以及DataTree和事务⽇志的存储。
6. 初始化QuorumPeer。将核⼼组件如FileTxnSnapLog、 ServerCnxnFactory、 ZKDatabase注册到QuorumPeer中，同时配置QuorumPeer的参数，如服务器列表地址、 Leader选举算法和会话超时时间限制等。
7. 恢复本地数据。
8. 启动ServerCnxnFactory主线程

### Leader选举

1. 初始化Leader选举。
    
    集群模式特有， Zookeeper⾸先会根据⾃身的服务器ID（SID）、最新的ZXID（lastLoggedZxid）和当前的服务器epoch（currentEpoch）来⽣成⼀个初始化投票，在初始化过程中，每个服务器都会给⾃⼰投票。然后，根据zoo.cfg的配置，创建相应Leader选举算法实现， Zookeeper提供了三种默认算法（LeaderElection、 AuthFastLeaderElection、FastLeaderElection），可通过zoo.cfg中的electionAlg属性来指定，但现只⽀持FastLeaderElection选举算法。在初始化阶段， Zookeeper会创建Leader选举所需的⽹络I/O层QuorumCnxManager，同时启动对Leader选举端⼝的监听，等待集群中其他服务器创建连接。
    
2. 注册JMX服务。
    
3. 检测当前服务器状态
    
    运⾏期间， QuorumPeer会不断检测当前服务器状态。在正常情况下， Zookeeper服务器的状态在LOOKING、 LEADING、 FOLLOWING/OBSERVING之间进⾏切换。在启动阶段， QuorumPeer的初始状态是LOOKING，因此开始进⾏Leader选举。
    
4. Leader选举
    
    ZooKeeper的Leader选举过程，简单地讲，就是⼀个集群中所有的机器相互之间进⾏⼀系列投票，选举产⽣最合适的机器成为Leader，同时其余机器成为Follower或是Observer的集群机器⻆⾊初始化过程。关于Leader选举算法，简⽽⾔之，就是集群中哪个机器处理的数据越新（通常我们根据每个服务器处理过的最⼤ZXID来⽐较确定其数据是否更新），其越有可能成为Leader。当然，如果集群中的所有机器处理的ZXID⼀致的话，那么SID最⼤的服务器成为Leader，其余机器称为Follower和Observer
    

### Leader和Follower启动期交互过程

到这⾥为⽌， ZooKeeper已经完成了Leader选举，并且集群中每个服务器都已经确定了⾃⼰的⻆⾊——通常情况下就分为 Leader 和 Follower 两种⻆⾊。下⾯我们来对 Leader和Follower在启动期间的交互进⾏介绍，其⼤致交互流程如图所示。

![](leader和follower交互.png)

1. 创建Leader服务器和Follower服务器。完成Leader选举后，每个服务器会根据⾃⼰服务器的⻆⾊创建相应的服务器实例，并进⼊各⾃⻆⾊的主流程。
2. Leader服务器启动Follower接收器LearnerCnxAcceptor。运⾏期间， Leader服务器需要和所有其余的服务器（统称为Learner）保持连接以确集群的机器存活情况， LearnerCnxAcceptor负责接收所有⾮Leader服务器的连接请求。
3. Learner服务器开始和Leader建⽴连接。所有Learner会找到Leader服务器，并与其建⽴连接。
4. Leader服务器创建LearnerHandler。 Leader接收到来⾃其他机器连接创建请求后，会创建⼀个LearnerHandler实例，每个LearnerHandler实例都对应⼀个Leader与Learner服务器之间的连接，其负责Leader和Learner服务器之间⼏乎所有的消息通信和数据同步。
5. 向Leader注册。 Learner完成和Leader的连接后，会向Leader进⾏注册，即将Learner服务器的基本信息（LearnerInfo），包括SID和ZXID，发送给Leader服务器。
6. Leader解析Learner信息，计算新的epoch。 Leader接收到Learner服务器基本信息后，会解析出该Learner的SID和ZXID，然后根据ZXID解析出对应的epoch_of_learner，并和当前Leader服务器的epoch_of_leader进⾏⽐较，如果该Learner的epoch_of_learner更⼤，则更新Leader的epoch_of_leader = epoch_of_learner + 1。然后LearnHandler进⾏等待，直到过半Learner已经向Leader进⾏了注册，同时更新了epoch_of_leader后， Leader就可以确定当前集群的epoch了。
7. 发送Leader状态。计算出新的epoch后， Leader会将该信息以⼀个LEADERINFO消息的形式发送给Learner，并等待Learner的响应。
8. Learner发送ACK消息。 Learner接收到LEADERINFO后，会解析出epoch和ZXID，然后向Leader反馈⼀个ACKEPOCH响应。
9. 数据同步。 Leader收到Learner的ACKEPOCH后，即可进⾏数据同步。
10. 启动Leader和Learner服务器。当有过半Learner已经完成了数据同步，那么Leader和Learner服务器实例就可以启动了

### Leader和Follower启动

1. 创建启动会话管理器。
2. 初始化Zookeeper请求处理链，集群模式的每个处理器也会在启动阶段串联请求处理链。
3. 注册JMX服务。

⾄此，集群版的Zookeeper服务器启动完毕

# 源码分析

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
## 源码分析

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

# 扩展

---

* [环境搭建](Zookeeper-extend#Zookeeper环境搭建)
* Zookeeper命令行操作
* [Java调用Zookeeper](Zookeeper-extend#Java调用Zookeeper)
