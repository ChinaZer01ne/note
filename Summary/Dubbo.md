# Dubbo 架构概述

## 什么是Dubbo

Apache Dubbo是一款高性能的Java RPC框架。其前身是阿里巴巴公司开源的一个高性能、轻量级的开源Java RPC框架，可以和Spring框架无缝集成。

## dubbo 的特性

参考官网首页 [特性一览](https://dubbo.apache.org/zh/#td-block-1)

## Dubbo 的服务治理

服务治理（SOA governance），企业为了确保项目顺利完成而实施的过程，包括最佳实践、架构原则、治理规程、规律以及其他决定性的因素。服务治理指的是用来管理SOA的采用和实现的过程。

# Dubbo 处理流程

![](https://secure2.wostatic.cn/static/m3whAKQ7oczKYQGykWmBmJ/image.png?auth_key=1719565522-ApDa9to2wCrDbvs7zPVVf-0-29e3099baf02d37f375dffc1aee0df1d)

|节点|角色名称|
|---|---|
|Provider|暴露服务的服务提供方|
|Consumer|调用远程服务的服务消费方|
|Registry|服务注册与发现的注册中心|
|Monitor|统计服务的调用次数和调用时间的监控中心|
|Container|服务运行容器，负责启动加载运行服务提供者|

调用流程:

1. 服务提供者在服务容器启动时，向注册中心注册自己提供的服务
2. 服务消费者在启动时 向注册中心订阅自己所需的服务
3. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心会基于长连接推送变更数据给消费者
4. 服务消费者从提供者地址列表中，基于软负载均衡算法选一台提供者进行调用，如果调用失败，则重新选择一台
5. 服务提供者和消费者在内存中的调用次数和调用时间定时每分钟发送给监控中心

# 服务注册中心Zookeeper

通过前面的Dubbo架构图可以看到，Registry（服务注册中心）在其中起着至关重要的作用。Dubbo官方推荐使用Zookeeper作为服务注册中心。Zookeeper 是 Apache Hadoop 的子项目，作为 Dubbo 服务的注册中心，工业强度较高，可用于生产环境，并推荐使用 。

# SPI

## SPI简介

SPI 全称为 (Service Provider Interface) ，是JDK内置的一种服务提供发现机制。 目前有不少框架用它来做服务的扩展发现，简单来说，它就是一种动态替换发现的机制。使用SPI机制的优势是实现解耦，使得第三方服务模块的装配控制逻辑与调用者的业务代码分离。

## JDK中的SPI

SPI

## Dubbo中的SPI

dubbo中大量的使用了SPI来作为扩展点，通过实现同一接口的前提下，可以进行定制自己的实现类。比如比较常见的协议，负载均衡，都可以通过SPI的方式进行定制化，自己扩展。Dubbo中已经存在的所有已经实现好的扩展点。

![](https://secure2.wostatic.cn/static/bfmLAYQNB3LvEJLoupGCmz/image.png?auth_key=1719565522-k5vkSGamjV4gYj9XpvDyon-0-1490731b52ac2ba5483ac5210689e187)

下图中则是Dubbo中默认提供的负载均衡策略。

![](https://secure2.wostatic.cn/static/nC524yZQxDe7bEJV8cjVpW/image.png?auth_key=1719565522-4ZV192Eo424ApWzri6xQc9-0-843259aa4601e1a8ccd7d0d10cc40551)

### Dubbo中扩展点使用方式

我们使用三个项目来演示Dubbo中扩展点的使用方式，一个主项目main，一个服务接口项目api，一个服务实现项目impl。

#### api项目创建

1. 导入坐标 dubbo
2. 创建接口，在接口上 使用@SPI

#### impl项目创建

1. 导入 api项目 的依赖
2. 建立实现类，为了表达支持多个实现的目的，这里分别创建两个实现。分别为HumanHelloService 和 DogHelloService 。
3. SPI进行声明操作，在 resources 目录下创建目录 META-INF/dubbo 目录，在目录下创建名称为com.lagou.dubbo.study.spi.demo.api.HelloService的文件，文件内部配置两个实现类名称和对应的全限定名：

```.properties
human=com.lagou.service.impl.HumanHelloService
dog=com.lagou.service.impl.DogHelloService
```

#### main项目创建

1. 导入坐标 接口项目 和 实现类项目
2. 创建DubboSpiMain和原先调用的方式不太相同， dubbo 有对其进行自我重新实现，需要借助ExtensionLoader，创建新的运行项目。这里demo中的示例和java中的功能相同，查询出所有的已知实现，并且调用

### dubbo自己做SPI的目的

1. JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源
2. JDK 标准的 SPI 如果有扩展点加载失败，则所有扩展点无法使用
3. dubbo提供了对扩展点包装的功能(Adaptive)，并且还支持通过set的方式对其他的扩展点进行注入

### Dubbo SPI中的Adaptive功能

Dubbo中的Adaptive功能，主要解决的问题是如何动态的选择具体的扩展点。通过`getAdaptiveExtension` 统一对指定接口对应的所有扩展点进行封装，通过URL的方式对扩展点来进行动态选择。 (dubbo中所有的注册信息都是通过URL的形式进行处理的)这里同样采用相同的方式进行实现。

1. 创建接口
    
    api中的 HelloService 扩展如下方法, 与原先类似，在sayHello中增加 Adaptive 注解，并且在参数中提供URL参数.注意这里的URL参数的类为 `org.apache.dubbo.common.URL`
    
    其中@SPI可以指定一个字符串参数，用于指明该SPI的默认实现。
    
2. 创建实现类
    
    与上面Service实现类代码相似，只需增加URL形参即可
    
3. 编写DubboAdaptiveMain
    
    最后在获取的时候方式有所改变，需要传入URL参数，并且在参数中指定具体的实现类参数
    
    如:
    

```java
public class DubboAdaptiveMain {
    public static void main(String[] args) {
        URL url = URL.valueOf("test://localhost/hello?hello.service=dog");
        final HelloService adaptiveExtension = ExtensionLoader.getExtensionLoader(HelloService.class).getAdaptiveExtension();
        adaptiveExtension.sayHello(url);
    }
}
```

注意：

- 因为在这里只是临时测试，所以为了保证URL规范，前面的信息均为测试值即可，关键的点在于hello.service 参数，这个参数的值指定的就是具体的实现方式。关于为什么叫hello.service 是因为这个接口的名称，其中后面的大写部分被dubbo自动转码为 . 分割。
- 通过 getAdaptiveExtension 来提供一个统一的类来对所有的扩展点提供支持(底层对所有的扩展点进行封装)。
- 调用时通过参数中增加 URL 对象来实现动态的扩展点使用。
- 如果URL没有提供该参数，则该方法会使用默认在 SPI 注解中声明的实现。

## Dubbo调用时拦截操作

与很多框架一样，Dubbo也存在拦截（过滤）机制，可以通过该机制在执行目标程序前后执行我们指定的代码。

Dubbo的Filter机制，是专门为服务提供方和服务消费方调用过程进行拦截设计的，每次远程方法执行，该拦截都会被执行。这样就为开发者提供了非常方便的扩展性，比如为dubbo接口实现ip白名单功能、监控功能 、日志记录等。

步骤如下：

1. 实现 `org.apache.dubbo.rpc.Filter` 接口
2. 使用 `org.apache.dubbo.common.extension.Activate` 接口进行对类进行注册，通过group 可以指定生产端消费端 如:

```java
@Activate(group = {CommonConstants.CONSUMER})
```

3. 扩展逻辑的代码实现（比如计算方法运行时间）
4. 在 META-INF.dubbo 中新建 org.apache.dubbo.rpc.Filter 文件，并将当前类的全名写入

```.properties
timerFilter=包名.过滤器的名字
```

注意：一般类似于这样的功能都是单独开发依赖的，所以在使用方的项目中只需要引入依赖，在调用接口时，该方法便会自动拦截。

# 负载均衡策略

## 负载均衡基本配置

负载均衡（Load Balance）, 其实就是将请求分摊到多个操作单元上进行执行，从而共同完成工作任务。

负载均衡策略主要用于客户端存在多个提供者时进行选择某个提供者。

在集群负载均衡时，Dubbo 提供了多种均衡策略（包括随机、轮询、最少活跃调用数、一致性Hash），缺省为random随机调用。 [参考官方文档](https://dubbo.apache.org/zh/docs/advanced/loadbalance/)

配置负载均衡策略，既可以在服务提供者一方配置，也可以在服务消费者一方配置，如下

```java
//在服务消费者一方配置负载均衡策略
@Reference(check = false,loadbalance = "random")
//在服务提供者一方配置负载均衡
@Service(loadbalance = "random")
public class HelloServiceImpl implements HelloService {
  public String sayHello(String name) {
    return "hello " + name;
  }
}

```

## 自定义负载均衡器

负载均衡器在Dubbo中的SPI接口是 `org.apache.dubbo.rpc.cluster.LoadBalance` , 可以通过实现这个接口来实现自定义的负载均衡规则。

1. 自定义负载均衡器
    
    在上一节的案例基础上创建名称为dubbo-spi-loadbalance的Maven模块，并创建负载均衡器OnlyFirstLoadbalancer。这里功能只是简单的选取所有机器中的第一个(按照字母排序 + 端口排序)。
    
2. 配置负载均衡器
    
    在dubbo-spi-loadbalance工程的 META-INF/dubbo 目录下新建org.apache.dubbo.rpc.cluster.LoadBalance 文件，并将当前类的全名写入
    

```.properties
onlyFirst=包名.负载均衡器
```

3. 在服务提供者工程实现类中编写用于测试负载均衡效果的方法，启动不同端口时，方法返回的信息不同
4. 启动多个服务，要求他们使用同一个接口注册到同一个注册中心，但是他们的dubbo通信端口不同
5. 在服务消费方指定自定义负载均衡器 onlyFirst
6. 测试自定义负载均衡的效果

# 异步调用

Dubbo不只提供了阻塞式的的同步调用，同时提供了异步调用的方式。这种方式主要应用于提供者接口响应耗时明显，消费者端可以利用调用接口的时间去做一些其他的接口调用,利用 Future 模式来异步等待和获取结果即可。这种方式可以大大的提升消费者端的利用率。 目前这种方式可以通过XML的方式进行引入。

## 异步调用实现

1. 为了能够模拟等待，通过 int timeToWait参数，标明需要休眠多少毫秒后才会进行返回。

```java
String sayHello(String name, int timeToWait);
```

2. 接口实现 为了模拟调用耗时 可以让线程等待一段时间
3. 在消费者端，配置异步调用，注意消费端默认超时时间1000毫秒 如果提供端耗时大于1000毫秒会出现超时，可以通过改变消费端的超时时间 通过timeout属性设置即可单位毫秒

```xml
<dubbo:reference id="helloService" interface="com.lagou.service.HelloService">
<dubbo:method name="sayHello" async="true" />
</dubbo:reference>
```

4. 测试，我们休眠100毫秒，然后再去进行获取结果。方法在同步调用时的返回值是空，我们可以通过 RpcContext.getContext().getFuture() 来进行获取Future对象来进行后续的结果等待操作。

## 异步调用特殊说明

需要特别说明的是，该方式的使用，请确保dubbo的版本在2.5.4及以后的版本使用。 原因在于在2.5.3及之前的版本使用的时候，会出现异步状态传递问题。

比如我们的服务调用关系是 A -> B -> C , 这时候如果A向B发起了异步请求，在错误的版本时，B向C发起的请求也会连带的产生异步请求。这是因为在底层实现层面，他是通过 RPCContext 中的attachment 实现的。在A向B发起异步请求时，会在 attachment 中增加一个异步标示字段来表明异步等待结果。B在接受到A中的请求时，会通过该字段来判断是否是异步处理。但是由于值传递问题，B向C发起时同样会将该值进行传递，导致C误以为需要异步结果，导致返回空。这个问题在2.5.4及以后的版本进行了修正。

# 线程池

## Dubbo已有线程池

dubbo在使用时，都是通过创建真实的业务线程池进行操作的。目前已知的线程池模型有两个，和java中的相互对应:

**fix**: 表示创建固定大小的线程池。也是Dubbo默认的使用方式，默认创建的执行线程数为200，并且是没有任何等待队列的。所以再极端的情况下可能会存在问题，比如某个操作大量执行时，可能存在阻塞的情况。后面也会讲相关的处理办法。

**cache**: 创建非固定大小的线程池，当线程不足时，会自动创建新的线程。但是使用这种的时候需要注意，如果突然有高TPS的请求过来，方法没有及时完成，则会造成大量的线程创建，对系统的CPU和负载都是压力，执行越多反而会拖慢整个系统。

## 自定义线程池

在真实的使用过程中可能会因为使用fix模式的线程池，导致具体某些业务场景因为线程池中的线程数量不足而产生错误，而很多业务研发是对这些无感知的，只有当出现错误的时候才会去查看告警或者通过客户反馈出现严重的问题才去查看，结果发现是线程池满了。所以可以在创建线程池的时，通过某些手段对这个线程池进行监控，这样就可以进行及时的扩缩容机器或者告警。下面的这个程序就是这样子的，会在创建线程池后进行对其监控，并且及时作出相应处理。

1. 线程池实现, 这里主要是基于对 FixedThreadPool 中的实现做扩展出线程监控的部分

```java
public class WatchingThreadPool extends FixedThreadPool implements Runnable {
    private static final Logger LOGGER = LoggerFactory.getLogger(WatchingThreadPool.class);
    private static final double ALARM_PERCENT = 0.90;
    private final Map<URL, ThreadPoolExecutor> THREAD_POOLS = new ConcurrentHashMap<>();

    public WatchingThreadPool() {
        // 每隔3秒打印线程使用情况
        Executors.newSingleThreadScheduledExecutor().scheduleWithFixedDelay(this, 1, 3, TimeUnit.SECONDS);
    }

    @Override
    public Executor getExecutor(URL url) {
        // 从父类中创建线程池
        final Executor executor = super.getExecutor(url);
        if (executor instanceof ThreadPoolExecutor) {
            THREAD_POOLS.put(url, ((ThreadPoolExecutor) executor));
        }
        return executor;
    }

    @Override
    public void run() {
        // 遍历线程池，如果超出指定的部分，进行操作，比如接入公司的告警系统或者短信平台
        for (Map.Entry<URL, ThreadPoolExecutor> entry : THREAD_POOLS.entrySet()) {
            final URL url = entry.getKey();
            final ThreadPoolExecutor executor = entry.getValue();
            // 当前执行中的线程数
            final int activeCount = executor.getActiveCount();
            // 总计线程数
            final int poolSize = executor.getCorePoolSize();
            double used = (double) activeCount / poolSize;
            final int usedNum = (int) (used * 100);
            LOGGER.info("线程池执行状态:[{}/{}]:{}%", activeCount, poolSize,
                    usedNum);
            if (used >= ALARM_PERCENT) {
                LOGGER.error("超出警戒值！host:{}, 当前已使用量:{}%, URL:{}", url.getIp(), usedNum, url);
            }
        }
    }
}

```

2. SPI声明，创建文件 META-INF/dubbo/org.apache.dubbo.common.threadpool.ThreadPool

```.properties
watching=包名.线程池名
```

3. 在服务提供方项目引入该依赖
4. 在服务提供方项目中设置使用该线程池生成器

```.properties
dubbo.provider.threadpool=watching
```

5. 接下来需要做的就是模拟整个流程，因为该线程当前是每1秒抓一次数据，所以我们需要对该方法的提供者超过1秒的时间(比如这里用休眠 Thread.sleep )，消费者则需要启动多个线程来并行执行，来模拟整个并发情况。
6. 在调用方则尝试简单通过for循环启动多个线程来执行 查看服务提供方的监控情况

# 路由规则

路由是决定一次请求中需要发往目标机器的重要判断，通过对其控制可以决定请求的目标机器。我们可以通过创建这样的规则来决定一个请求会交给哪些服务器去处理。

## 路由规则快速入门

1. 提供两个提供者(一台本机作为提供者，一台为其他的服务器)，每个提供者会在调用时可以返回不同的信息以区分提供者。
2. 针对于消费者，我们这里通过一个死循环，每次等待用户输入，再进行调用，来模拟真实的请求情况。通过调用的返回值确认具体的提供者。
3. 我们通过ipconfig来查询到我们的IP地址，并且单独启动一个客户端，来进行如下配置(这里假设我们希望隔离掉本机的请求，都发送到另外一台机器上)。

```java
public class DubboRouterMain {
    public static void main(String[] args) {
        RegistryFactory registryFactory =
        ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
        Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://127.0.0.1:2181"));
        registry.register(URL.valueOf("condition://0.0.0.0/com.lagou.service.HelloService?category=routers&force=true&dynamic=true&rule=" + URL.encode("=> host != 你的机器ip不能是127.0.0.1")));
    }
}
```

4. 通过这个程序执行后，我们就通过消费端不停的发起请求，看到真实的请求都发到了除去本机以外的另外一台机器上。

## 路由规则详解

通过上面的程序，我们实际本质上就是通过在zookeeper中保存一个节点数据，来记录路由规则。消费者会通过监听这个服务的路径，来感知整个服务的路由规则配置，然后进行适配。这里主要介绍路由配置的参数。具体请[参考文档](https://dubbo.apache.org/zh/docs/v3.0/references/features/routing-rule-deprecated/#m-zhdocsv30referencesfeaturesrouting-rule-deprecated), 这里只对关键的参数做说明。

- `route://` 表示路由规则的类型，支持条件路由规则和脚本路由规则，可扩展，必填。
    
- `0.0.0.0` 表示对所有 IP 地址生效，如果只想对某个 IP 的生效，请填入具体 IP，必填。
    
- com.lagou.service.HelloService 表示只对指定服务生效，必填。
    
- `category=routers` 表示该数据为动态配置类型，必填。
    
- `dynamic` : 是否为持久数据，当指定服务重启时是否继续生效。必填。
    
- `runtime` : 是否在设置规则时自动缓存规则，如果设置为true则会影响部分性能。
    
- `rule` : 是整个路由最关键的配置，用于配置路由规则。
    
    ... => ... 在这里 => 前面的就是表示消费者方的匹配规则，可以不填(代表全部)。 => 后方则必须填写，表示当请求过来时，如果选择提供者的配置。官方这块儿也给出了详细的示例，可以按照那里来讲。
    
    其中使用最多的便是 host 参数。 必填。
    

## 路由与上线系统结合

当公司到了一定的规模之后，一般都会有自己的上线系统，专门用于服务上线。方便后期进行维护和记录的追查。我们去想象这样的一个场景，一个dubbo的提供者要准备进行上线，一般都提供多台提供者来同时在线上提供服务。这时候一个请求刚到达一个提供者，提供者却进行了关闭操作。那么此次请求就应该认定为失败了。所以基于这样的场景，我们可以通过路由的规则，把预发布(灰度)的机器进行从机器列表中移除。并且等待一定的时间，让其把现有的请求处理完成之后再进行关闭服务。同时，在启动时，同样需要等待一定的时间，以免因为尚未重启结束，就已经注册上去。等启动到达一定时间之 后，再进行开启流量操作。

### 实现主体思路

1. 利用zookeeper的路径感知能力，在服务准备进行重启之前将当前机器的IP地址和应用名写入zookeeper。
2. 服务消费者监听该目录，读取其中需要进行关闭的应用名和机器IP列表并且保存到内存中。
3. 当前请求过来时，判断是否是请求该应用，如果是请求重启应用，则将该提供者从服务列表中移除

### 步骤

1. 引入 Curator 框架，用于方便操作Zookeeper
2. 编写Zookeeper的操作类，用于方便进行zookeeper处理
3. 编写需要进行预发布的路径管理器，用于缓存和监听所有的待灰度机器信息列表。
4. 编写路由类(实现 org.apache.dubbo.rpc.cluster.Router )，主要目的在于对ReadyRestartInstances 中的数据进行处理，并且移除路由调用列表中正在重启中的服务。
5. 由于 Router 机制比较特殊，所以需要利用一个专门的 RouterFactory 来生成，原因在于并不是所有的都需要添加路由，所以需要利用 @Activate 来锁定具体哪些服务才需要生成使用。

```java
@Activate
public class RestartingInstanceRouterFactory implements RouterFactory {
  @Override
  public Router getRouter(URL url) {
    return new RestartingInstanceRouter(url);
  }
}
```

6. 对 RouterFactory 进行注册，同样放入到META-INF/dubbo/org.apache.dubbo.rpc.cluster.RouterFactory 文件中。

```.properties
restartInstances=com.lagou.router.RestartingInstanceRouterFactory
```

7. 将dubbo-spi-router项目引入至 consumer 项目的依赖中。
8. 这时直接启动程序，还是利用上面中所写好的 consumer 程序进行执行，确认各个 provider 可以正常执行。
9. 单独写一个 main 函数来进行将某台实例设置为启动中的状态，比如这里我们认定为当前这台机器中的 service-provider 这个提供者需要进行重启操作。

```java
ReadyRestartInstances.create().addRestartingInstance("service-provider", "正在重新启动的机器IP");
```

10. 执行完成后，再次进行尝试通过 consumer 进行调用，即可看到当前这台机器没有再发送任何请求
11. 一般情况下，当机器重启到一定时间后，我们可以再通过 removeRestartingInstance 方法对这个机器设定为既可以继续执行。
12. 调用完成后，我们再次通过 consumer 去调用，即可看到已经再次恢当前机器的请求参数。

# 服务动态降级

## 什么是服务降级

服务降级，当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务有策略的降低服务级别，以释放服务器资源，保证核心任务的正常运行。

## 为什么要服务降级

而为什么要使用服务降级，这是防止分布式服务发生雪崩效应，什么是雪崩？就是蝴蝶效应，当一个请求发生超时，一直等待着服务响应，那么在高并发情况下，很多请求都是因为这样一直等着响应，直到服务资源耗尽产生宕机，而宕机之后会导致分布式其他服务调用该宕机的服务也会出现资源耗尽宕机，这样下去将导致整个分布式服务都瘫痪，这就是雪崩。

## dubbo 服务降级实现方式

#### 第一种 在 dubbo 管理控制台配置服务降级

**屏蔽和容错 **

- mock=force:return+null
    
    表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
    
- mock=fail:return+null
    
    表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。
    

![](https://secure2.wostatic.cn/static/g5atwJS84Sws2H1L8XhHva/image.png?auth_key=1719565523-rZ1pzuicGujvACd7h2h1nN-0-7b7c6c15abf8dd29135293d39272de8f)

#### 第二种 指定返回简单值或者null

```xml
<dubbo:reference id="xxService" check="false" interface="com.xx.XxService" timeout="3000" mock="return null" />
<dubbo:reference id="xxService2" check="false" interface="com.xx.XxService2" timeout="3000" mock="return 1234" />
```

如果是标注 则使用@Reference(mock="return null")、@Reference(mock="return 简单值") 也支持 @Reference(mock="force:return null")

#### 第三种 使用java代码 动态写入配置中心

```java
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension(); 
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://IP:端口"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null"));
```

#### 第四种 整合整合 hystrix 会在后期SpringCloud课程中详细讲解

# 其他

---

[开发案例](https://www.wolai.com/peVpDQj86yM2oBT6ecB6rR)

[Dubbo管理控制台 dubbo-admin](https://www.wolai.com/6YTXgoLafcAaMyDfmbzc5J)

[Dubbo源码剖析](https://www.wolai.com/xc2TK4L6YKrEkoV36RFTaa)