# Nacos 介绍

Nacos (Dynamic Naming and Configuration Service)是阿里巴巴开源的一个针对微服务架构中服务发现、配置管理和服务管理平台。

> Nacos就是注册中心+配置中心的组合(Nacos=Eureka+Config+Bus)

* 官网:[https://nacos.io](https://nacos.io)
* 下载地址:[https://github.com/alibaba/Nacos](https://github.com/alibaba/Nacos)

# Nacos功能特性

- 服务发现与健康检查
- 动态配置管理
- 动态DNS服务
- 服务和元数据管理(管理平台的⻆度，nacos也有一个ui⻚面，可以看到注册的 服务及其实例信息(元数据信息)等)，动态的服务权重调整、动态服务优雅下 线，都可以去做。

# Nacos 单例服务部署

- 下载解压安装包，执行命令启动(我们使用最近比较稳定的版本 nacos-server-1.2.0.tar.gz)

```bash
 linux/mac:sh startup.sh -m standalone 
 windows:cmd startup.cmd
```

- 访问nacos管理界面:[http://127.0.0.1:8848/nacos/#/login](http://127.0.0.1:8848/nacos/#/login)(默认端口8848， 账号和密码nacos/nacos)
    
    ![](https://secure2.wostatic.cn/static/3Mv7BtUMaBSBZLtdWPYb3M/image.png?auth_key=1719568334-rAQqF1BRNxjX9NsnHv9dgH-0-484bfb9627cb055d95db927732da05ae)
    

# Nacos 服务注册中心

## 服务提供者注册到Nacos

- 在父pom中引入SCA依赖

```xml
<dependencyManagement>
    <dependencies>
        <!--SCA -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.1.0.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
    <!--SCA -->
</dependencyManagement>
```

- 在服务提供者工程中引入nacos客户端依赖(注释eureka客户端)

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

- application.yml修改，添加nacos配置信息

```yaml
server:
  port: 8082
spring:
  application:
    name: lagou-service-resume
  cloud:
    nacos:
      discovery:
        # 配置nacos server地址
        server-addr: 127.0.0.1:8848
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/lagou?useUnicode=true&characterEncoding=utf8
    username: root
    password: 123456
  jpa:
    database: MySQL
    show-sql: true
        hibernate:
          naming:
            physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
management:
  endpoints:
    web:
      exposure:
        include: '*' 
```

- 启动微服务，观察nacos控制台
    
    ![](https://secure2.wostatic.cn/static/iVdEvdP2GCWR5MXYAydqEz/image.png?auth_key=1719568334-mHFks2YpRD2hGihsvBuG3r-0-9571d8f60f53326cf863801491c60a62)
    
    保护阈值:可以设置为0-1之间的浮点数，它其实是一个比例值(当前服务健康实例 数/当前服务总实例数)
    
    场景:
    
    一般流程下，nacos是服务注册中心，服务消费者要从nacos获取某一个服务的可用实例信息，对于服务实例有健康/不健康状态之分，nacos在返回给消费者实例信息的时候，会返回健康实例。这个时候在一些高并发、大流量场景下会存在一定的问题
    
    如果服务A有100个实例，98个实例都不健康了，只有2个实例是健康的，如果nacos 只返回这两个健康实例的信息的话，那么后续消费者的请求将全部被分配到这两个 实例，流量洪峰到来，2个健康的实例也扛不住了，整个服务A 就扛不住，上游的微 服务也会导致崩溃，产生雪崩效应。
    
    保护阈值的意义在于当服务A健康实例数/总实例数 < 保护阈值 的时候，说明健康实例真的不多了，这个时候保护阈值会被触发(状态true)
    
    nacos将会把该服务所有的实例信息(健康的+不健康的)全部提供给消费者，消费者可能访问到不健康的实例，请求失败，但这样也比造成雪崩要好，牺牲了一些请求，保证了整个系统的一个可用。
    
    注意:阿里内部在使用nacos的时候，也经常调整这个保护阈值参数。
    

## 服务消费者从Nacos获取服务提供者

- 同服务提供者
    
- 测试
    
    ![](https://secure2.wostatic.cn/static/35zWMmDRtFHR8T9esXZ3Mb/image.png?auth_key=1719568334-oV4bArPjufGroCfMeGeuK6-0-d24493596204eb60c04a5a74de58be06)
    
    ![](https://secure2.wostatic.cn/static/mfya28TenEiWyPwVszLQpJ/image.png?auth_key=1719568334-sf3FeMUB2sCtMccWbrWiyn-0-56575ac013d12e94b4f1b896f011b005)
    

## 负载均衡

Nacos客户端引入的时候，会关联引入Ribbon的依赖包，我们使用OpenFiegn的时候也会引入Ribbon的依赖，Ribbon包括Hystrix都按原来方式进行配置即可。此处，我们将简历微服务，又启动了一个8083端口，注册到Nacos上，便于测试负载均衡，我们通过后台也可以看出。

## Nacos 数据模型(领域模型)

Namespace命名空间、Group分组、集群这些都是为了进行归类管理，把服务和配置文件进行归类，归类之后就可以实现一定的效果，比如隔离 比如，对于服务来说，不同命名空间中的服务不能够互相访问调用

![](https://secure2.wostatic.cn/static/n8cEDcVQmMC4AtgjGydnXr/image.png?auth_key=1719568334-5YpQ1HfuCUvEbboLU1UuA2-0-5d6ac996198736ed4673d549320fa146)

- Namespace：命名空间，对不同的环境进行隔离，比如隔离开发环境、测试环境和 生产环境
- Group：分组，将若干个服务或者若干个配置集归为一组，通常习惯一个系统归为 一个组
- Service：某一个服务，比如简历微服务
- DataId：配置集或者可以认为是一个配置文件

Namespace + Group + Service 如同 Maven 中的GAV坐标，GAV坐标是为了锁定 Jar，而这里是为了锁定服务。

Namespace + Group + DataId 如同 Maven 中的GAV坐标，GAV坐标是为了锁定 Jar，而这里是为了锁定配置文件。

### 最佳实践

Nacos抽象出了Namespace、Group、Service、DataId等概念，具体代表什么取决于怎么用(非常灵活)，推荐用法如下

|概念|描述|
|---|---|
|Namespace|代表不同的环境，如开发dev、测试test、生产环境prod|
|Group|代表某项目，比如拉勾云项目|
|Service|某个项目中具体xxx服务|
|DataId|某个项目中具体的xxx配置文件|

- Nacos服务的分级模型
    
    ![](https://secure2.wostatic.cn/static/xrKvB75DEdz2gSCvT73JiF/image.png?auth_key=1719568334-t5XagLHmf2LsSqRnCrPSWr-0-6aa21be02420fc783d2962becda7e7bf)
    

## Nacos Server 数据持久化

Nacos 默认使用嵌入式数据库进行数据存储，它支持改为外部Mysql存储

- 新建数据库 nacos_config，数据库初始化脚本文件{nacoshome}/conf/nacos-mysql.sql
- 修改{nacoshome}/conf/application.properties，增加Mysql数据源配置

```.properties
spring.datasource.platform=mysql
### Count of DB:
db.num=1
### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?
characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&au
toReconnect=true
db.user=root
db.password=123456
```

## Nacos Server 集群

- 安装3个或3个以上的Nacos
    
    复制解压后的nacos文件夹，分别命名为nacos-01、nacos-02、nacos-03
    
- 修改配置文件
    
    - 同一台机器模拟，将上述三个文件夹中application.properties中的 server.port分别改为 8848、8849、8850
        
        同时给当前实例节点绑定ip，因为服务器可能绑定多个ip
        

```.properties
nacos.inetutils.ip-address=127.0.0.1

```

```
- 复制一份conf/cluster.conf.example文件，命名为cluster.conf 在配置文件中设置集群中每一个节点的信息
```

```text
# 集群节点配置 
127.0.0.1:8848 
127.0.0.1:8849 
127.0.0.1:8850
```

- 分别启动每一个实例(可以批处理脚本完成)

```bash
sh startup.sh -m cluster
```

# Nacos 配置中心

- 之前:Spring Cloud Config + Bus
    
    1. Github 上添加配置文件
    2. 创建Config Server 配置中心—>从Github上去下载配置信息
    3. 具体的微服务(最终使用配置信息的)中配置Config Client—> ConfigServer获取配置信息
- 有Nacos之后，分布式配置就简单很多。Github不需要了(配置信息直接配置在Nacos server中)，Bus也不需要了(依然可以完成动态刷新)
    
    1、去Nacos server中添加配置信息
    
    2、改造具体的微服务，使其成为Nacos Config Client，能够从Nacos Server中获取 到配置信息
    

### Nacos server 添加配置集

![](https://secure2.wostatic.cn/static/dXPnpCDEtajRdrkihD3wjo/image.png?auth_key=1719568334-22NUyB8Nbir5Ug7oeEWoA8-0-a70a444899c99b867dc9b4fb22fc6874)

Nacos 服务端已经搭建完毕，那么我们可以在我们的微服务中开启 Nacos 配置管理

1. 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

2. 微服务中如何锁定 Nacos Server 中的配置文件(dataId)  
    通过 Namespace + Group + dataId 来锁定配置文件，Namespace不指定就默认public，Group不指定就默认 DEFAULT_GROUP dataId 的完整格式如下

```text
${prefix}-${spring.profile.active}.${file-extension}
```

```
- prefix 默认为 spring.application.name 的值，也可以通过配置项spring.cloud.nacos.config.prefix 来配置。
- spring.profile.active 即为当前环境对应的 profile。 注意:当spring.profile.active 为空时，对应的连接符 - 也将不存在，dataId 的拼 接格式变成 {prefix}.{file-extension}
- file-exetension 为配置内容的数据格式，可以通过配置项 spring.cloud.nacos.config.file-extension 来配置。目前只支持 properties 和 yaml 类型。
```

```yaml
cloud:
  nacos:
    discovery:
      # 集群中各节点信息都配置在这里(域名-VIP-绑定映射到各个实例的地址信息)
      server-addr: 127.0.0.1:8848
  config:
    server-addr: 127.0.0.1:8848
    namespace: f965f7e4-7294-40cf-825c-ef363c269d37
    group: DEFAULT_GROUP
    file-extension: yaml
```

3. 通过 Spring Cloud 原生注解 @RefreshScope 实现配置自动更新

```java
 
/**
* 该类用于模拟，我们要使用共享的那些配置信息做一些事情 */
@RestController
@RequestMapping("/config")
@RefreshScope
public class ConfigController {
    // 和取本地配置信息一样 
    @Value("${lagou.message}") 
    private String lagouMessage; 
    @Value("${mysql.url}") 
    private String mysqlUrl;
    // 内存级别的配置信息
    // 数据库，redis配置信息
    @GetMapping("/viewconfig")
    public String viewconfig() {
        return "lagouMessage==>" + lagouMessage  + " mysqlUrl=>" + mysqlUrl;
    } 
}
```

思考：一个微服务希望从配置中心Nacos server中获取多个dataId的配置信息，可 以的，扩展多个dataId

```yaml
 
# nacos配置 
cloud:
  nacos:
    discovery:
      # 集群中各节点信息都配置在这里(域名-VIP-绑定映射到各个实例的地址信息)
      server-addr: 127.0.0.1:8848,127.0.0.1:8849,127.0.0.1:8850
    # nacos config 配置 
    config:
      server-addr: 127.0.0.1:8848,127.0.0.1:8849,127.0.0.1:8850 # 锁定server端的配置文件(读取它的配置项)
      namespace: 07137f0a-bf66-424b-b910-20ece612395a # 命名空间id
      group: DEFAULT_GROUP # 默认分组就是DEFAULT_GROUP，如果使用默认 分组可以不配置
      file-extension: yaml #默认properties
      # 根据规则拼接出来的dataId效果:lagou-service-resume.yaml 
      ext-config[0]:
        data-id: abc.yaml
        group: DEFAULT_GROUP
        refresh: true #开启扩展dataId的动态刷新
      ext-config[1]:
        data-id: def.yaml
        group: DEFAULT_GROUP
        refresh: true #开启扩展dataId的动态刷新
   

```

优先级：根据规则生成的dataId > 扩展的dataId(对于扩展的dataId，[n] n越大优 先级越高)

[源码分析](https://www.wolai.com/2uTkXEgNYqkr5Rm3h1AFhg)