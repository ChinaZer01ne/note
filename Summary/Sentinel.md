## P0面试优先级（2026）
- `P0-1` 限流规则：QPS/线程数、直接/关联/链路模式。
- `P0-2` 熔断降级：慢调用、异常比例、异常数三种维度。
- `P0-3` 系统保护：系统负载、入口流量、资源维度联动。
- `P0-4` 规则治理：动态推送、版本化管理、灰度生效。
- `P0-5` 实战排障：限流误伤、热点参数、规则冲突定位。

## P0常见误区修正（本页已修订）
- Sentinel 不是只做限流，降级与系统保护同样关键。
- 阈值不能拍脑袋，需要基于压测与线上基线。
- 限流后必须有降级返回，避免上游级联超时。

## Sentinel 介绍

Sentinel是一个面向云原生微服务的流量控制、熔断降级组件。

## 为什么选择Sentinel而不是Hystrix

- Sentinel 规则可动态下发，生产环境治理成本更低。
- 组件仍在持续维护，和 Spring Cloud Alibaba 体系集成更顺畅。
- 提供流控、降级、系统保护、热点参数限流等更完整能力。
## Sentinel 部署

略
# Sentinel 关键概念

|概 念 名 称|概念描述|
|---|---|
|资 源|它可以是 Java 应用程序中的任何内容，例如，由应用程序提供的服务，或 由应用程序调用的其它应用提供的服务，甚至可以是一段代码。我们请求 的API接口就是资源|
|规 则|围绕资源的实时状态设定的规则，可以包括流量控制规则、熔断降级规则 以及系统保护规则。所有规则可以动态实时调整。|

# Sentinel 流量规则模块

系统并发能力有限，比如系统A的QPS支持1个，如果太多请求过来，那么A就应该进行流量控制了，比如其他请求直接拒绝

![](https://secure2.wostatic.cn/static/3nRpd4iD7JGpCktj9jX1JR/image.png?auth_key=1719565506-dkpkGqjJNPFkT5Xjvgk5Af-0-8f92a9ead54efc287162ce6f4f679f60)

- 资源名：默认请求路径
- 针对来源：Sentinel可以针对调用者进行限流，填写微服务名称，默认default(不 区分来源)
- 阈值类型/单机阈值
    - QPS:(每秒钟请求数量)当调用该资源的QPS达到阈值时进行限流
    - 线程数：当调用该资源的线程数达到阈值的时候进行限流(线程处理请求的时候， 如果说业务逻辑执行时间很⻓，流量洪峰来临时，会耗费很多线程资源，这些线程 资源会堆积，最终可能造成服务不可用，进一步上游服务不可用，最终可能服务雪 崩)
- 是否集群：是否集群限流
- 流控模式：
    - 直接：资源调用达到限流条件时，直接限流
    - 关联：关联的资源调用达到阈值时候限流自己
    - 链路：只记录指定链路上的流量
- 流控效果：
    - 快速失败：直接失败，抛出异常
    - Warm Up：根据冷加载因子(默认3)的值，从阈值/冷加载因子，经过预热时⻓， 才达到设置的QPS阈值
    - 排队等待：匀速排队，让请求匀速通过，阈值类型必须设置为QPS，否则无效

## 流控模式之关联限流

关联的资源调用达到阈值时候限流自己，比如用户注册接口，需要调用身份证校验接口(往往身份证校验接口)，如果身份证校验接口请求达到阈值，使用关联，可以对用户注册接口进行限流。

![](https://secure2.wostatic.cn/static/mVVWhoxZXBUZxx4xEaftBg/image.png?auth_key=1719565506-4hwFEN1UQvGdUUkMjnm6VZ-0-7d4c138a1acf752df9659522100161b7)

```java
@RestController
@RequestMapping("/user")
public class UserController {
/**
* 用户注册接口 * @return */
 
@GetMapping("/register")
    public String register() {
        System.out.println("Register success!");
        return "Register success!";
    }
/**
* 验证注册身份证接口(需要调用公安户籍资源) * @return
*/
    @GetMapping("/validateID")
    public String findResumeOpenState() {
        System.out.println("validateID");
        return "ValidateID success!";
    }
}

```

模拟密集式请求/user/validateID验证接口，我们会发现/user/register接口也被限流 了

## 流控模式之链路限流

链路指的是请求链路(调用链)

链路模式下会控制该资源所在的调用链路入口的流量。需要在规则中配置入口资源，即该调用链路入口的上下文名称。

一棵典型的调用树如下图所示:(阿里云提供)

```text
         machine-root
          /           \ 
      Entrance1     Entrance2
        /              \
DefaultNode(nodeA)   DefaultNode(nodeA)
```

![](https://secure2.wostatic.cn/static/8dRCESAAEudsgJAtQuXjh7/image.png?auth_key=1719565506-s6TRFwsE8B2Kt6U8NhwJRG-0-784b1da4855d91f83e35e3a10aea1f21)

上图中来自入口 Entrance1 和 Entrance2 的请求都调用到了资源 NodeA ， Sentinel 允许只根据某个调用入口的统计信息对资源限流。比如链路模式下设置入 口资源为 Entrance1 来表示只有从入口 Entrance1 的调用才会记录到 NodeA 的限 流统计当中，而不关心经 Entrance2 到来的调用。

![](https://secure2.wostatic.cn/static/hzBei62cLbMa5L9sbAm6p5/image.png?auth_key=1719565506-3qgWemBLxPvjN7uCG6cwPQ-0-e811bc36c1f715ff9157814c620620f1)

## 流控效果之Warm up

当系统⻓期处于空闲的情况下，当流量突然增加时，直接把系统拉升到高水位可能瞬间把系统压垮，比如电商网站的秒杀模块。

通过 Warm Up 模式(预热模式)，让通过的流量缓慢增加，经过设置的预热时间 以后，到达系统处理请求速率的设定值。

Warm Up 模式默认会从设置的 QPS 阈值的 1/3 开始慢慢往上增加至 QPS 设置值。

![](https://secure2.wostatic.cn/static/oN27WoAwK4tfowiCheikQc/image.png?auth_key=1719565506-2j3rTN3852so2Hs3BHdGR9-0-0dd4f3cb945a089dbe836662c7a12fda)

## 流控效果之排队等待

排队等待模式下会严格控制请求通过的间隔时间，即请求会匀速通过，允许部分请求排队等待，通常用于消息队列削峰填谷等场景。需设置具体的超时时间，当计算的等待时间超过超时时间时请求就会被拒绝。

很多流量过来了，并不是直接拒绝请求，而是请求进行排队，一个一个匀速通过 (处理)，请求能等就等着被处理，不能等(等待时间>超时时间)就会被拒绝

例如，QPS 配置为 5，则代表请求每 200 ms 才能通过一个，多出的请求将排队等待通过。超时时间代表最大排队时间，超出最大排队时间的请求将会直接被拒绝。 排队等待模式下，QPS 设置值不要超过 1000(请求间隔 1 ms)。

# Sentinel 降级规则模块

流控是对外部来的大流量进行控制，熔断降级的视⻆是对内部问题进行处理。

Sentinel 降级会在调用链路中某个资源出现不稳定状态时(例如调用超时或异常比 例升高)，对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联错误。当资源被降级后，在接下来的降级时间窗口之内，对该资源的调用都自动熔断.

=======>>>> 这里的降级其实是Hystrix中的熔断

还记得当时Hystrix的工作流程么

![](https://secure2.wostatic.cn/static/rPBHtehyLFGoDfEDqz5mH8/image.png?auth_key=1719565506-dgeKqBrSt7PhxR357Vsbrd-0-1cf9fd784aadb91fb961f50d80571c63)

### 策略

Sentinel不会像Hystrix那样放过一个请求尝试自我修复，就是明明确确按照时间窗口来，熔断触发后，时间窗口内拒绝请求，时间窗口后就恢复。

- RT(平均响应时间 )
    
    当 1s 内持续进入 >=5 个请求，平均响应时间超过阈值(以 ms 为单位)，那么 在接下的时间窗口(以 s 为单位)之内，对这个方法的调用都会自动地熔断(抛 出 DegradeException)。注意 Sentinel 默认统计的 RT 上限是 4900 ms，超出 此阈值的都会算作 4900 ms，若需要变更此上限可以通过启动配置项 - Dcsp.sentinel.statistic.max.rt=xxx 来配置。
    
    ![](https://secure2.wostatic.cn/static/nniessFucXGkTQDpwiMnt3/image.png?auth_key=1719565506-kiggSA7wziBBqWjsuGShuP-0-f4c6e2502671bb9feabbb122a391d1e3)
    
- 异常比例
    
    当资源的每秒请求量 >= 5，并且每秒异常总数占通过量的比值超过阈值之后， 资源进入降级状态，即在接下的时间窗口(以 s 为单位)之内，对这个方法的调 用都会自动地返回。异常比率的阈值范围是 [0.0, 1.0] ，代表 0% - 100%。
    
    ![](https://secure2.wostatic.cn/static/uNqXrudiD5J3sh2B5gDDvS/image.png?auth_key=1719565506-hs85dBVkLRcA93crhG1WEn-0-925b536c21dd92c3182a6d0c7ae94ec0)
    
- 异常数
    
    当资源近 1 分钟的异常数目超过阈值之后会进行熔断。注意由于统计时间窗口 是分钟级别的，若 timeWindow 小于 60s，则结束熔断状态后仍可能再进入熔断状态。
    
    时间窗口 >= 60s
    
    ![](https://secure2.wostatic.cn/static/h6BonEtJZ1ERZ6JJSoMi6e/image.png?auth_key=1719565506-7kfWgwYJoSNDAUB8xFPE9M-0-a6c2a6f622cf622e55f796023c05b02a)
    

Sentinel 其他模块了解(略，参考课堂视频)

# Sentinel 自定义兜底逻辑

@SentinelResource注解类似于Hystrix中的@HystrixCommand注解

@SentinelResource注解中有两个属性需要我们进行区分，blockHandler属性用来指定不满足Sentinel规则的降级兜底方法，fallback属性用于指定Java运行时异常兜底方法

- 在API接口资源处配置

```java
 
/**
 * @SentinelResource
    value:定义资源名 
    blockHandlerClass:指定Sentinel规则异常兜底逻辑所在class类 
    blockHandler:指定Sentinel规则异常兜底逻辑具体哪个方法 
    fallbackClass:指定Java运行时异常兜底逻辑所在class类
    fallback:指定Java运行时异常兜底逻辑具体哪个方法 
*/
@GetMapping("/checkState/{userId}")
@SentinelResource(
    value = "findResumeOpenState",
    blockHandlerClass = SentinelFallbackClass.class, 
    blockHandler = "handleException",
    fallback ="handleError",
    fallbackClass = SentinelFallbackClass.class)
public Integer findResumeOpenState(@PathVariable Long userId) {
    // 模拟降级: 
            /*try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }*/
    // 模拟降级:异常比例
    int i = 1/0;
    Integer defaultResumeState = resumeServiceFeignClient.findDefaultResumeState(userId);
    return defaultResumeState;
}
 

```

- 自定义兜底逻辑类 注意：兜底类中的方法为static静态方法

```java
public class SentinelHandlersClass {
    // 整体要求和当时Hystrix一样，这里还需要在形参最后添加 BlockException参数，用于接收异常
    // 注意:方法是静态的
    public static Integer handleException(Long userId, BlockException blockException) {
        return -100;
    }
    public static Integer handleError(Long userId) {
        return -500; 
    }
}
```

# 基于 Nacos 实现 Sentinel 规则持久化

目前，Sentinel Dashboard中添加的规则数据存储在内存，微服务停掉规则数据就消失，在生产环境下不合适。我们可以将Sentinel规则数据持久化到Nacos配置中 心，让微服务从Nacos获取规则数据。

![](https://secure2.wostatic.cn/static/8LEMJUBbWwnqrJvoi3sTKY/image.png?auth_key=1719565508-by4BXFzSn6DSsZYzkyxPx4-0-f435bf33e94815a6441dc887e0dd51ae)

- 自动投递微服务的pom.xml中添加依赖

```xml
<!-- Sentinel支持采用 Nacos 作为规则配置数据源，引入该适配依赖 --> 
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

- 自动投递微服务的application.yml中配置Nacos数据源

```yaml
spring:
  application:
    name: lagou-service-autodeliver
  cloud:
    nacos:
      discovery:
        server-addr:
          127.0.0.1:8848,127.0.0.1:8849,127.0.0.1:8850
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080 # sentinel dashboard/console地址
        port: 8719 # sentinel会在该端⼝启动http server，那么这样的话，控制台定义的一些限流等规则才能发送传递过来，
        #如果8719端⼝被占⽤，那么会依次+1
      datasource:
    # 此处的flow为自定义数据源名 
      flow: # 流控规则
        nacos:
          server-addr: ${spring.cloud.nacos.discovery.server-addr}
          data-id: ${spring.application.name}-flow-rules 
          groupId: DEFAULT_GROUP
          data-type: json
          rule-type: flow # 类型来自RuleType类
      degrade: # 降级规则
        nacos:
          server-addr: ${spring.cloud.nacos.discovery.server-addr}
          data-id: ${spring.application.name}-degrade-rules 
          groupId: DEFAULT_GROUP
          data-type: json
          rule-type: degrade # 类型来自RuleType类

```

- Nacos Server中添加对应规则配置集(public命名空间—>DEFAULT_GROUP中 添加)
    
    流控规则配置集 lagou-service-autodeliver-flow-rules
    

```json
 
[
  {
  "resource":"findResumeOpenState",
  "limitApp":"default",
  "grade":1,
  "count":1,
  "strategy":0,
  "controlBehavior":0,
  "clusterMode":false
  } 
]
```

所有属性来自源码FlowRule类

- resource：资源名称
- limitApp：来源应用
- grade：阈值类型 0 线程数 1 QPS
- count：单机阈值
- strategy：流控模式，0 直接 1 关联 2 链路
- controlBehavior：流控效果，0 快速失败 1 Warm Up 2 排队等待
- clusterMode：true/false 是否集群

降级规则配置集 lagou-service-autodeliver-degrade-rules

```json
 
[
  {
  "resource":"findResumeOpenState",
  "grade":2,
  "count":1,
  "timeWindow":5
  } 
]

```

所有属性来自源码DegradeRule类

- resource:资源名称
- grade:降级策略 0 RT 1 异常比例 2 异常数
- count:阈值
- timeWindow:时间窗

Rule 源码体系结构

![](https://secure2.wostatic.cn/static/xiMwzC9dUrmCdxey6BJG56/image.png?auth_key=1719565508-rZdi2EV29LNTcJQ6kWFr26-0-619faa8252487b6540ad0d7c74255652)

注意

- 一个资源可以同时有多个限流规则和降级规则，所以配置集中是一个json数组
- Sentinel控制台中修改规则，仅是内存中生效，不会修改Nacos中的配置 值，重启后恢复原来的值; Nacos控制台中修改规则，不仅内存中生效，Nacos 中持久化规则也生效，重启后规则依然保持

[源码分析](https://www.wolai.com/kmVWSDuVm1igc9YbJrouSM)