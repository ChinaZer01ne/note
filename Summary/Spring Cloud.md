## Eureka

### Eureka 基础架构

![](eureka架构.png)

### Eureka 交互流程及原理

Eureka 包含两个组件： **`Eureka Server`** 和 **`Eureka Client`**。
* Eureka Client是⼀个Java客户端，⽤于简化与Eureka Server的交互； 
* Eureka Server提供服务发现的能⼒，各个微服务启动时，会通过Eureka Client向Eureka Server 进⾏注册⾃⼰的信息（例如⽹络信息）， Eureka Server会存储该服务的信息；

![](eureka官网架构图.png)

- 图中`us-east-1c`、 `us-east-1d`， `us-east-1e`代表不同的区也就是不同的机房
- 图中每⼀个`Eureka Server`都是⼀个集群。
- **注册发现**：图中`Application Service`作为服务提供者向`Eureka Server`中注册服务，`Eureka Server`接受到注册事件会在集群和分区中进⾏数据同步， `Application Client`作为消费端（服务消费者）可以从`Eureka Server`中获取到服务注册信息，进⾏服务调⽤。
- **续约**：微服务启动后，会周期性地向`Eureka Server`发送⼼跳（默认周期为30秒）以续约⾃⼰的信息
- **剔除**：`Eureka Server`在⼀定时间内没有接收到某个微服务节点的⼼跳， `Eureka Server`将会注销该微服务节点（默认90秒）
- **同步**：每个`Eureka Server`同时也是`Eureka Client`，多个`Eureka Server`之间通过复制的⽅式完成服务注册列表的同步
- **缓存注册列表**：`Eureka Client`会缓存`Eureka Server`中的信息。即使所有的`Eureka Server`节点都宕掉，服务消费者依然可以使⽤缓存中的信息找到服务提供者

**Eureka通过⼼跳检测、健康检查和客户端缓存等机制，提⾼系统的灵活性、可伸缩性和可⽤性。**

### Eureka细节详解

#### Eureka元数据详解

Eureka的元数据有两种：标准元数据和⾃定义元数据。

- 标准元数据： 主机名、 IP地址、端⼝号等信息，这些信息都会被发布在服务注册表中，⽤于服务之间的调⽤。
- ⾃定义元数据： 可以使⽤eureka.instance.metadata-map配置，符合KEY/VALUE的存储格式。这 些元数据可以在远程客户端中访问。

```YAML
# 标准元数据
instance:
  prefer-ip-address: true
  metadata-map:
    # ⾃定义元数据(kv⾃定义)
    cluster: cl1
    region: rn1
```

#### Eureka客户端详解

服务提供者（也是Eureka客户端）要向Eureka Server注册服务，并完成服务续约等⼯作

##### 服务注册详解（服务提供者）

- 服务在启动时会向注册中⼼发起注册请求，携带服务元数据信息
- Eureka注册中⼼会把服务的信息保存在Map中。

##### 服务续约详解（服务提供者）

服务每隔30秒会向注册中⼼续约(⼼跳)⼀次（也称为报活），如果没有续约，租约在90秒后到期，然后服务会被失效。每隔30秒的续约操作我们称之为⼼跳检测。

```YAML
#向Eureka服务中⼼集群注册服务
eureka:
  instance:
    # 租约续约间隔时间，默认30秒
    lease-renewal-interval-in-seconds: 30
    # 租约到期，服务时效时间，默认值90秒,服务超过90秒没有发⽣⼼跳，EurekaServer会将服务从列表移除
    lease-expiration-duration-in-seconds: 90
```

##### 获取服务列表详解（服务消费者）

每隔30秒服务会从注册中⼼中拉取⼀份服务列表。

```YAML
#向Eureka服务中⼼集群注册服务
eureka:
  client:
    # 每隔多久拉取⼀次服务列表
    registry-fetch-interval-seconds: 30
```

- 服务消费者启动时，从 Eureka Server服务列表获取只读备份，缓存到本地
- 每隔30秒，会重新获取并更新数据
- 每隔30秒的时间可以通过配置`eureka.client.registry-fetch-interval-seconds`修改

#### Eureka服务端详解

##### 服务下线

- 当服务正常关闭操作时，会发送服务下线的REST请求给Eureka Server。
- 服务中⼼接受到请求后，将该服务置为下线状态

##### 失效剔除

Eureka Server会定时（间隔值是`eureka.server.eviction-interval-timer-in-ms`，默认60s）进⾏检查，如果发现实例在在⼀定时间（此值由客户端设置的`eureka.instance.lease-expiration-duration-in-seconds`定义，默认值为90s）内没有收到⼼跳，则会注销此实例。

##### ⾃我保护

服务提供者 —> 注册中⼼

定期的续约（服务提供者和注册中⼼通信），假如服务提供者和注册中⼼之间的⽹络有点问题，不代表服务提供者不可⽤，不代表服务消费者⽆法访问服务提供者。

如果在15分钟内超过85%的客户端节点都没有正常的⼼跳，那么Eureka就认为客户端与注册中⼼出现了⽹络故障， Eureka Server⾃动进⼊⾃我保护机制。

> 为什么会有⾃我保护机制？  
> 默认情况下，如果Eureka Server在⼀定时间内（默认90秒）没有接收到某个微服务实例的⼼跳， Eureka Server将会移除该实例。但是当⽹络分区故障发⽣时，微服务与Eureka Server之间⽆法正常通信，⽽微服务本身是正常运⾏的，此时不应该移除这个微服务，所以引⼊了⾃我保护机制。

当处于⾃我保护模式时

- 不会剔除任何服务实例（可能是服务提供者和EurekaServer之间⽹络问题），保证了⼤多数服务依然可⽤
- Eureka Server仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上，保证当前节点依然可⽤，当⽹络稳定时，当前Eureka Server新的注册信息会被同步到其它节点中。
- 在Eureka Server⼯程中通过`eureka.server.enable-self-preservation`配置可⽤关停⾃我保护，默认值是打开

```yaml
eureka:
  server:
    enable-self-preservation: false # 关闭⾃我保护模式（缺省为打开）
```

经验：建议⽣产环境打开⾃我保护机制

### Eureka核⼼源码剖析

#### Eureka Server启动过程

SpringCloud充分利⽤了SpringBoot的⾃动装配的特点，观察eureka-server的jar包，发现在`META-INF`下⾯有配置⽂件`spring.factories`，springboot应⽤启动时会加载`EurekaServerAutoConfiguration`⾃动配置类

##### EurekaServerAutoConfiguration类

```java
@Configuration(proxyBeanMethods = false)
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class, InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
```

首先需要有⼀个`EurekaServerMarkerConfiguration.Marker`，才能装配Eureka Server，那么这个marker其实是由`@EnableEurekaServer`注解决定的

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```

可以看到这个注解导入了一个类`EurekaServerMarkerConfiguration`，他注入了我们说的`EurekaServerMarkerConfiguration.Marker`类，也就是说只有添加了`@EnableEurekaServer`注解，才会有后续的动作，这是成为⼀个EurekaServer的前提。

```java
@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

  @Bean
  public Marker eurekaServerMarkerBean() {
    return new Marker();
  }

  class Marker {

  }

}
```

在`EurekaServerAutoConfiguration`中有一些其他配置，比如

- 控制台

```java
@Bean
@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
public EurekaController eurekaController() {
  return new EurekaController(this.applicationInfoManager);
}
```

- 对等节点感知实例注册器
    
    集群模式下注册服务使用到的注册器，EurekaServer集群中各个节点是对等的，没有主从之分

```java
@Bean
public PeerAwareInstanceRegistry peerAwareInstanceRegistry(ServerCodecs serverCodecs) {
  this.eurekaClient.getApplications(); // force initialization
  return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs, this.eurekaClient,  this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
      this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
}
```

- 注入了EurekaServer上下文

```java
@Bean
@ConditionalOnMissingBean
public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
    PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
  return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
      registry, peerEurekaNodes, this.applicationInfoManager);
}
```

```
`DefaultEurekaServerContext`创建的时候，会执行里面的initialize方法，来做一些事情。
```

```java
@PostConstruct
@Override
public void initialize() {
    logger.info("Initializing ...");
    peerEurekaNodes.start();
    try {
        registry.init(peerEurekaNodes);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    logger.info("Initialized");
}
```

```
⽽在` com.netflix.eureka.cluster.PeerEurekaNodes#start`⽅法中，创建了线程池，更新了对等节点信息，因为EurekaServer集群节点可能发生变化
```

```java
public void start() {
    // 创建了线程池
    taskExecutor = Executors.newSingleThreadScheduledExecutor(
            new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                    thread.setDaemon(true);
                    return thread;
                }
            }
    );
    try {
        updatePeerEurekaNodes(resolvePeerUrls());
        Runnable peersUpdateTask = new Runnable() {
            @Override
            public void run() {
                try {
                    // 更新对等节点信息
                    updatePeerEurekaNodes(resolvePeerUrls());
                } catch (Throwable e) {
                    logger.error("Cannot update the replica Nodes", e);
                }

            }
        };
        taskExecutor.scheduleWithFixedDelay(
                peersUpdateTask,
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                TimeUnit.MILLISECONDS
        );
    } catch (Exception e) {
        throw new IllegalStateException(e);
    }
    for (PeerEurekaNode node : peerEurekaNodes) {
        logger.info("Replica node URL:  {}", node.getServiceUrl());
    }
}
```

- 注入了EurekaServerBootStrap类，后续启动要使用该对象

```java
@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
    EurekaServerContext serverContext) {
  return new EurekaServerBootstrap(this.applicationInfoManager,
      this.eurekaClientConfig, this.eurekaServerConfig, registry,
      serverContext);
}
```

- 注册Jersey过滤器，他是一个rest框架帮我们发布restful服务接口的类似springmvc

```java
@Bean
public FilterRegistrationBean<?> jerseyFilterRegistration(
    javax.ws.rs.core.Application eurekaJerseyApp) {
  FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
  bean.setFilter(new ServletContainer(eurekaJerseyApp));
  bean.setOrder(Ordered.LOWEST_PRECEDENCE);
  bean.setUrlPatterns(Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));
  return bean;
}
```

需要注意的是，`EurekaServerAutoConfiguration`这个类还导入了一个类：`EurekaServerInitializerConfiguration`

```java
public class EurekaServerInitializerConfiguration
    implements ServletContextAware, SmartLifecycle, Ordered {
```

它实现了SmartLifecycle接口，可以在Spring容器的Bean创建完成之后做一些事情（start方法）

```java
@Override
public void start() {
  new Thread(() -> {
    try {
      // TODO: is this class even needed now?
      // 初始化EurekaServer，重点
      eurekaServerBootstrap.contextInitialized(
          EurekaServerInitializerConfiguration.this.servletContext);
      log.info("Started Eureka Server");
      // 发布事件
      publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
      // 状态属性设置
      EurekaServerInitializerConfiguration.this.running = true;
      // 发布事件
      publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
    }
    catch (Exception ex) {
      // Help!
      log.error("Could not initialize Eureka servlet context", ex);
    }
  }).start();
}
```

在 `org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#contextInitialized`中，初始化了环境信息和其他细节

```java
public void contextInitialized(ServletContext context) {
  try {
    // 初始化环境信息
    initEurekaEnvironment();
    // 初始化上下文
    initEurekaServerContext();

    context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
  }
  catch (Throwable e) {
    log.error("Cannot bootstrap eureka server :", e);
    throw new RuntimeException("Cannot bootstrap eureka server :", e);
  }
}
```

在初始化上下文`initEurekaServerContext`的时候

```java
protected void initEurekaServerContext() throws Exception {
  // For backward compatibility
  JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
      XStream.PRIORITY_VERY_HIGH);
  XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
      XStream.PRIORITY_VERY_HIGH);

  if (isAws(this.applicationInfoManager.getInfo())) {
    this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
        this.eurekaClientConfig, this.registry, this.applicationInfoManager);
    this.awsBinder.start();
  }
  // 为非IOC容器提供获取serverContext对象的接口
  EurekaServerContextHolder.initialize(this.serverContext);

  log.info("Initialized server context");

  // Copy registry from neighboring eureka node
  // 某一个server实例启动的时候，从集群中其他的server拷贝（同步）注册信息过来，每一个server对于其他server来说也是客户端
  int registryCount = this.registry.syncUp();
  // 更改实例状态为UP，对外服务
  this.registry.openForTraffic(this.applicationInfoManager, registryCount);

  // Register all monitoring statistics.
  // 注册统计器
  EurekaMonitors.registerAllStats();
}
```

- `syncUp`

```java
@Override
public int syncUp() {
    // Copy entire entry from neighboring DS node
    // 计数
    int count = 0;

    // 可能一次没连上远程server，重试
    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        if (i > 0) {
            try {
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
        // 获取到其他server的注册信息
        Applications apps = eurekaClient.getApplications();
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
                        // 把远程获取过来的注册信息注册到自己的注册表中（map）
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```

```
`com.netflix.eureka.registry.AbstractInstanceRegistry#register`
```

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    // 读锁
    read.lock();
    try {
        // registry是保存所有应⽤实例信息的Map：ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>
        // 从registry中获取当前appName的所有实例信息
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        // 注册统计+1 
        REGISTER.increment(isReplication);
        // 如果当前appName实例信息为空，新建Map
        if (gMap == null) {
            // 本地注册表是个map
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        // 获取实例的Lease租约信息
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        // 如果已经有租约，则保留最后⼀个脏时间戳⽽不覆盖它
        // （⽐较当前请求实例租约 和 已有租约 的LastDirtyTimestamp，选择靠后的）
        if (existingLease != null && (existingLease.getHolder() != null)) {
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                        " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            // The lease does not exist and hence it is a new registration
            // 如果之前不存在实例的租约，说明是新实例注册
            // expectedNumberOfRenewsPerMin期待的每分钟续约数+2（因为30s⼀个）
            // 并更新numberOfRenewsPerMinThreshold每分钟续约阀值（85%）
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        //当前实例信息放到维护注册信息的Map
        gMap.put(registrant.getId(), lease);
        // 同步维护最近注册队列
        recentRegisteredQueue.add(new Pair<Long, String>(
                System.currentTimeMillis(),
                registrant.getAppName() + "(" + registrant.getId() + ")"));
        // This is where the initial state transfer of overridden status happens
        // 如果当前实例已经维护了OverriddenStatus，将其也放到此Eureka Server的overriddenInstanceStatusMap中
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                            + "overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }
        // 根据overridden status规则，设置状态
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        // Set the status based on the overridden status rules
        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);

        // If the lease is registered with UP status, set lease service up timestamp
        // 如果租约以UP状态注册，设置租赁服务时间戳
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        // ActionType为ADD
        registrant.setActionType(ActionType.ADDED);
        // 维护recentlyChangedQueue
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        // 更新最后更新时间
        registrant.setLastUpdatedTimestamp();
        // 使当前应⽤的ResponseCache失效
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```

- `com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#openForTraffic`

```java
@Override
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    this.expectedNumberOfClientsSendingRenews = count;
    updateRenewsPerMinThreshold();
    logger.info("Got {} instances from neighboring DS node", count);
    logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
    this.startupTime = System.currentTimeMillis();
    if (count > 0) {
        this.peerInstancesTransferEmptyOnStartup = false;
    }
    DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
    boolean isAws = Name.Amazon == selfName;
    if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
        logger.info("Priming AWS connections for all replicas..");
        primeAwsReplicas(applicationInfoManager);
    }
    logger.info("Changing status to UP");
    // 实例状态变更
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    // 开启任务，默认每隔60秒进行一次服务实例失效剔除
    super.postInit();
}
```

* `postInit`

```java
protected void postInit() {
    // 失效剔除定时任务
    renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    evictionTaskRef.set(new EvictionTask());
    // 失效剔除定时任务
    evictionTimer.schedule(evictionTaskRef.get(),
            serverConfig.getEvictionIntervalTimerInMs(),
            serverConfig.getEvictionIntervalTimerInMs());
}
```

#### Eureka Server服务接⼝暴露策略

在Eureka Server启动过程中主配置类注册了Jersey框架（是⼀个发布restful⻛格接⼝的框架，类似于我们的springmvc）

```java
// 启动时注册的Jersey Filter
@Bean
public FilterRegistrationBean<?> jerseyFilterRegistration(
    javax.ws.rs.core.Application eurekaJerseyApp) {
  FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
  bean.setFilter(new ServletContainer(eurekaJerseyApp));
  bean.setOrder(Ordered.LOWEST_PRECEDENCE);
  bean.setUrlPatterns(Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

  return bean;
}
```

Jersey细节

```java
@Bean
public javax.ws.rs.core.Application jerseyApplication(Environment environment,
    ResourceLoader resourceLoader) {

  ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
      false, environment);

  // Filter to include only classes that have a particular annotation.
  // 配置Jersey注解，Path类似于springmvc中的@RequestMapping
  provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
  provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

  // Find classes in Eureka packages (or subpackages)
  // 扫描注解，指定扫描的包，会扫描包及其子包，做的事情类似于spring component scan
  // EUREKA_PACKAGES是固定的classpath下的 "com.netflix.discovery"和"com.netflix.eureka" 
  // Jersey要扫描的包，在springmvc中叫controller，jersey中对外提供接口的类叫做资源
  Set<Class<?>> classes = new HashSet<>();
  for (String basePackage : EUREKA_PACKAGES) {
    Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
    for (BeanDefinition bd : beans) {
      Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
          resourceLoader.getClassLoader());
      classes.add(cls);
    }
  }

  // Construct the Jersey ResourceConfig
  Map<String, Object> propsAndFeatures = new HashMap<>();
  propsAndFeatures.put(
      // Skip static content used by the webapp
      ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
      EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

  DefaultResourceConfig rc = new DefaultResourceConfig(classes);
  rc.setPropertiesAndFeatures(propsAndFeatures);

  return rc;
}
```

![](eureka接口.png)

这些就是使⽤Jersey发布的供Eureka Client调⽤的Restful⻛格服务接⼝（完成服务注册、⼼跳续约等接⼝）

#### Eureka Server服务注册接⼝（接受客户端注册服务）

`ApplicationResource`类的`addInstance()`⽅法中代码：`registry.register(info, "true".equals(isReplication));`

```java
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
    // validate that the instanceinfo contains all the necessary required fields
    if (isBlank(info.getId())) {
        return Response.status(400).entity("Missing instanceId").build();
    } else if (isBlank(info.getHostName())) {
        return Response.status(400).entity("Missing hostname").build();
    } else if (isBlank(info.getIPAddr())) {
        return Response.status(400).entity("Missing ip address").build();
    } else if (isBlank(info.getAppName())) {
        return Response.status(400).entity("Missing appName").build();
    } else if (!appName.equals(info.getAppName())) {
        return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
    } else if (info.getDataCenterInfo() == null) {
        return Response.status(400).entity("Missing dataCenterInfo").build();
    } else if (info.getDataCenterInfo().getName() == null) {
        return Response.status(400).entity("Missing dataCenterInfo Name").build();
    }

    // handle cases where clients may be registering with bad DataCenterInfo with missing data
    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
    if (dataCenterInfo instanceof UniqueIdentifier) {
        String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
        if (isBlank(dataCenterInfoId)) {
            boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
            if (experimental) {
                String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                return Response.status(400).entity(entity).build();
            } else if (dataCenterInfo instanceof AmazonInfo) {
                AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                if (effectiveId == null) {
                    amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                }
            } else {
                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
            }
        }
    }
    // 进行注册
    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```

`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#register` - 注册服务信息并同步到其它Eureka节点

```java
@Override
public void register(final InstanceInfo info, final boolean isReplication) {
    // 服务失效间隔
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    // 注册
    super.register(info, leaseDuration, isReplication);
    // 将信息同步给其他节点
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

- `AbstractInstanceRegistry#register()`：注册，实例信息存储到注册表是⼀个 `ConcurrentHashMap`

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    // 读锁
    read.lock();
    try {
        // registry是保存所有应⽤实例信息的Map：ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>
        // 从registry中获取当前appName的所有实例信息
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        // 注册统计+1 
        REGISTER.increment(isReplication);
        // 如果当前appName实例信息为空，新建Map
        if (gMap == null) {
            // 本地注册表是个map
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        // 获取实例的Lease租约信息
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        // 如果已经有租约，则保留最后⼀个脏时间戳⽽不覆盖它
        // （⽐较当前请求实例租约 和 已有租约 的LastDirtyTimestamp，选择靠后的）
        if (existingLease != null && (existingLease.getHolder() != null)) {
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                        " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            // The lease does not exist and hence it is a new registration
            // 如果之前不存在实例的租约，说明是新实例注册
            // expectedNumberOfRenewsPerMin期待的每分钟续约数+2（因为30s⼀个）
            // 并更新numberOfRenewsPerMinThreshold每分钟续约阀值（85%）
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        //当前实例信息放到维护注册信息的Map
        gMap.put(registrant.getId(), lease);
        // 同步维护最近注册队列
        recentRegisteredQueue.add(new Pair<Long, String>(
                System.currentTimeMillis(),
                registrant.getAppName() + "(" + registrant.getId() + ")"));
        // This is where the initial state transfer of overridden status happens
        // 如果当前实例已经维护了OverriddenStatus，将其也放到此Eureka Server的overriddenInstanceStatusMap中
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                            + "overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }
        // 根据overridden status规则，设置状态
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        // Set the status based on the overridden status rules
        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);

        // If the lease is registered with UP status, set lease service up timestamp
        // 如果租约以UP状态注册，设置租赁服务时间戳
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        // ActionType为ADD
        registrant.setActionType(ActionType.ADDED);
        // 维护recentlyChangedQueue
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        // 更新最后更新时间
        registrant.setLastUpdatedTimestamp();
        // 使当前应⽤的ResponseCache失效
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```

- `PeerAwareInstanceRegistryImpl#replicateToPeers()` ：复制到Eureka对等节点

```java
private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        // 如果是复制操作（针对当前节点， false）
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        // 如果它已经是复制，请不要再次复制，直接return
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }
        // 从peerEurekaNodes中获取对等节点信息，然后一个个同步
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            // 如果不是自己，才同步信息
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}
```

* `PeerAwareInstanceRegistryImpl#replicateInstanceActionsToPeers`

```java
private void replicateInstanceActionsToPeers(Action action, String appName,
                                                 String id, InstanceInfo info, InstanceStatus newStatus,
                                                 PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            case Cancel: // 下线
                node.cancel(appName, id);
                break;
            case Heartbeat:  // 心跳续约
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register: // 注册
                node.register(info);
                break;
            case StatusUpdate: // 状态更新
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride: // 删除OverrideStatus
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
    } finally {
        CurrentRequestVersion.remove();
    }
}
```

#### Eureka Server服务续约接⼝（接受客户端续约）

`InstanceResource`的`renewLease`⽅法中完成客户端的⼼跳（续约）处理，关键代码： `registry.renew(app.getName(), id, isFromReplicaNode)`

```java
@PUT
public Response renewLease(
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
        @QueryParam("overriddenstatus") String overriddenStatus,
        @QueryParam("status") String status,
        @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    boolean isFromReplicaNode = "true".equals(isReplication);
    // 续约调用
    boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

    // Not found in the registry, immediately ask for a register
    if (!isSuccess) {
        logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
        return Response.status(Status.NOT_FOUND).build();
    }
    // Check if we need to sync based on dirty time stamp, the client
    // instance might have changed some value
    Response response;
    if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
        response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
        // Store the overridden status since the validation found out the node that replicates wins
        if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
                && (overriddenStatus != null)
                && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
                && isFromReplicaNode) {
            registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
        }
    } else {
        response = Response.ok().build();
    }
    logger.debug("Found (Renew): {} - {}; reply status={}", app.getName(), id, response.getStatus());
    return response;
}
```

* `com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#renew`

```java
public boolean renew(final String appName, final String id, final boolean isReplication) {
    if (super.renew(appName, id, isReplication)) {
        // renew操作要同步到其他peer节点去
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}
```

renew()⽅法中—>leaseToRenew.renew()—>对最后更新时间戳进⾏更新

- `super.renew(appName, id, isReplication)`

```java
public boolean renew(String appName, String id, boolean isReplication) {
        RENEW.increment(isReplication);
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToRenew = null;
        if (gMap != null) {
            leaseToRenew = gMap.get(id);
        }
        if (leaseToRenew == null) {
            RENEW_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
            return false;
        } else {
            InstanceInfo instanceInfo = leaseToRenew.getHolder();
            if (instanceInfo != null) {
                // touchASGCache(instanceInfo.getASGName());
                InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                        instanceInfo, leaseToRenew, isReplication);
                if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                    logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                            + "; re-register required", instanceInfo.getId());
                    RENEW_NOT_FOUND.increment(isReplication);
                    return false;
                }
                if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                    logger.info(
                            "The instance status {} is different from overridden instance status {} for instance {}. "
                                    + "Hence setting the status to overridden status", instanceInfo.getStatus().name(),
                                    overriddenInstanceStatus.name(),
                                    instanceInfo.getId());
                    instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);

                }
            }
            renewsLastMin.increment();
            leaseToRenew.renew();
            return true;
        }
    }
```


* `leaseToRenew.renew(); `更新一下最后更新时间戳

```java
public void renew() {
    lastUpdateTimestamp = System.currentTimeMillis() + duration;
}
```

- `replicateInstanceActionsToPeers() `复制Instance实例操作到其它节点

```java
private void replicateInstanceActionsToPeers(Action action, String appName,
                                                 String id, InstanceInfo info, InstanceStatus newStatus,
                                                 PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register:
                node.register(info);
                break;
            case StatusUpdate:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
    } finally {
        CurrentRequestVersion.remove();
    }
}
```

#### Eureka Client注册服务

启动过程： Eureka客户端在启动时也会装载很多配置类，我们通过spring-cloudnetflix-eureka-client-2.1.0.RELEASE.jar下的spring.factories⽂件可以看到加载的配置类，主要是`EurekaClientAutoConfiguration`

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
    CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = {
    "org.springframework.cloud.netflix.eureka.config.DiscoveryClientOptionalArgsConfiguration",
    "org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
    "org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
    "org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration {
```

思考： EurekaClient启动过程要做什么事情？？？？？？

1. 读取配置⽂件
2. 启动时从EurekaServer获取服务实例信息
3. 注册⾃⼰到Eureka Server（addInstance）
4. 开启⼀些定时任务（⼼跳续约，刷新本地服务缓存列表）

##### 读取配置文件

在`EurekaClientAutoConfiguration`中注入了读取配置文件的类，读取了配置，封装成配置类

```java
@Bean
@ConditionalOnMissingBean(value = EurekaInstanceConfig.class, search = SearchStrategy.CURRENT)
public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils,
    ManagementMetadataProvider managementMetadataProvider) {
  String hostname = getProperty("eureka.instance.hostname");
  boolean preferIpAddress = Boolean
      .parseBoolean(getProperty("eureka.instance.prefer-ip-address"));
  String ipAddress = getProperty("eureka.instance.ip-address");
  boolean isSecurePortEnabled = Boolean
      .parseBoolean(getProperty("eureka.instance.secure-port-enabled"));

  String serverContextPath = env.getProperty("server.servlet.context-path", "/");
  int serverPort = Integer.parseInt(
      env.getProperty("server.port", env.getProperty("port", "8080")));

  Integer managementPort = env.getProperty("management.server.port", Integer.class);
  String managementContextPath = env
      .getProperty("management.server.servlet.context-path");
  Integer jmxPort = env.getProperty("com.sun.management.jmxremote.port",
      Integer.class);
  EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);

  instance.setNonSecurePort(serverPort);
  instance.setInstanceId(getDefaultInstanceId(env));
  instance.setPreferIpAddress(preferIpAddress);
  instance.setSecurePortEnabled(isSecurePortEnabled);
  if (StringUtils.hasText(ipAddress)) {
    instance.setIpAddress(ipAddress);
  }

  if (isSecurePortEnabled) {
    instance.setSecurePort(serverPort);
  }

  if (StringUtils.hasText(hostname)) {
    instance.setHostname(hostname);
  }
  String statusPageUrlPath = getProperty("eureka.instance.status-page-url-path");
  String healthCheckUrlPath = getProperty("eureka.instance.health-check-url-path");

  if (StringUtils.hasText(statusPageUrlPath)) {
    instance.setStatusPageUrlPath(statusPageUrlPath);
  }
  if (StringUtils.hasText(healthCheckUrlPath)) {
    instance.setHealthCheckUrlPath(healthCheckUrlPath);
  }

  ManagementMetadata metadata = managementMetadataProvider.get(instance, serverPort,
      serverContextPath, managementContextPath, managementPort);

  if (metadata != null) {
    instance.setStatusPageUrl(metadata.getStatusPageUrl());
    instance.setHealthCheckUrl(metadata.getHealthCheckUrl());
    if (instance.isSecurePortEnabled()) {
      instance.setSecureHealthCheckUrl(metadata.getSecureHealthCheckUrl());
    }
    Map<String, String> metadataMap = instance.getMetadataMap();
    metadataMap.computeIfAbsent("management.port",
        k -> String.valueOf(metadata.getManagementPort()));
  }
  else {
    // without the metadata the status and health check URLs will not be set
    // and the status page and health check url paths will not include the
    // context path so set them here
    if (StringUtils.hasText(managementContextPath)) {
      instance.setHealthCheckUrlPath(
          managementContextPath + instance.getHealthCheckUrlPath());
      instance.setStatusPageUrlPath(
          managementContextPath + instance.getStatusPageUrlPath());
    }
  }

  setupJmxPort(instance, jmxPort);
  return instance;
}
```

##### 启动时从EurekaServer获取服务实例信息

首先会创建CloudEurekaClient类

```java
@Bean(destroyMethod = "shutdown")
@ConditionalOnMissingBean(value = EurekaClient.class,
    search = SearchStrategy.CURRENT)
public EurekaClient eurekaClient(ApplicationInfoManager manager,
    EurekaClientConfig config) {
  return new CloudEurekaClient(manager, config, this.optionalArgs,
      this.context);
}
```

```java
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
  EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args, ApplicationEventPublisher publisher) {
  
  super(applicationInfoManager, config, args);
  this.applicationInfoManager = applicationInfoManager;
  this.publisher = publisher;
  this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class,
      "eurekaTransport");
  ReflectionUtils.makeAccessible(this.eurekaTransportField);
}
```

```java
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args) {
    this(applicationInfoManager, config, args, ResolverUtils::randomize);
}
```

```java
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args, EndpointRandomizer randomizer) {
    this(applicationInfoManager, config, args, new Provider<BackupRegistry>() {
        private volatile BackupRegistry backupRegistryInstance;

        @Override
        public synchronized BackupRegistry get() {
            if (backupRegistryInstance == null) {
                String backupRegistryClassName = config.getBackupRegistryImpl();
                if (null != backupRegistryClassName) {
                    try {
                        backupRegistryInstance = (BackupRegistry) Class.forName(backupRegistryClassName).newInstance();
                        logger.info("Enabled backup registry of type {}", backupRegistryInstance.getClass());
                    } catch (InstantiationException e) {
                        logger.error("Error instantiating BackupRegistry.", e);
                    } catch (IllegalAccessException e) {
                        logger.error("Error instantiating BackupRegistry.", e);
                    } catch (ClassNotFoundException e) {
                        logger.error("Error instantiating BackupRegistry.", e);
                    }
                }

                if (backupRegistryInstance == null) {
                    logger.warn("Using default backup registry implementation which does not do anything.");
                    backupRegistryInstance = new NotImplementedRegistryImpl();
                }
            }

            return backupRegistryInstance;
        }
    }, randomizer);
}
```

最终会获取注册列表

```java
@Inject
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
    if (args != null) {
        this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
        this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
        this.eventListeners.addAll(args.getEventListeners());
        this.preRegistrationHandler = args.preRegistrationHandler;
    } else {
        this.healthCheckCallbackProvider = null;
        this.healthCheckHandlerProvider = null;
        this.preRegistrationHandler = null;
    }
    
    this.applicationInfoManager = applicationInfoManager;
    InstanceInfo myInfo = applicationInfoManager.getInfo();

    clientConfig = config;
    staticClientConfig = clientConfig;
    transportConfig = config.getTransportConfig();
    instanceInfo = myInfo;
    if (myInfo != null) {
        appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
    } else {
        logger.warn("Setting instanceInfo to a passed in null value");
    }

    this.backupRegistryProvider = backupRegistryProvider;
    this.endpointRandomizer = endpointRandomizer;
    this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
    localRegionApps.set(new Applications());

    fetchRegistryGeneration = new AtomicLong(0);

    remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
    remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));

    if (config.shouldFetchRegistry()) {
        this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }

    if (config.shouldRegisterWithEureka()) {
        this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }

    logger.info("Initializing Eureka in region {}", clientConfig.getRegion());
    if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
        logger.info("Client configured to neither register nor query for data.");
        scheduler = null;
        heartbeatExecutor = null;
        cacheRefreshExecutor = null;
        eurekaTransport = null;
        instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        initRegistrySize = this.getApplications().size();
        registrySize = initRegistrySize;
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, initRegistrySize);

        return;  // no need to setup up an network tasks and we are done
    }

    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());

        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        cacheRefreshExecutor = new ThreadPoolExecutor(
                1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        eurekaTransport = new EurekaTransport();
        scheduleServerEndpointTask(eurekaTransport, args);

        AzToRegionMapper azToRegionMapper;
        if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
            azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
        } else {
            azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
        }
        if (null != remoteRegionsToFetch.get()) {
            azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
        }
        instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
    } catch (Throwable e) {
        throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
    }

    if (clientConfig.shouldFetchRegistry()) {
        try {
            // 获取注册列表信息
            boolean primaryFetchRegistryResult = fetchRegistry(false);
            if (!primaryFetchRegistryResult) {
                logger.info("Initial registry fetch from primary servers failed");
            }
            boolean backupFetchRegistryResult = true;
            if (!primaryFetchRegistryResult && !fetchRegistryFromBackup()) {
                backupFetchRegistryResult = false;
                logger.info("Initial registry fetch from backup servers failed");
            }
            if (!primaryFetchRegistryResult && !backupFetchRegistryResult && clientConfig.shouldEnforceFetchRegistryAtInit()) {
                throw new IllegalStateException("Fetch registry error at startup. Initial fetch failed.");
            }
        } catch (Throwable th) {
            logger.error("Fetch registry error at startup: {}", th.getMessage());
            throw new IllegalStateException(th);
        }
    }

    // call and execute the pre registration handler before all background tasks (inc registration) is started
    if (this.preRegistrationHandler != null) {
        this.preRegistrationHandler.beforeRegistration();
    }

    if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
        try {
            // 注册自己到Eureka Server
            if (!register() ) {
                throw new IllegalStateException("Registration error at startup. Invalid server response.");
            }
        } catch (Throwable th) {
            logger.error("Registration error at startup: {}", th.getMessage());
            throw new IllegalStateException(th);
        }
    }

    // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
    initScheduledTasks();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register timers", e);
    }

    // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
    // to work with DI'd DiscoveryClient
    DiscoveryManager.getInstance().setDiscoveryClient(this);
    DiscoveryManager.getInstance().setEurekaClientConfig(config);

    initTimestampMs = System.currentTimeMillis();
    initRegistrySize = this.getApplications().size();
    registrySize = initRegistrySize;
    logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
            initTimestampMs, initRegistrySize);
}
```

获取注册表信息

```java
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

    try {
        // If the delta is disabled or if it is the first time, get all
        // applications
        // 拿一下本地缓存
        Applications applications = getApplications();

        if (clientConfig.shouldDisableDelta()
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                || forceFullRegistryFetch
                || (applications == null)
                || (applications.getRegisteredApplications().size() == 0)
                || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
        {
            logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
            logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
            logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
            logger.info("Application is null : {}", (applications == null));
            logger.info("Registered Applications size is zero : {}",
                    (applications.getRegisteredApplications().size() == 0));
            logger.info("Application version is -1: {}", (applications.getVersion() == -1));
            // 全量获取
            getAndStoreFullRegistry();
        } else {
            // 增量获取
            getAndUpdateDelta(applications);
        }
        applications.setAppsHashCode(applications.getReconcileHashCode());
        logTotalInstances();
    } catch (Throwable e) {
        logger.info(PREFIX + "{} - was unable to refresh its cache! This periodic background refresh will be retried in {} seconds. status = {} stacktrace = {}",
                appPathIdentifier, clientConfig.getRegistryFetchIntervalSeconds(), e.getMessage(), ExceptionUtils.getStackTrace(e));
        return false;
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }

    // Notify about cache refresh before updating the instance remote status
    onCacheRefreshed();

    // Update remote status based on refreshed data held in the cache
    updateInstanceRemoteStatus();

    // registry was fetched successfully, so return true
    return true;
}
```

##### 注册自己到Eureka Server

完成服务注册列表的拉去后，把自己注册到Eureka Server

```java
if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
    try {
        if (!register() ) {
            throw new IllegalStateException("Registration error at startup. Invalid server response.");
        }
    } catch (Throwable th) {
        logger.error("Registration error at startup: {}", th.getMessage());
        throw new IllegalStateException(th);
    }
}
```

register()，底层使⽤Jersey客户端进⾏远程请求。

```java
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        // 向serviceUrl配置的Eureka Server端发起rest请求，注册自己
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```

##### 开启⼀些定时任务（⼼跳续约，刷新本地服务缓存列表）

当完成把自己注册到Eureka Server后，会开启一些定时任务

```java
initScheduledTasks();
```

```java
private void initScheduledTasks() {
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        // 定时刷新本地缓存的任务
        cacheRefreshTask = new TimedSupervisorTask(
                "cacheRefresh",
                scheduler,
                cacheRefreshExecutor,
                registryFetchIntervalSeconds,
                TimeUnit.SECONDS,
                expBackOffBound,
                new CacheRefreshThread()
        );
        scheduler.schedule(
                cacheRefreshTask,
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }

    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

        // 心跳续约的任务
        // Heartbeat timer
        heartbeatTask = new TimedSupervisorTask(
                "heartbeat",
                scheduler,
                heartbeatExecutor,
                renewalIntervalInSecs,
                TimeUnit.SECONDS,
                expBackOffBound,
                new HeartbeatThread()
        );
        scheduler.schedule(
                heartbeatTask,
                renewalIntervalInSecs, TimeUnit.SECONDS);

        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize

        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (statusChangeEvent.getStatus() == InstanceStatus.DOWN) {
                    logger.error("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };

        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }

        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```

- 刷新本地缓存的任务

```java
class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}
```

```java
@VisibleForTesting
void refreshRegistry() {
    try {
        boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

        boolean remoteRegionsModified = false;
        // This makes sure that a dynamic change to remote regions to fetch is honored.
        String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
        if (null != latestRemoteRegions) {
            String currentRemoteRegions = remoteRegionsToFetch.get();
            if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                    if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                        String[] remoteRegions = latestRemoteRegions.split(",");
                        remoteRegionsRef.set(remoteRegions);
                        instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                        remoteRegionsModified = true;
                    } else {
                        logger.info("Remote regions to fetch modified concurrently," +
                                " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                    }
                }
            } else {
                // Just refresh mapping to reflect any DNS/Property change
                instanceRegionChecker.getAzToRegionMapper().refreshMapping();
            }
        }

        boolean success = fetchRegistry(remoteRegionsModified);
        if (success) {
            registrySize = localRegionApps.get().size();
            lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
        }

        if (logger.isDebugEnabled()) {
            StringBuilder allAppsHashCodes = new StringBuilder();
            allAppsHashCodes.append("Local region apps hashcode: ");
            allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
            allAppsHashCodes.append(", is fetching remote regions? ");
            allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
            for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                allAppsHashCodes.append(", Remote region: ");
                allAppsHashCodes.append(entry.getKey());
                allAppsHashCodes.append(" , apps hashcode: ");
                allAppsHashCodes.append(entry.getValue().getAppsHashCode());
            }
            logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                    allAppsHashCodes);
        }
    } catch (Throwable e) {
        logger.error("Cannot fetch registry from server", e);
    }
}
```

- 心跳续约定时任务

```java
private class HeartbeatThread implements Runnable {
   public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}
```

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        // 向续约接口发送请求
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        // 找不到可能是发生了服务剔除
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            // 如果被剔除了，需要重新注册自己
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}

```

#### Eureka Client下架服务

在EurekaClientAutoConfiguration中注入了shutdown bean，服务下架入口，当客户端工程关闭容器销毁的时候，会调用shutdown方法执行一些清理操作，下线操作

```java
@Bean(destroyMethod = "shutdown")
@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config) {
  return new CloudEurekaClient(manager, config, this.optionalArgs, this.context);
}
```

`com.netflix.discovery.DiscoveryClient#shutdown`

```java
@PreDestroy
@Override
public synchronized void shutdown() {
    if (isShutdown.compareAndSet(false, true)) {
        logger.info("Shutting down DiscoveryClient ...");

        if (statusChangeListener != null && applicationInfoManager != null) {
            applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
        }

        cancelScheduledTasks();

        // If APPINFO was registered
        if (applicationInfoManager != null
                && clientConfig.shouldRegisterWithEureka()
                && clientConfig.shouldUnregisterOnShutdown()) {
            applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
            // 向服务注册中心发送下线请求
            unregister();
        }

        if (eurekaTransport != null) {
            eurekaTransport.shutdown();
        }

        heartbeatStalenessMonitor.shutdown();
        registryStalenessMonitor.shutdown();

        Monitors.unregisterObject(this);

        logger.info("Completed shut down of DiscoveryClient");
    }
}
```

`unregister()`

```java
void unregister() {
    // It can be null if shouldRegisterWithEureka == false
    if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
        try {
            logger.info("Unregistering ...");
            // 底层手捧玫瑰Jersey发送下线请求
            EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
            logger.info(PREFIX + "{} - deregister  status: {}", appPathIdentifier, httpResponse.getStatusCode());
        } catch (Exception e) {
            logger.error(PREFIX + "{} - de-registration failed{}", appPathIdentifier, e.getMessage(), e);
        }
    }
}
```

#### Eureka Client⼼跳续约

参考前面

# Ribbon负载均衡

## 关于负载均衡

![[Architecture Design#负载均衡设计]]
## Ribbon使用

不需要引⼊额外的Jar坐标，因为在服务消费者中我们引⼊过eureka-client，它会引⼊Ribbon相关Jar。

代码中使⽤如下，在RestTemplate上添加对应注解

```java
@Bean
// Ribbon负载均衡
@LoadBalanced
public RestTemplate getRestTemplate() {
    return new RestTemplate();
}
```

## Ribbon负载均衡策略

Ribbon内置了多种负载均衡策略，内部负责复杂均衡的顶级接⼝为 `com.netflix.loadbalancer.IRule`

|负载均衡策略|描述|
|---|---|
|RoundRobinRule： 轮询策略|轮询服务列表，默认获取到的server超过10次都不可⽤，会返回⼀个空的server|
|RandomRule： 随机策略|随机选择一个服务，如果随机到的server为null或者不可⽤的话，会while不停的循环选取|
|RetryRule： 重试策略|⼀定时限内循环重试。默认继承RoundRobinRule，也⽀持⾃定义注⼊， RetryRule会在每次选取之后，对选举的server进⾏判断，是否为null，是否alive，并且在500ms内会不停的选取判断。⽽RoundRobinRule失效的策略是超过10次， RandomRule是没有失效时间的概念，只要serverList没都挂。|
|BestAvailableRule： 最⼩连接数策略|遍历serverList，选取出可⽤的且连接数最⼩的⼀个server。该算法⾥⾯有⼀个LoadBalancerStats的成员变量，会存储所有server的运⾏状况和连接数。如果选取到的server为null或者服务的连接数一样，那么会调⽤RoundRobinRule选取。|
|AvailabilityFilteringRule： 可⽤过滤策略|扩展了轮询策略，会先通过默认的轮询选取⼀个server，再去判断该server是否超时可⽤，当前连接数是否超限，都成功再返回。|
|ZoneAvoidanceRule： 区域权衡策略（默认策略）|扩展了轮询策略，继承了2个过滤器：ZoneAvoidancePredicate和AvailabilityPredicate，除了过滤超时和链接数过多的server，还会过滤掉不符合要求的zone区域⾥⾯的所有节点， 在⼀个区域/机房内的服务实例中轮询|

消费者修改负载均衡策略（因为是客户端负载均衡，所以很容易理解要在客户端配置）

```yaml
#针对的生产者微服务名称,不加就是全局⽣效
lagou-service-resume:
  ribbon:
    NFLoadBalancerRuleClassName:
      com.netflix.loadbalancer.RandomRule #负载策略调整
```

## Ribbon核⼼源码剖析

### Ribbon⼯作原理

![](ribbon.png)

重点： Ribbon给`RestTemplate`添加了⼀个拦截器

思考： Ribbon在做什么：

当我们访问`http://xxxService/resume/openstate/`的时候， ribbon应该根据服务名`xxxService`获取到该服务的实例列表并按照⼀定的负载均衡策略从实例列表中获取⼀个实例Server，并最终通过`RestTemplate`进⾏请求访问

Ribbon细节结构图（涉及到底层的⼀些组件/类的描述）

1. 获取服务实例列表
2. 从列表中选择⼀个server

![](ribbon2.png)

图中核⼼是负载均衡管理器LoadBalancer，围绕它周围的多有IRule、 IPing等

- IRule：是在选择实例的时候的负载均衡策略对象
- IPing：是⽤来向服务发起⼼跳检测的，通过⼼跳检测来判断该服务是否可⽤
- ServerListFilter：根据⼀些规则过滤传⼊的服务实例列表
- ServerListUpdater：定义了⼀系列的对服务列表的更新操作

### @LoadBalanced源码剖析

我们在RestTemplate实例上添加了⼀个`@LoadBalanced`注解，就可以实现负载均衡，很神奇，我们接下来分析这个注解背后的操作（负载均衡过程）

查看`@LoadBalanced`注解，那这个注解是在哪⾥被识别到的呢？

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {

}

```

LoadBalancerClient类（实现类RibbonLoadBalancerClient， 待⽤）

```JAVA
public interface LoadBalancerClient extends ServiceInstanceChooser {
  // 根据服务执⾏请求内容
  <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
  // 根据服务执⾏请求内容
  <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
  // 拼接请求⽅式 传统中是ip:port 现在是服务名称:port 形式
  URI reconstructURI(ServiceInstance instance, URI original);
}
```

`spring.factories`配置⽂件中增加了`RibbonAutoConfiguration` 的自动配置

```java
@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
@RibbonClients
@AutoConfigureAfter(
    name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
    AsyncLoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,
    ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration {
```

`LoadBalancerAutoConfiguration`

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerProperties.class)
public class LoadBalancerAutoConfiguration {

  // 将所有加了LoadBalanced注解的RestTemplate注入进来（LoadBalanced注解是一个Qualifier注解）
  @LoadBalanced
  @Autowired(required = false)
  private List<RestTemplate> restTemplates = Collections.emptyList();
  
  

  static class LoadBalancerInterceptorConfig {
    // ...
    // 注⼊RestTemplate定制器
    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
      return restTemplate -> {
        List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
        // 添加了拦截器
        list.add(loadBalancerInterceptor);
        restTemplate.setInterceptors(list);
      };
    }
    // ...
  }
 } 
```

到这⾥，我们明⽩，添加了注解的RestTemplate对象会被添加⼀个拦截器 `LoadBalancerInterceptor`，该拦截器就是后续拦截请求进⾏负载处理的。

所以，下⼀步重点我们该分析拦截器`LoadBalancerInterceptor#intercept()`⽅法

```java
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
    final ClientHttpRequestExecution execution) throws IOException {
    // 获取拦截到的请求uri
    final URI originalUri = request.getURI();
    // 获取uri中的服务名
    String serviceName = originalUri.getHost();
    Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
    // 执行
    return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
}

```

`requestFactory.createRequest(request, body, execution)`

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(
  final HttpRequest request, final byte[] body,
  final ClientHttpRequestExecution execution) {
  return instance -> {
    HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
        this.loadBalancer);
    if (this.transformers != null) {
      for (LoadBalancerRequestTransformer transformer : this.transformers) {
        serviceRequest = transformer.transformRequest(serviceRequest,
            instance);
      }
    }
    // 这里会调用执行方法，RestTemplate底层也是执行这个方法
    // 当外部执行request.apply的时候才会执行
    return execution.execute(serviceRequest, body);
  };
}
```

回到上一步execute方法

```java
@Override
public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
    throws IOException {
  return execute(serviceId, request, null);
}
```

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
      throws IOException {
  // 获取一个负载均衡器，是从spring容器中获取的，负载均衡器是在RibbonAutoConfiguration中注入的
  // 默认实现为ZoneAwareLoadBalancer
  ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
  // 通过负载均衡器获取一个server
  Server server = getServer(loadBalancer, hint);
  if (server == null) {
    throw new IllegalStateException("No instances available for " + serviceId);
  }
  RibbonServer ribbonServer = new RibbonServer(serviceId, server,
      isSecure(server, serviceId),
      serverIntrospector(serviceId).getMetadata(server));

  return execute(serviceId, ribbonServer, request);
}
```

- `getServer(loadBalancer, hint); `获取一个server

```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
  if (loadBalancer == null) {
    return null;
  }
  // Use 'default' on a null hint, or just pass it on?
  return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

```java
@Override
public Server chooseServer(Object key) {
    // 如果只有一个区域，直接选择该区域
    if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
    Server server = null;
    try {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
        logger.debug("Zone snapshots: {}", zoneSnapshot);
        if (triggeringLoad == null) {
            triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
        }

        if (triggeringBlackoutPercentage == null) {
            triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
        }
        Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
        logger.debug("Available zones: {}", availableZones);
        if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
            String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
            logger.debug("Zone chosen: {}", zone);
            if (zone != null) {
                BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                server = zoneLoadBalancer.chooseServer(key);
            }
        }
    } catch (Exception e) {
        logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
    }
    if (server != null) {
        return server;
    } else {
        logger.debug("Zone avoidance logic is not invoked.");
        return super.chooseServer(key);
    }
}
```

```java
public Server chooseServer(Object key) {
    if (counter == null) {
        counter = createCounter();
    }
    counter.increment();
    if (rule == null) {
        return null;
    } else {
        try {
            // 通过负载均衡策略选择服务器
            return rule.choose(key);
        } catch (Exception e) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
            return null;
        }
    }
}
```

```java
@Override
public Server choose(Object key) {
    ILoadBalancer lb = getLoadBalancer();
    // 从服务列表过滤之后，根据轮询策略选择
    Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
    if (server.isPresent()) {
        return server.get();
    } else {
        return null;
    }       
}
```

```java
public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
    // 过滤获取满足条件的server
    List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
    if (eligible.size() == 0) {
        return Optional.absent();
    }
    // 实例索引值计算，然后获取
    return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
}
```

- ⾮常核⼼的⼀个⽅法： `RibbonLoadBalancerClient.execute()`

```java
@Override
public <T> T execute(String serviceId, ServiceInstance serviceInstance,
    LoadBalancerRequest<T> request) throws IOException {
    
    Server server = null;
    if (serviceInstance instanceof RibbonServer) {
      server = ((RibbonServer) serviceInstance).getServer();
    }
    if (server == null) {
      throw new IllegalStateException("No instances available for " + serviceId);
    }
  
    RibbonLoadBalancerContext context = this.clientFactory
        .getLoadBalancerContext(serviceId);
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);
  
    try {
      // 向server实例发起请求的关键步骤，前面已经调用过requestFactory.createRequest(request, body, execution)，此时都会执行其中定义好的逻辑
      T returnVal = request.apply(serviceInstance);
      statsRecorder.recordStats(returnVal);
      return returnVal;
    }
    // catch IOException and rethrow so RestTemplate behaves correctly
    catch (IOException ex) {
      statsRecorder.recordStats(ex);
      throw ex;
    }
    catch (Exception ex) {
      statsRecorder.recordStats(ex);
      ReflectionUtils.rethrowRuntimeException(ex);
    }
    return null;
}

```

那么 `RibbonLoadBalancerClient`对象是在哪⾥注⼊的？回到最初的⾃动配置类`RibbonAutoConfiguration`中，可以看到是交给了我们最初看到的`RibbonLoadBalancerClient`对象

```java
public class RibbonAutoConfiguration {

  // ...
  @Bean
  @ConditionalOnMissingBean(LoadBalancerClient.class)
  public LoadBalancerClient loadBalancerClient() {
    return new RibbonLoadBalancerClient(springClientFactory());
  }
```

在进⾏负载chooseServer的时候， `LoadBalancer`负载均衡器中已经有了serverList，那么这个serverList是什么时候被注⼊到`LoadBalancer`中的，它的⼀个机制⼤概是怎样的？

首先回到`RibbonAutoConfiguration`中，可以找到注入了一个`SpringClientFactory`

```java
@Bean
@ConditionalOnMissingBean
public SpringClientFactory springClientFactory() {
  SpringClientFactory factory = new SpringClientFactory();
  factory.setConfigurations(this.configurations);
  return factory;
}
```

`SpringClientFactory`的创建会保存这个类，之后会注册这个类

```java
public SpringClientFactory() {
  super(RibbonClientConfiguration.class, NAMESPACE, "ribbon.client.name");
}
```

来到`RibbonClientConfiguration`，发现其注入了serverList

```java
@Bean
@ConditionalOnMissingBean
@SuppressWarnings("unchecked")
public ServerList<Server> ribbonServerList(IClientConfig config) {
  if (this.propertiesFactory.isSet(ServerList.class, name)) {
    return this.propertiesFactory.get(ServerList.class, config, name);
  }
  ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
  serverList.initWithNiwsConfig(config);
  return serverList;
}
```

这个serverList刚开始是没有数据的，是在后续操作中初始化的

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
    ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
    IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
  if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
    return this.propertiesFactory.get(ILoadBalancer.class, config, name);
  }
  return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
      serverListFilter, serverListUpdater);
}
```

在`ZoneAwareLoadBalancer`父类中，可以发现有一个方法

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, ServerList<T> serverList, ServerListFilter<T> filter, ServerListUpdater serverListUpdater) {
    super(clientConfig, rule, ping);
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
    }
    // 初始化
    restOfInit(clientConfig);
}
```

进入`restOfInit(clientConfig);`其调用了updateListOfServers来初始化了serverList

```java
void restOfInit(IClientConfig clientConfig) {
    boolean primeConnection = this.isEnablePrimingConnections();
    // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
    this.setEnablePrimingConnections(false);
    // 开启定时任务，其中也做了一些数据的获取包括serverList，定时任务做的事定时延时拉取
    enableAndInitLearnNewServersFeature();
    // 更新serverList，这里是立即执行，和上边不冲突
    updateListOfServers();
    if (primeConnection && this.getPrimeConnections() != null) {
        this.getPrimeConnections()
                .primeConnections(getReachableServers());
    }
    this.setEnablePrimingConnections(primeConnection);
}
```

```java
@VisibleForTesting
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        servers = serverListImpl.getUpdatedListOfServers();
        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
        }
    }
    // 更新所有serverList
    updateAllServerList(servers);
}
```

大体看一下enableAndInitLearnNewServersFeature方法

```java
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
    @Override
    public void doUpdate() {
        // 又回到了上面
        updateListOfServers();
    }
};

public void enableAndInitLearnNewServersFeature() {
    serverListUpdater.start(updateAction);
}
```

`com.netflix.loadbalancer.PollingServerListUpdater#start`

```java
@Override
public synchronized void start(final UpdateAction updateAction) {
    if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                if (!isActive.get()) {
                    if (scheduledFuture != null) {
                        scheduledFuture.cancel(true);
                    }
                    return;
                }
                try {
                    updateAction.doUpdate();
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };

        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                wrapperRunnable,
                initialDelayMs,
                refreshIntervalMs,
                TimeUnit.MILLISECONDS
        );
    } else {
        logger.info("Already active, no-op");
    }
}
```

# Hystrix熔断器

[[MicroService Design#熔断限流]]

## Hystrix简介

Hystrix由Netflix开源的⼀个延迟和容错库，⽤于隔离访问远程系统、服务或者第三⽅库，防⽌级联失败，从⽽提升系统的可⽤性与容错性。 Hystrix主要通过以下⼏点实现延迟和容错。

- 包裹请求：使⽤HystrixCommand包裹对依赖的调⽤逻辑。
- 跳闸机制：当某服务的错误率超过⼀定的阈值时， Hystrix可以跳闸，停⽌请求该服务⼀段时间。
- 资源隔离： Hystrix为每个依赖都维护了⼀个⼩型的线程池(舱壁模式)（或者信号量）。如果该线程池已满， 发往该依赖的请求就被⽴即拒绝，⽽不是排队等待，从⽽加速失败判定。
- 监控： Hystrix可以近乎实时地监控运⾏指标和配置的变化，例如成功、失败、超时、以及被拒绝 的请求等。
- 回退机制：当请求失败、超时、被拒绝，或当断路器打开时，执⾏回退逻辑。回退逻辑由开发⼈员⾃⾏提供，例如返回⼀个缺省值。
- ⾃我修复：断路器打开⼀段时间后，会⾃动进⼊“半开”状态。

## Hystrix熔断应⽤

TODO

## Hystrix舱壁模式（线程池隔离策略）

默认Hystrix有一个线程池（10个），为所有的添加了`@HystrixCommand`方法提供线程，如果这些方法接受请求超过了10个，其他请求就得等待或者拒绝链接。为了避免问题服务请求过多导致正常服务⽆法访问， Hystrix 不是采⽤增加线程数，⽽是单独的为每⼀个控制⽅法创建⼀个线程池的⽅式，这种模式叫做“**舱壁模式**"，也是线程隔离的⼿段。

![](hystrix舱壁模式.png)

Hystrix舱壁模式程序修改

```java
/**
* 提供者模拟处理超时，调⽤⽅法添加Hystrix控制
* @param userId
* @return
*/
// 使⽤@HystrixCommand注解进⾏熔断控制
@HystrixCommand(
    // 线程池标识，要保持唯⼀，不唯⼀的话就共⽤了
    threadPoolKey = "findResumeOpenStateTimeout",
    // 线程池细节属性配置
    threadPoolProperties = {
        @HystrixProperty(name="coreSize",value = "1"),
        // 线程数
        @HystrixProperty(name="maxQueueSize",value="20") // 等待队列⻓度
    },
    // commandProperties熔断的⼀些细节属性配置
    commandProperties = {
        // 每⼀个属性都是⼀个HystrixProperty
        @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="2000")
    }
)
@GetMapping("/checkStateTimeout/{userId}")
public Integer findResumeOpenStateTimeout(@PathVariable Long userId) {
    // 使⽤ribbon不需要我们⾃⼰获取服务实例然后选择⼀个那么去访问了（⾃⼰的负载均衡）
    String url = "http://xxxService/resume/openstate/" + userId; // 指定服务名
    Integer forObject = restTemplate.getForObject(url, Integer.class);
    return forObject;
}
@GetMapping("/checkStateTimeoutFallback/{userId}")
@HystrixCommand(
    // 线程池标识，要保持唯⼀，不唯⼀的话就共⽤了
    threadPoolKey = "findResumeOpenStateTimeoutFallback",
    // 线程池细节属性配置
    threadPoolProperties = {
        @HystrixProperty(name="coreSize",value = "2"),
        // 线程数
        @HystrixProperty(name="maxQueueSize",value="20") // 等待队列⻓度
    },
    // commandProperties熔断的⼀些细节属性配置
    commandProperties = {
        // 每⼀个属性都是⼀个HystrixProperty
        @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="2000")
    },
    fallbackMethod = "myFallBack" // 回退⽅法
)
public Integer findResumeOpenStateTimeoutFallback(@PathVariable Long userId) {
    // 使⽤ribbon不需要我们⾃⼰获取服务实例然后选择⼀个那么去访问了（⾃⼰的负载均衡）
    String url = "http://xxxService/resume/openstate/" + userId; // 指定服务名
    Integer forObject = restTemplate.getForObject(url, Integer.class);
    return forObject;
}

```

## Hystrix⼯作流程与⾼级应⽤

![](hystrix工作流程.png)

1. 当调⽤出现问题时，开启⼀个时间窗（10s）
2. 在这个时间窗内，统计调⽤次数是否达到最⼩请求数？
    - 如果没有达到，则重置统计信息，回到第1步
    - 如果达到了，则统计失败的请求数占所有请求数的百分⽐，是否达到阈值？
        - 如果达到，则跳闸（不再请求对应服务）
            - 如果跳闸，则会开启⼀个活动窗⼝（默认5s），每隔5s， Hystrix会让⼀个请求通过,到达那个问题服务，看是否调⽤成功，如果成功，重置断路器回到第1步，如果失败，则继续开启活动窗口重试
        - 如果没有达到，则重置统计信息，回到第1步

```java
/**
* 8秒钟内，请求次数达到2个，并且失败率在50%以上，就跳闸
* 跳闸后活动窗⼝设置为3s
*/
@HystrixCommand(
    commandProperties = {
        @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds",value = "8000"),
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "2"),
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "50"),
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "3000")
    }
)
```

## Hystrix核⼼源码剖析

TODO
# Feign远程调⽤组件

## Feign简介

Feign是Netflix开发的⼀个轻量级RESTful的HTTP服务客户端（⽤它来发起请求，远程调⽤的） ，是以Java接⼝注解的⽅式调⽤Http请求，⽽不⽤像Java中通过封装HTTP请求报⽂的⽅式直接调⽤， Feign被⼴泛应⽤在Spring Cloud 的解决⽅案中。

类似于Dubbo，服务消费者拿到服务提供者的接⼝，然后像调⽤本地接⼝⽅法⼀样去调⽤，实际发出的是远程的请求。

Feign可帮助我们更加便捷，优雅的调⽤HTTP API：不需要我们去拼接url然后呢调⽤restTemplate的api。SpringCloud对Feign进⾏了增强，使Feign⽀持了SpringMVC注解（OpenFeign）

本质：**封装了Http调⽤流程，更符合⾯向接⼝化的编程习惯，类似于Dubbo的服务调⽤Dubbo的调⽤⽅式其实就是很好的⾯向接⼝编程**

## 原理

本质：**动态代理 + Ribbon + 熔断**

## Feign配置应⽤

在服务调⽤者⼯程（消费）创建接⼝（添加注解）

（效果） **Feign = RestTemplate + Ribbon + Hystrix**

服务消费者⼯程（⾃动投递微服务）中引⼊Feign依赖（或者⽗类⼯程）

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

```

服务消费者⼯程（⾃动投递微服务）启动类使⽤注解@EnableFeignClients添加Feign⽀持

```java
@SpringBootApplication
@EnableDiscoveryClient // 开启服务发现
@EnableFeignClients // 开启Feign
public class AutodeliverFeignApplication8092 {
    public static void main(String[] args) {
        SpringApplication.run(AutodeliverFeignApplication8092.class, args);
    }
}
```

注意： 此时去掉Hystrix熔断的⽀持注解@EnableCircuitBreaker即可包括引⼊的依赖，因为Feign会⾃动引⼊

创建Feign接⼝

```java
// name：调⽤的服务名称，和服务提供者yml⽂件中spring.application.name保持⼀致
@FeignClient(name="lagou-service-resume")
public interface ResumeFeignClient {
    //调⽤的请求路径
    @RequestMapping(value = "/resume/openstate/{userId}",method = RequestMethod.GET)
    public Integer findResumeOpenState(@PathVariable(value = "userId") Long userId);
}
```

注意：

* `@FeignClient`注解的name属性⽤于指定要调⽤的服务提供者名称，和服务提供者yml⽂件中spring.application.name保持⼀致
  
* 接⼝中的接⼝⽅法，就好⽐是远程服务提供者Controller中的Hander⽅法（只不过如同本地调⽤了），那么在进⾏参数绑定的时，可以使⽤`@PathVariable`、 `@RequestParam`、 `@RequestHeader`等，这也是OpenFeign对SpringMVC注解的⽀持，但是需要注意value必须设置，否则会抛出异常

使⽤接⼝中⽅法完成远程调⽤（注⼊接⼝即可，实际注⼊的是接⼝的实现）

```java
@Autowired
private ResumeFeignClient resumeFeignClient;
@Test
public void testFeignClient(){
    Integer resumeOpenState = resumeFeignClient.findResumeOpenState(1545132l);
    System.out.println("=======>>>resumeOpenState： " + resumeOpenState);
}
```

## Feign对负载均衡的⽀持

Feign 本身已经集成了Ribbon依赖和⾃动配置，因此我们不需要额外引⼊依赖，可以通过 ribbon.xx 来进 ⾏全局配置,也可以通过服务名.ribbon.xx 来对指定服务进⾏细节配置配置（参考之前，此处略）

Feign默认的请求处理超时时⻓1s，有时候我们的业务确实执⾏的需要⼀定时间，那么这个时候，我们就需要调整请求处理超时时⻓， Feign⾃⼰有超时设置，如果配置Ribbon的超时，则会以Ribbon的为准

**Ribbon设置**

```yaml
#针对的被调⽤⽅微服务名称,不加就是全局⽣效
xxxService:
  ribbon:
    #请求连接超时时间
    ConnectTimeout: 2000
    #请求处理超时时间
    ReadTimeout: 5000
    #对所有操作都进⾏重试
    #针对的被调⽤⽅微服务名称,不加就是全局⽣效

```

## Feign对熔断器的⽀持

1. 在Feign客户端⼯程配置⽂件（`application.yml`）中开启Feign对熔断器的⽀持
	
	```yaml
	# 开启Feign的熔断功能
	feign:
	  hystrix:
	    enabled: true
	```
	
	Feign的超时时⻓设置那其实就上⾯Ribbon的超时时⻓设置Hystrix超时设置。
	
	注意：  
	
	1. 开启Hystrix之后， Feign中的⽅法都会管理，⼀旦出现问题就进⼊对应的回退逻辑处理  
	2. Feign把复杂的http请求包装成我们只需要加注解就可以实现，但是底层使用的还是Ribbon。Feign的调用，总共分两层，即Ribbon的调用和Hystrix(熔断处理)的调用，***高版本的Hystrix默认是关闭的***。综上所得：**Feign的熔断超时时间需要同时设置Ribbon和Hystrix的超时时间设置**
	3. 针对超时这⼀点，当前有两个超时时间设置（Feign/hystrix），熔断的时候是根据这两个时间的最⼩值来进⾏的，即处理时⻓超过最短的那个超时时间了就熔断进⼊回退降级逻辑
	
	```yaml
	hystrix:
	  command:
	    default:
	      execution:
	        isolation:
	          thread:
	            # Hystrix的超时时⻓设置
	            timeoutInMilliseconds: 15000
```

2. ⾃定义FallBack处理类（需要实现FeignClient接⼝）

```java
/**
* 降级回退逻辑需要定义⼀个类，实现FeignClient接⼝，实现接⼝中的⽅法
**
*/
@Component // 别忘了这个注解，还应该被扫描到
public class ResumeFallback implements ResumeServiceFeignClient {
    @Override
    public Integer findDefaultResumeState(Long userId) {
        return -6;
    }
}
```

3. 在@FeignClient注解中关联第二步中⾃定义的处理类

```java
@FeignClient(value = "xxxService",fallback = ResumeFallback.class,path = "/resume") 
// 使⽤fallback的时候，类上的@RequestMapping的url前缀限定，改成配置在@FeignClient的path属性中
//@RequestMapping("/resume")
public interface ResumeServiceFeignClient {
```

## Feign对请求压缩和响应压缩的⽀持

Feign ⽀持对请求和响应进⾏GZIP压缩，以减少通信过程中的性能损耗。通过下⾯的参数即可开启请求与响应的压缩功能：

```yaml
feign:
  compression:
    request:
      enabled: true # 开启请求压缩
      mime-types: text/html,application/xml,application/json # 设置压缩的数据类型，此处也是默认值
      min-request-size: 2048 # 设置触发压缩的⼤⼩下限，此处也是默认值
    response:
      enabled: true # 开启响应压缩
```

## Feign的⽇志级别配置

Feign是http请求客户端，类似于咱们的浏览器，它在请求和接收响应的时候，可以打印出⽐较详细的⼀些⽇志信息（响应头，状态码等等）

如果我们想看到Feign请求时的⽇志，我们可以进⾏配置，默认情况下Feign的⽇志没有开启。

1. 开启Feign⽇志功能及级别

```java
// Feign的⽇志级别（Feign请求过程信息）
// NONE：默认的，不显示任何⽇志----性能最好
// BASIC：仅记录请求⽅法、 URL、响应状态码以及执⾏时间----⽣产问题追踪
// HEADERS：在BASIC级别的基础上，记录请求和响应的header
// FULL：记录请求和响应的header、 body和元数据----适⽤于开发及测试环境定位问题
@Configuration
public class FeignConfig {
    @Bean
    Logger.Level feignLevel() {
        return Logger.Level.FULL;
    }
}
```

2. 配置log⽇志级别为debug

```yaml
logging:
  level:
    # Feign⽇志只会对⽇志级别为debug的做出响应
    com.xxx.xxx.controller.service.ResumeServiceFeignClient: debug
```

## Feign核⼼源码剖析

* 从@`EnableFeignClients` 正向切⼊

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
```

类中导入了一个类`FeignClientsRegistrar`

```java
class FeignClientsRegistrar
    implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
```

该类实现了`ImportBeanDefinitionRegistrar`接口，实现该接口，重写`registerBeanDefinitions`方法，可以完成一些Bean的注入·

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata,
    BeanDefinitionRegistry registry) {
    // 把FeignClient的全局默认配置注入到容器  
    registerDefaultConfiguration(metadata, registry);
    // 把标记了@FeignClient的类创建对象注入到容器
    registerFeignClients(metadata, registry);
}
```

- registerDefaultConfiguration

```java
private void registerDefaultConfiguration(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
  // 把@EnableFeignClients中的defaultConfiguration属性张配置的class类注入到容器
  Map<String, Object> defaultAttrs = metadata
      .getAnnotationAttributes(EnableFeignClients.class.getName(), true);

  if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
    String name;
    if (metadata.hasEnclosingClass()) {
      name = "default." + metadata.getEnclosingClassName();
    }
    else {
      name = "default." + metadata.getClassName();
    }
    registerClientConfiguration(registry, name,
        defaultAttrs.get("defaultConfiguration"));
  }
}
```

- `registerFeignClients(metadata,  registry);`

```java
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {

  LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
  Map<String, Object> attrs = metadata
      .getAnnotationAttributes(EnableFeignClients.class.getName());
  AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
      FeignClient.class);
  final Class<?>[] clients = attrs == null ? null
      : (Class<?>[]) attrs.get("clients");
  if (clients == null || clients.length == 0) {
    // 定义了扫描器，主要是扫描@FeignClient注解
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    scanner.setResourceLoader(this.resourceLoader);
    scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
    Set<String> basePackages = getBasePackages(metadata);
    // 使用扫描器扫描@FeignClient注解标识的类（在basePackages指定的目录中扫描，如果不指定，扫默认启动类下的）
    for (String basePackage : basePackages) {
      candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
    }
  }
  else {
    for (Class<?> clazz : clients) {
      candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
    }
  }

  for (BeanDefinition candidateComponent : candidateComponents) {
    if (candidateComponent instanceof AnnotatedBeanDefinition) {
      // verify annotated class is an interface
      AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
      AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
      Assert.isTrue(annotationMetadata.isInterface(),
          "@FeignClient can only be specified on an interface");

      Map<String, Object> attributes = annotationMetadata
          .getAnnotationAttributes(FeignClient.class.getCanonicalName());

      String name = getClientName(attributes);
      // @FeignClient注解有一个配置属性configuration，取出来注入到容器
      registerClientConfiguration(registry, name,
          attributes.get("configuration"));
      // 注入FeignClient客户端对象，这个对象也是controller中使用的对象
      registerFeignClient(registry, annotationMetadata, attributes);
    }
  }
}
```

    注册客户端，给每⼀个客户端⽣成代理对象

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
  String className = annotationMetadata.getClassName();
  // 封装BeanDefination对象，根据扫描的元信息，这里是FeignClientFactoryBean，真正获取对象是其中的getObject方法
  BeanDefinitionBuilder definition = BeanDefinitionBuilder
      .genericBeanDefinition(FeignClientFactoryBean.class);
  validate(attributes);
  definition.addPropertyValue("url", getUrl(attributes));
  definition.addPropertyValue("path", getPath(attributes));
  String name = getName(attributes);
  definition.addPropertyValue("name", name);
  String contextId = getContextId(attributes);
  definition.addPropertyValue("contextId", contextId);
  definition.addPropertyValue("type", className);
  definition.addPropertyValue("decode404", attributes.get("decode404"));
  definition.addPropertyValue("fallback", attributes.get("fallback"));
  definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
  definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

  String alias = contextId + "FeignClient";
  AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
  beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);

  // has a default, won't be null
  boolean primary = (Boolean) attributes.get("primary");

  beanDefinition.setPrimary(primary);

  String qualifier = getQualifier(attributes);
  if (StringUtils.hasText(qualifier)) {
    alias = qualifier;
  }

  BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
      new String[] { alias });
  BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

所以，下⼀步，关注FeignClientFactoryBean这个⼯⼚Bean的getObject⽅法，这个⽅法会返回我们的代理对象。

```java
@Override
public Object getObject() throws Exception {
  return getTarget();
}
```

```java
<T> T getTarget() {
  FeignContext context = applicationContext.getBean(FeignContext.class);
  Feign.Builder builder = feign(context);
  // 判断FeignClient注解属性url是否为空
  // url如果为空，那么生成的FeignClient客户端对象应该是一个带有负载均衡功能的客户端对象feign + ribbon
  if (!StringUtils.hasText(url)) {
    if (!name.startsWith("http")) {
      url = "http://" + name;
    }
    else {
      url = name;
    }
    url += cleanPath();
    return (T) loadBalance(builder, context,
        new HardCodedTarget<>(type, name, url));
  }
  if (StringUtils.hasText(url) && !url.startsWith("http")) {
    url = "http://" + url;
  }
  String url = this.url + cleanPath();
  Client client = getOptional(context, Client.class);
  if (client != null) {
    if (client instanceof LoadBalancerFeignClient) {
      // not load balancing because we have a url,
      // but ribbon is on the classpath, so unwrap
      client = ((LoadBalancerFeignClient) client).getDelegate();
    }
    if (client instanceof FeignBlockingLoadBalancerClient) {
      // not load balancing because we have a url,
      // but Spring Cloud LoadBalancer is on the classpath, so unwrap
      client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
    }
    builder.client(client);
  }
  Targeter targeter = get(context, Targeter.class);
  return (T) targeter.target(this, builder, context,
      new HardCodedTarget<>(type, name, url));
}
```

- url为空，调用loadBalance方法

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
      HardCodedTarget<T> target) {
  Client client = getOptional(context, Client.class);
  if (client != null) {
    builder.client(client);
    Targeter targeter = get(context, Targeter.class);
    return targeter.target(this, builder, context, target);
  }

  throw new IllegalStateException(
      "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```

    org.springframework.cloud.openfeign.HystrixTargeter#target

```java
@Override
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
    FeignContext context, Target.HardCodedTarget<T> target) {
  if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
    return feign.target(target);
  }
  feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
  String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
      : factory.getContextId();
  SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
  if (setterFactory != null) {
    builder.setterFactory(setterFactory);
  }
  Class<?> fallback = factory.getFallback();
  if (fallback != void.class) {
    return targetWithFallback(name, context, target, builder, fallback);
  }
  Class<?> fallbackFactory = factory.getFallbackFactory();
  if (fallbackFactory != void.class) {
    return targetWithFallbackFactory(name, context, target, builder,
        fallbackFactory);
  }

  return feign.target(target);
}
```

```java
public <T> T target(Target<T> target) {
  return build().newInstance(target);
}
```

```java
public Feign build() {
  Client client = Capability.enrich(this.client, capabilities);
  Retryer retryer = Capability.enrich(this.retryer, capabilities);
  List<RequestInterceptor> requestInterceptors = this.requestInterceptors.stream()
      .map(ri -> Capability.enrich(ri, capabilities))
      .collect(Collectors.toList());
  Logger logger = Capability.enrich(this.logger, capabilities);
  Contract contract = Capability.enrich(this.contract, capabilities);
  Options options = Capability.enrich(this.options, capabilities);
  Encoder encoder = Capability.enrich(this.encoder, capabilities);
  Decoder decoder = Capability.enrich(this.decoder, capabilities);
  InvocationHandlerFactory invocationHandlerFactory =
      Capability.enrich(this.invocationHandlerFactory, capabilities);
  QueryMapEncoder queryMapEncoder = Capability.enrich(this.queryMapEncoder, capabilities);

  SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
      new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
          logLevel, decode404, closeAfterDecode, propagationPolicy, forceDecoding);
  ParseHandlersByName handlersByName =
      new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
          errorDecoder, synchronousMethodHandlerFactory);
  return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

    在ReflectiveFeign中创建了代理对象

```java
@Override
public <T> T newInstance(Target<T> target) {
  Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
  Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
  List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

  for (Method method : target.type().getMethods()) {
    if (method.getDeclaringClass() == Object.class) {
      continue;
    } else if (Util.isDefault(method)) {
      DefaultMethodHandler handler = new DefaultMethodHandler(method);
      defaultMethodHandlers.add(handler);
      methodToHandler.put(method, handler);
    } else {
      methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
    }
  }
  // 这里创建的是FeignInvocationHandler对象，具体逻辑在里面
  InvocationHandler handler = factory.create(target, methodToHandler);
  // 创建代理，JDK动态代理
  T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
      new Class<?>[] {target.type()}, handler);

  for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
    defaultMethodHandler.bindTo(proxy);
  }
  return proxy;
}
```

请求进来时候，是进⼊增强逻辑的，所以接下来我们要关注增强逻辑部分，`FeignInvocationHandler`

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  if ("equals".equals(method.getName())) {
    try {
      Object otherHandler =
          args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
      return equals(otherHandler);
    } catch (IllegalArgumentException e) {
      return false;
    }
  } else if ("hashCode".equals(method.getName())) {
    return hashCode();
  } else if ("toString".equals(method.getName())) {
    return toString();
  }
  // 具体方法的增强逻辑又交给了对应的MethodHandler来处理看
  return dispatch.get(method).invoke(args);
}
```

`SynchronousMethodHandler#invoke`

```java
@Override
public Object invoke(Object[] argv) throws Throwable {
  RequestTemplate template = buildTemplateFromArgs.create(argv);
  Options options = findOptions(argv);
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
      // 执行后续逻辑
      return executeAndDecode(template, options);
    } catch (RetryableException e) {
      try {
        retryer.continueOrPropagate(e);
      } catch (RetryableException th) {
        Throwable cause = th.getCause();
        if (propagationPolicy == UNWRAP && cause != null) {
          throw cause;
        } else {
          throw th;
        }
      }
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);
      }
      continue;
    }
  }
}
```

```java
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    // ...
    response = client.execute(request, options);
    // ...
}
```

`org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute`

```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
  try {
    URI asUri = URI.create(request.url());
    String clientName = asUri.getHost();
    URI uriWithoutHost = cleanUrl(request.url(), clientName);
    // 构建ribbon请求
    FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
        this.delegate, request, uriWithoutHost);

    IClientConfig requestConfig = getClientConfig(options, clientName);
    // 包含了负载均衡处理
    return lbClient(clientName)
        .executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
  }
  catch (ClientException e) {
    IOException io = findIOException(e);
    if (io != null) {
      throw io;
    }
    throw new RuntimeException(e);
  }
}
```

`AbstractLoadBalancerAwareClient#executeWithLoadBalancer()`

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

    try {
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        // 执行远程调用
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    } 
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}
```

进⼊submit⽅法，我们进⼀步就会发现使⽤Ribbon在做负载均衡了

```java
public Observable<T> submit(final ServerOperation<T> operation) {
    final ExecutionInfoContext context = new ExecutionInfoContext();
    // ...
    // 进行了负载均衡
    // Use the load balancer
    Observable<T> o = 
            (server == null ? selectServer() : Observable.just(server))
            .concatMap(new Func1<Server, Observable<T>>() {
     // ...
                   
}
```

```java
private Observable<Server> selectServer() {
    return Observable.create(new OnSubscribe<Server>() {
        @Override
        public void call(Subscriber<? super Server> next) {
            try {
                Server server = loadBalancerContext.getServerFromLoadBalancer(loadBalancerURI, loadBalancerKey);   
                next.onNext(server);
                next.onCompleted();
            } catch (Exception e) {
                next.onError(e);
            }
        }
    });
}
```

```java
public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
    // ...
    // Various Supported Cases
    // The loadbalancer to use and the instances it has is based on how it was registered
    // In each of these cases, the client might come in using Full Url or Partial URL
    ILoadBalancer lb = getLoadBalancer();
    if (host == null) {
        // Partial URI or no URI Case
        // well we have to just get the right instances from lb - or we fall back
        if (lb != null){
            // 选择服务器，通过ZoneAwareLoadBalancer
            Server svc = lb.chooseServer(loadBalancerKey);
            if (svc == null){
                throw new ClientException(ClientException.ErrorType.GENERAL,
                        "Load balancer does not have available server for client: "
                                + clientName);
            }
            host = svc.getHost();
            if (host == null){
                throw new ClientException(ClientException.ErrorType.GENERAL,
                        "Invalid Server for :" + svc);
            }
            logger.debug("{} using LB returned Server: {} for request {}", new Object[]{clientName, svc, original});
            return svc;
        } 
      // ...
    }
    // ...
}
```

最后回到`AbstractLoadBalancerAwareClient#executeWithLoadBalancer()`，里面会执行`execute`方法，最终请求的发起使⽤的是`HttpURLConnection`

```java
public Response execute(Request request, Options options) throws IOException {
    HttpURLConnection connection = this.convertAndSend(request, options);
    return this.convertResponse(connection, request);
}
```

# GateWay⽹关组件

## GateWay简介

Spring Cloud GateWay是Spring Cloud的⼀个全新项⽬，⽬标是取代Netflix Zuul，它基于Spring5.0+SpringBoot2.0+WebFlux（基于⾼性能的Reactor模式响应式通信框架Netty，异步⾮阻塞模型）等技术开发，性能⾼于Zuul，官⽅测试， GateWay是Zuul的1.6倍，旨在为微服务架构提供⼀种简单有效的统⼀的API路由管理⽅式。

Spring Cloud GateWay不仅提供统⼀的路由⽅式（反向代理）并且基于 Filter(定义过滤器对请求过滤，完成⼀些功能) 链的⽅式提供了⽹关基本的功能，例如：鉴权、流量控制、熔断、路径重写、⽇志监控等。

![](https://secure2.wostatic.cn/static/oufKgDMGQNT6YCw3iy1mQ8/image.png)

![](https://secure2.wostatic.cn/static/94SnPgyzvEbcA2QHGYbUog/image.png)

## GateWay核⼼概念

Zuul1.x 阻塞式IO 2.x 基于Netty。Spring Cloud GateWay天⽣就是异步⾮阻塞的，基于Reactor模型，⼀个请求—>⽹关根据⼀定的条件匹配—匹配成功之后可以将请求转发到指定的服务地址；⽽在这个过程中，我们可以进⾏⼀些⽐较具体的控制（限流、⽇志、⿊⽩名单）

- 路由（route）： ⽹关最基础的部分，也是⽹关⽐较基础的⼯作单元。路由由⼀个ID、⼀个⽬标URL（最终路由到的地址）、⼀系列的断⾔（匹配条件判断）和 Filter过滤器（精细化控制）组成。如果断⾔为true，则匹配该路由。
- 断⾔（predicates）：参考了Java8中的断⾔java.util.function.Predicate，开发⼈员可以匹配Http请求中的所有内容（包括请求头、请求参数等）（类似于nginx中的location匹配⼀样），如果断⾔与请求相匹配则路由。
- 过滤器（filter）：⼀个标准的Spring webFilter，使⽤过滤器，可以在请求之前或者之后执⾏业务逻辑。

来⾃官⽹的⼀张图

![](https://secure2.wostatic.cn/static/wUrhF41FKffsUYyCLT3F83/image.png)

![](https://secure2.wostatic.cn/static/kN6KWUJrNVJbTntKjzno8S/image.png)

其中， Predicates断⾔就是我们的匹配条件，⽽Filter就可以理解为⼀个⽆所不能的拦截器，有了这两个元素，结合⽬标URL，就可以实现⼀个具体的路由转发。

## GateWay⼯作过程（How It Works）

![](https://secure2.wostatic.cn/static/fL5V2PthURG4MAFAvXZfEq/image.png)

客户端向Spring Cloud GateWay发出请求，然后在GateWay Handler Mapping中找到与请求相匹配的路由，将其发送到GateWay Web Handler； Handler再通过指定的过滤器链来将请求发送到我们实际的服务执⾏业务逻辑，然后返回。过滤器之间⽤虚线分开是因为过滤器可能会在发送代理请求之前（pre）或者之后（post）执⾏业务逻辑。

Filter在“pre”类型过滤器中可以做参数校验、权限校验、流量监控、⽇志输出、协议转换等，在“post”类型的过滤器中可以做响应内容、响应头的修改、⽇志的输出、流量监控等。

GateWay核⼼逻辑：路由转发+执⾏过滤器链

## GateWay应⽤

使⽤⽹关对⾃动投递微服务进⾏代理（添加在它的上游，相当于隐藏了具体微服务的信息，对外暴露的是⽹关）

1. 创建⼯程lagou-cloud-gateway-server-9002导⼊依赖
    
    GateWay不需要使⽤web模块，它引⼊的是WebFlux（类似于SpringMVC）
    

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>lagou-cloud-gateway-9002</artifactId>
    <!--spring boot ⽗启动器依赖-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-commons</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eurekaclient</artifactId>
        </dependency>
        <!--GateWay ⽹关-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-startergateway</artifactId>
        </dependency>
        <!--引⼊webflux-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <!--⽇志依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>
        <!--测试依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--lombok⼯具-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
            <scope>provided</scope>
        </dependency>
        <!--引⼊Jaxb，开始-->
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-core</artifactId>
            <version>2.2.11</version>
        </dependency>
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
            <version>2.2.11</version>
        </dependency>
        <dependency>
            <groupId>org.glassfish.jaxb</groupId>
            <artifactId>jaxb-runtime</artifactId>
            <version>2.2.10-b140310.1920</version>

        </dependency>
        <dependency>
            <groupId>javax.activation</groupId>
            <artifactId>activation</artifactId>
            <version>1.1.1</version>
        </dependency>
        <!--引⼊Jaxb，结束-->
        <!-- Actuator可以帮助你监控和管理Spring Boot应⽤-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starteractuator</artifactId>
        </dependency>
        <!--热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <!--spring cloud依赖版本管理-->
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-clouddependencies</artifactId>
                <version>Greenwich.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <build>
        <plugins>
            <!--编译插件-->
            <plugin>

                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                    <encoding>utf-8</encoding>
                </configuration>
            </plugin>
            <!--打包插件-->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-mavenplugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```

2. application.yml 配置⽂件部分内容

```yaml
server:
  port: 9002
eureka:
  client:
    serviceUrl: # eureka server的路径
      #把 eureka 集群中的所有 url 都填写了进来，也可以只写⼀台，因为各个 eureka server 可以同步注册表
      defaultZone: http://lagoucloudeurekaservera:8761/eureka/,http://lagoucloudeurekaserverb:8762/eureka/ 
  instance:
    #使⽤ip注册，否则会使⽤主机名注册了（此处考虑到对⽼版本的兼容，新版本经过实验都是ip）
  prefer-ip-address: true
  #⾃定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
  instance-id: ${spring.cloud.client.ipaddress}:${spring.application.name}:${server.port}:@project.version@
spring:
  application:
    name: lagou-cloud-gateway
  cloud:
    gateway:
      routes: # 路由可以有多个
        - id: service-autodeliver-router # 我们⾃定义的路由 ID，保持唯⼀
          uri: http://127.0.0.1:8096 # ⽬标服务地址 ⾃动投递微服务（部署多实例） 动态路由： uri配置的应该是⼀个服务名称，⽽不应该是⼀个具体的服务实例的地址
          # gateway⽹关从服务注册中⼼获取实例信息然后负载后路由
          #断⾔：路由条件， Predicate 接受⼀个输⼊参数，返回⼀个布尔值结果。该接⼝包含多种默 认⽅法来将 Predicate 组合成其他复杂的逻辑（⽐如：与，或，⾮）。
          predicates: 
            - Path=/autodeliver/**
        - id: service-resume-router # 我们⾃定义的路由 ID，保持唯⼀
          uri: http://127.0.0.1:8081 # ⽬标服务地址
          #http://localhost:9002/resume/openstate/1545132
          #http://127.0.0.1:8081/openstate/1545132
          #断⾔：路由条件， Predicate 接受⼀个输⼊参数，返回⼀个布尔值结果。该接⼝包含多种默 认⽅法来将 Predicate 组合成其他复杂的逻辑（⽐如：与，或，⾮）。
          predicates: 
            - Path=/resume/**
          filters:
            - StripPrefix=1

```

```
上⾯这段配置的意思是，配置了⼀个 id 为 service-autodeliver-router 的路由规则，当向⽹关发起请求 [http://localhost:9002/autodeliver/checkAndBegin/15](http://localhost:9002/autodeliver/checkAndBegin/15)45132，请求会被分发路由到对应的微服务上
```

## GateWay路由规则详解

Spring Cloud GateWay 帮我们内置了很多 Predicates功能，实现了各种路由匹配规则（通过 Header、请求参数等作为条件）匹配到对应的路由。

![](https://secure2.wostatic.cn/static/6hNW9PthZ4prVQPmNhfP4X/image.png)

- 时间点后匹配·

```YAML
spring:
  cloud:
    gateway:
      routes:
        - id: after_route
          uri: https://example.org
          predicates:
            - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```

- 时间点前匹配

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: before_route
          uri: https://example.org
          predicates:
            - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```

- 时间区间匹配

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: between_route
          uri: https://example.org
          predicates:
            - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
```

- 指定Cookie正则匹配指定值

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: cookie_route
          uri: https://example.org
          predicates:
            - Cookie=chocolate, ch.p
```

- 指定Header正则匹配指定值

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: header_route
          uri: https://example.org
          predicates:
            - Header=X-Request-Id, \d+
```

- 请求Host匹配指定值

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: host_route
          uri: https://example.org
          predicates:
            - Host=**.somehost.org,**.anotherhost.org
```

- 请求Method匹配指定请求⽅式

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: method_route
          uri: https://example.org
          predicates:
            - Method=GET,POST
```

- 请求路径正则匹配

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: path_route
          uri: https://example.org
          predicates:
            - Path=/red/{segment},/blue/{segment}
```

- 请求包含某参数

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: query_route
          uri: https://example.org
          predicates:
            - Query=green
```

- 请求包含某参数并且参数值匹配正则表达式

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: query_route
          uri: https://example.org
          predicates:
            - Query=red, gree.
```

- 远程地址匹配

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: remoteaddr_route
          uri: https://example.org
          predicates:
            - RemoteAddr=192.168.1.1/24
```

## GateWay动态路由详解

GateWay⽀持⾃动从注册中⼼中获取服务列表并访问，即所谓的动态路由

实现步骤如下

1. pom.xml中添加注册中⼼客户端依赖（因为要获取注册中⼼服务列表， 需要引⼊eureka客户端依赖，上面已引入）
    
2. 动态路由配置
    
    注意：动态路由设置时， uri以 lb: //开头（lb代表从注册中⼼获取服务），后⾯是需要转发到的服务名称
    

```yaml
spring:
  application:
    name: lagou-cloud-gateway
  cloud:
    gateway:
      routes: # 路由可以有多个
        - id: service-autodeliver-router # 我们⾃定义的路由 ID，保持唯⼀
          uri: lb://lagou-service-autodeliver
          # gateway⽹关从服务注册中⼼获取实例信息然后负载后路由
          #断⾔：路由条件， Predicate 接受⼀个输⼊参数，返回⼀个布尔值结果。该接⼝包含多种默 认⽅法来将 Predicate 组合成其他复杂的逻辑（⽐如：与，或，⾮）。
          predicates: 
            - Path=/autodeliver/**
```

## GateWay过滤器

### GateWay过滤器简介

从过滤器⽣命周期（影响时机点）的⻆度来说，主要有两个pre和post：

|⽣命周期时机点|作⽤|
|---|---|
|pre|这种过滤器在请求被路由之前调⽤。我们可利⽤这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。|
|post|这种过滤器在路由到微服务以后执⾏。这种过滤器可⽤来为响应添加标准的 HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。|

从过滤器类型的⻆度， Spring Cloud GateWay的过滤器分为GateWayFilter和GlobalFilter两种

|过滤器类型|影响范围|
|---|---|
|GateWayFilter|应⽤到单个路由上|
|GlobalFilter|应⽤到所有的路由上|

如Gateway Filter可以去掉url中的占位后转发路由，⽐如

```yaml
predicates:
  - Path=/resume/**
  filters:
    - StripPrefix=1 # 可以去掉resume之后转发
```

**注意： GlobalFilter全局过滤器是程序员使⽤⽐较多的过滤器，我们主要讲解这种类型**

### ⾃定义全局过滤器实现IP访问限制（⿊⽩名单）

请求过来时，判断发送请求的客户端的ip，如果在⿊名单中，拒绝访问⾃定义GateWay全局过滤器时，我们实现Global Filter接⼝即可，通过全局过滤器可以实现⿊⽩名单、限流等功能。

```java
/**
 * 定义全局过滤器，会对所有路由⽣效
 */
@Slf4j
@Component // 让容器扫描到，等同于注册了
public class BlackListFilter implements GlobalFilter, Ordered {
    // 模拟⿊名单（实际可以去数据库或者redis中查询）
    private static List<String> blackList = new ArrayList<>();
    static {
        blackList.add("0:0:0:0:0:0:0:1"); // 模拟本机地址
    }
    /**
     * 过滤器核⼼⽅法
     * @param exchange 封装了request和response对象的上下⽂
     * @param chain ⽹关过滤器链（包含全局过滤器和单路由过滤器）
     * @return
     */
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 思路：获取客户端ip，判断是否在⿊名单中，在的话就拒绝访问，不在的话就放⾏
        // 从上下⽂中取出request和response对象
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        // 从request对象中获取客户端ip
        String clientIp = request.getRemoteAddress().getHostString();
        // 拿着clientIp去⿊名单中查询，存在的话就决绝访问
        if(blackList.contains(clientIp)) {
            // 决绝访问，返回
            response.setStatusCode(HttpStatus.UNAUTHORIZED); // 状态码
            log.debug("=====>IP:" + clientIp + " 在⿊名单中，将被拒绝访问！ ");
            String data = "Request be denied!"; 
            DataBuffer wrap = response.bufferFactory().wrap(data.getBytes());
            return response.writeWith(Mono.just(wrap));
        }
        // 合法请求，放⾏，执⾏后续的过滤器
        return chain.filter(exchange);
    }

    /**
     * 返回值表示当前过滤器的顺序(优先级)，数值越⼩，优先级越⾼
     * @return
     */
    @Override
    public int getOrder() {
        return 0;
    }
}

```

# GateWay⾼可⽤

⽹关作为⾮常核⼼的⼀个部件，如果挂掉，那么所有请求都可能⽆法路由处理，因此我们需要做GateWay的⾼可⽤。

GateWay的⾼可⽤很简单： 可以启动多个GateWay实例来实现⾼可⽤，在GateWay的上游使⽤Nginx等负载均衡设备进⾏负载转发以达到⾼可⽤的⽬的。

启动多个GateWay实例（假如说两个，⼀个端⼝9002，⼀个端⼝9003），剩下的就是使⽤Nginx等完成负载代理即可。示例如下：

```nginx
#配置多个GateWay实例
upstream gateway {
    server 127.0.0.1:9002;
    server 127.0.0.1:9003;
}
location / {
    proxy_pass http://gateway;
}
```

# Spring Cloud Config 分布式配置中⼼

[[MicroService Design#分布式配置中⼼]]

## Spring Cloud Config

### Config简介

Spring Cloud Config是⼀个分布式配置管理⽅案，包含了 Server端和 Client端两个部分。

![](https://secure2.wostatic.cn/static/wByQCanRHUHgyFVoyZuif4/image.png)

![](https://secure2.wostatic.cn/static/mThGcspJFHT9SAH9oS9VQQ/image.png)

Server 端：提供配置⽂件的存储、以接⼝的形式将配置⽂件的内容提供出去，通过使⽤@EnableConfigServer注解在 Spring boot 应⽤中⾮常简单的嵌⼊

Client 端：通过接⼝获取配置数据并初始化⾃⼰的应⽤

### Config分布式配置应⽤

说明： Config Server是集中式的配置服务，⽤于集中管理应⽤程序各个环境下的配置。 默认使⽤Git存储配置⽂件内容，也可以SVN。

⽐如，我们要对“简历微服务”的application.yml进⾏管理（区分开发环境、测试环境、⽣产环境）

- 登录码云，创建项⽬lagou-config-repo
    
- 上传yml配置⽂件，命名规则如下：{application}-{profile}.yml 或者 {application}-{profile}.properties，其中， application为应⽤名称， profile指的是环境（⽤于区分开发环境，测试环境、⽣产环境等）
    
    示例： lagou-service-resume-dev.yml、 lagou-service-resume-test.yml、 lagouservice-resume-prod.yml
    
- 构建Config Server统⼀配置中⼼
    

**服务端构建**

1. 新建SpringBoot⼯程，引⼊依赖坐标（需要注册⾃⼰到Eureka）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>lagou-parent</artifactId>
        <groupId>com.lagou.edu</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>lagou-config1</artifactId>
    <dependencies>
        <!--eureka client 客户端依赖引⼊-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eurekaclient</artifactId>
        </dependency>
        <!--config配置中⼼服务端-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
    </dependencies>
</project>
```

2. 配置启动类，使⽤注解@EnableConfigServer开启配置中⼼服务器功能

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer // 开启配置服务器功能
public class ConfigApp9006 {
    public static void main(String[] args) {
        SpringApplication.run(ConfigApp9003.class,args);
    }
}
```

3. application.yml配置

```yaml
server:
  port: 9006
#注册到Eureka服务中⼼
eureka:
  client:
    service-url:
      # 注册到集群，就把多个Eurekaserver地址使⽤逗号连接起来即可；注册到单实例（⾮集群模式），那就写⼀个就ok
      defaultZone: http://LagouCloudEurekaServerA:8761/eureka,http://LagouCloudEurekaServerB:8762/eureka
  instance:
    prefer-ip-address: true #服务实例中显示ip，⽽不是显示主机名（兼容⽼的eureka版本）
    # 实例名称： 192.168.1.103:lagou-service-resume:8080，我们可以⾃定义它
    instance-id: ${spring.cloud.client.ipaddress}:${spring.application.name}:${server.port}:@project.version@
spring:
  application:
    name: lagou-service-autodeliver
  cloud:
    config:
      server:
        git:
          uri: https://github.com/5173098004/lagou-config-repo.git
          #配置git服务地址
          username: 517309804@qq.com #配置git⽤户名
          password: yingdian12341 #配置git密码
          search-paths:
            - lagou-config-repo
      # 读取分⽀
      label: master
#针对的被调⽤⽅微服务名称,不加就是全局⽣效
#lagou-service-resume:
# ribbon:
# NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #负载策略调整
# springboot中暴露健康检查等断点接⼝
management:
  endpoints:
    web:
      exposure:
        include: "*"
# 暴露健康接⼝的细节
endpoint:
  health:
    show-details: always

```

4. 测试访问： [http://localhost:9006/master/lagou-service-resume-dev.yml，查看到](http://localhost:9006/master/lagou-service-resume-dev.yml%EF%BC%8C%E6%9F%A5%E7%9C%8B%E5%88%B0)配置⽂件内容

**构建Client客户端（在已有简历微服务基础上） **

1. 已有⼯程中添加依赖坐标

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

2. application.yml修改为bootstrap.yml配置⽂件
    
    bootstrap.yml是系统级别的，优先级⽐application.yml⾼，应⽤启动时会检查这个配置⽂件，在这个配置⽂件中指定配置中⼼的服务地址，会⾃动拉取所有应⽤配置并且启⽤。
    
    （主要是把与统⼀配置中⼼连接的配置信息放到bootstrap.yml）
    
    注意：需要统⼀读取的配置信息，从集中配置中⼼获取bootstrap.yml
    

```yaml
server:
  port: 8080
spring:
  application:
    name: lagou-service-resume
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
        #避免将驼峰命名转换为下划线命名
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
  cloud:
  # config客户端配置,和ConfigServer通信，并告知ConfigServer希望获取的配置信息在哪个⽂件中
    config:
      name: lagou-service-resume #配置⽂件名称
      profile: dev #后缀名称
      label: master #分⽀名称
      uri: http://localhost:9006 #ConfigServer配置中⼼地址
#注册到Eureka服务中⼼
eureka:
  client:
    service-url:
    # 注册到集群，就把多个Eurekaserver地址使⽤逗号连接起来即可；注册到单实例（⾮集群模式），那就写⼀个就ok
      defaultZone: http://LagouCloudEurekaServerA:8761/eureka,http://LagouCloudEurekaServerB:8762/eureka
    instance:
      prefer-ip-address: true #服务实例中显示ip，⽽不是显示主机名（兼容⽼的eureka版本）
      # 实例名称： 192.168.1.103:lagou-service-resume:8080，我们可以⾃定义它
      instance-id: ${spring.cloud.client.ipaddress}:${spring.application.name}:${server.port}:@project.version@
    # ⾃定义Eureka元数据
    metadata-map:
      cluster: cl1
      region: rn1
management:
  endpoints:
    web:
      exposure:
        include: "*"

```

## Config配置⼿动刷新

不⽤重启微服务，只需要⼿动的做⼀些其他的操作（访问⼀个地址/refresh）刷新，之后再访问即可

此时，客户端取到了配置中⼼的值，但当我们修改GitHub上⾯的值时，服务端（Config Server）能实时获取最新的值，但客户端（Config Client）读的是缓存，⽆法实时获取最新值。 Spring Cloud已 经为我们解决了这个问题，那就是客户端使⽤post去触发refresh，获取最新数据。

1. Client客户端添加依赖springboot-starter-actuator（已添加）
2. Client客户端bootstrap.yml中添加配置（暴露通信端点）

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh
# 也可以暴露所有的端⼝
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

3. Client客户端使⽤到配置信息的类上添加@RefreshScope4）
    
    ⼿动向Client客户端发起POST请求， [http://localhost:8080/actuator/refresh，](http://localhost:8080/actuator/refresh%EF%BC%8C)刷新配置信息
    
    注意：⼿动刷新⽅式避免了服务重启（流程： Git改配置—>for循环脚本⼿动刷新每个微服务）
    
    思考：可否使⽤⼴播机制，⼀次通知，处处⽣效，⽅便⼤范围配置刷新？
    

## Config配置⾃动更新

实现⼀次通知处处⽣效

拉勾内部做分布式配置，⽤的是zk（存储+通知）， zk中数据变更，可以通知各个监听的客户端，客户端收到通知之后可以做出相应的操作（内存级别的数据直接⽣效，对于数据库连接信息、连接池等信息变化更新的，那么会在通知逻辑中进⾏处理，⽐如重新初始化连接池）

在微服务架构中，我们可以结合消息总线（Bus）实现分布式配置的⾃动更新（Spring Cloud Config+Spring Cloud Bus）

### 消息总线Bus

所谓消息总线Bus，即我们经常会使⽤MQ消息代理构建⼀个共⽤的Topic，通过这个Topic连接各个微服务实例， MQ⼴播的消息会被所有在注册中⼼的微服务实例监听和消费。 换⾔之就是通过⼀个主题连接各个微服务，打通脉络。

Spring Cloud Bus（基于MQ的，⽀持RabbitMq/Kafka） 是Spring Cloud中的消息总线⽅案， Spring Cloud Config + Spring Cloud Bus 结合可以实现配置信息的⾃动更新。

![](https://secure2.wostatic.cn/static/x8GoodCdVV7NKjnaMKfL3K/image.png)

### Spring Cloud Config+Spring Cloud Bus 实现⾃动刷新

MQ消息代理，我们还选择使⽤RabbitMQ， ConfigServer和ConfigClient都添加都消息总线的⽀持以及与RabbitMq的连接信息

1. Config Server服务端添加消息总线⽀持

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

2. ConfigServer添加配置

```yaml
spring:
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: guest
    password: guest
```

3. 微服务暴露端⼝

```yaml
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
# 建议暴露所有的端⼝
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

4. 重启各个服务，更改配置之后，向配置中⼼服务端发送post请求[http://localhost:9003/actuator/bus-refresh，各个客户端配置即可⾃动刷新](http://localhost:9003/actuator/bus-refresh%EF%BC%8C%E5%90%84%E4%B8%AA%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%85%8D%E7%BD%AE%E5%8D%B3%E5%8F%AF%E2%BE%83%E5%8A%A8%E5%88%B7%E6%96%B0)在⼴播模式下实现了⼀次请求，处处更新。
    
    如果我只想定向更新呢？在发起刷新请求的时候http://localhost:9006/actuator/bus-refresh/lagou-serviceresume:8081即为最后⾯跟上要定向刷新的实例的 **服务名:端⼝号**即可
    

# Spring Cloud Stream消息驱动组件

Spring Cloud Stream 消息驱动组件帮助我们更快速，更⽅便，更友好的去构建消息驱动微服务的。

## Stream解决的痛点问题

MQ消息中间件⼴泛应⽤在应⽤解耦合、异步消息处理、流量削峰等场景中。不同的MQ消息中间件内部机制包括使⽤⽅式都会有所不同，⽐如RabbitMQ中有Exchange（交换机/交换器）这⼀概念， kafka有Topic、 Partition分区这些概念， MQ消息中间件的差异性不利于我们上层的开发应⽤，当我们的系统希望从原有的RabbitMQ切换到Kafka时，我们会发现⽐较困难，很多要操作可能重来（因为应⽤程序和具体的某⼀款MQ消息中间件耦合在⼀起了）。

Spring Cloud Stream进⾏了很好的上层抽象，可以让我们与具体消息中间件解耦合，屏蔽掉了底层具体MQ消息中间件的细节差异，就像Hibernate屏蔽掉了具体数据库（Mysql/Oracle⼀样）。如此⼀来，我们学习、开发、维护MQ都会变得轻松。

⽬前Spring Cloud Stream⽀持RabbitMQ和Kafka。

本质： 屏蔽掉了底层不同MQ消息中间件之间的差异，统⼀了MQ的编程模型，降低了学习、开发、维护MQ的成本

## Stream重要概念

Spring Cloud Stream 是⼀个构建消息驱动微服务的框架。应⽤程序通过inputs（相当于消息消费者consumer）或者outputs（相当于消息⽣产者producer）来与Spring Cloud Stream中的binder对象交互，⽽Binder对象是⽤来屏蔽底层MQ细节的，它负责与具体的消息中间件交互。

说⽩了：对于我们来说，只需要知道如何使⽤Spring Cloud Stream与Binder对象交互即可

![](https://secure2.wostatic.cn/static/849iVhy3sQ6FGN8ykKG5qC/image.png)

![](https://secure2.wostatic.cn/static/d5o6YXG3XW54x5ijgpoK45/image.png)

**Binder绑定器 **

Binder绑定器是Spring Cloud Stream 中⾮常核⼼的概念，就是通过它来屏蔽底层不同MQ消息中间件的细节差异，当需要更换为其他消息中间件时，我们需要做的就是更换对应的Binder绑定器⽽不需要修改任何应⽤逻辑（Binder绑定器的实现是框架内置的， Spring Cloud Stream⽬前⽀持Rabbit、 Kafka两种消息队列）

## 传统MQ模型与Stream消息驱动模型

![](https://secure2.wostatic.cn/static/2LePB3iq5JmJQq749h1ycx/image.png)

## Stream消息通信⽅式及编程模型

### Stream消息通信⽅式

Stream中的消息通信⽅式遵循了发布—订阅模式。

在Spring Cloud Stream中的消息通信⽅式遵循了发布-订阅模式，当⼀条消息被投递到消息中间件之 后，它会通过共享的 Topic 主题进⾏⼴播，消息消费者在订阅的主题中收到它并触发⾃身的业务逻辑处理。这⾥所提到的 Topic 主题是Spring Cloud Stream中的⼀个抽象概念，⽤来代表发布共享消息给消 费者的地⽅。在不同的消息中间件中， Topic 可能对应着不同的概念，⽐如：在RabbitMQ中的它对应了Exchange、在Kakfa中则对应了Kafka中的Topic。

### Stream编程注解

如下的注解⽆⾮在做⼀件事，把我们结构图中那些组成部分上下关联起来，打通通道（这样的话⽣产者的message数据才能进⼊mq， mq中数据才能进⼊消费者⼯程） 。

|注解|描述|
|---|---|
|@Input（在消费者⼯程中使⽤）|注解标识输⼊通道，通过该输⼊通道接收到的消息进⼊应⽤程序|
|@Output（在⽣产者⼯程中使⽤）|注解标识输出通道，发布的消息将通过该通道离开应⽤程序|
|@StreamListener（在消费者⼯程中使⽤，监听message的到来）|监听队列，⽤于消费者的队列的消息的接收（有消息监听.....）|
|@EnableBinding|把Channel和Exchange（对于RabbitMQ）绑定在⼀起|

接下来，我们创建三个⼯程（我们基于RabbitMQ， RabbitMQ的安装和使⽤这⾥不再说明）

- lagou-cloud-stream-producer-9090， 作为⽣产者端发消息
- lagou-cloud-stream-consumer-9091，作为消费者端接收消息
- lagou-cloud-stream-consumer-9092，作为消费者端接收消息

### Stream消息驱动之开发⽣产者端

1. 在lagou_parent下新建⼦module： lagou-cloud-stream-producer-9090
2. pom.xml中添加依赖

```xml
<!--eureka client 客户端依赖引⼊-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eurekaclient</artifactId>
</dependency>
<!--spring cloud stream 依赖（rabbit） -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

3. application.yml添加配置

```yaml
server:
  port: 9090
spring:
  application:
    name: lagou-cloud-stream-producer
  cloud:
    stream:
      binders: # 绑定MQ服务信息（此处我们是RabbitMQ）
        lagouRabbitBinder: # 给Binder定义的名称，⽤于后⾯的关联
          type: rabbit # MQ类型，如果是Kafka的话，此处配置kafka
          environment: # MQ环境配置（⽤户名、密码等）
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 关联整合通道和binder对象
        output: # output是我们定义的通道名称，此处不能乱改
          destination: lagouExchange # 要使⽤的Exchange名称（消息队列主题名称）
          content-type: text/plain # application/json # 消息类型设置，⽐如json
          binder: lagouRabbitBinder # 关联MQ服务
eureka:
  client:
    serviceUrl: # eureka server的路径
      defaultZone: http://lagoucloudeurekaservera:8761/eureka/,http://lagoucloudeurekaserverb:8762/eureka/ #把 eureka 集群中的所有 url 都填写了进来，也可以只写⼀台，因为各个 eureka server 可以同步注册表
instance:
  prefer-ip-address: true #使⽤ip注册
```

4. 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class StreamProducerApplication9090 {
  public static void main(String[] args) {
    SpringApplication.run(StreamProducerApplication9090.class,args);
  }
}
```

5. 业务类开发（发送消息接⼝、接⼝实现类、 Controller）接⼝

```java
public interface IMessageProducer {
  public void sendMessage(String content);
}
```

```
实现类
```

```java
// Source.class⾥⾯就是对输出通道的定义（这是Spring Cloud Stream内置的通道封装）
@EnableBinding(Source.class)
public class MessageProducerImpl implements IMessageProducer {
  // 将MessageChannel的封装对象Source注⼊到这⾥使⽤
  @Autowired
  private Source source;
  @Override
  public void sendMessage(String content) {
    // 向mq中发送消息（并不是直接操作mq，应该操作的是spring cloudstream）
    // 使⽤通道向外发出消息(指的是Source⾥⾯的output通道)
    source.output().send(MessageBuilder.withPayload(content).build());
  }
}

```

```
测试类
```

```java
@SpringBootTest(classes = {StreamProducerApplication9090.class})
@RunWith(SpringJUnit4ClassRunner.class)
public class MessageProducerTest {
    @Autowired
    private IMessageProducer iMessageProducer;

    @Test
    public void testSendMessage() {
        iMessageProducer.sendMessage("hello world-lagou101");
    }
}
```

### Stream消息驱动之开发消费者端

此处我们记录lagou-cloud-stream-consumer-9091编写过程， 9092⼯程类似

1. application.yml

```yaml
server:
  port: 9091
spring:
  application:
    name: lagou-cloud-stream-consumer
  cloud:
    stream:
      binders: # 绑定MQ服务信息（此处我们是RabbitMQ）
        lagouRabbitBinder: # 给Binder定义的名称，⽤于后⾯的关联
          type: rabbit # MQ类型，如果是Kafka的话，此处配置kafka
          environment: # MQ环境配置（⽤户名、密码等）
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 关联整合通道和binder对象
        intput: # output是我们定义的通道名称，此处不能乱改
          destination: lagouExchange # 要使⽤的Exchange名称（消息队列主题名称）
          content-type: text/plain # application/json # 消息类型设置，⽐如json
          binder: lagouRabbitBinder # 关联MQ服务
          group: lagou001
eureka:
  client:
    serviceUrl: # eureka server的路径
      defaultZone: http://lagoucloudeurekaservera:8761/eureka/,http://lagoucloudeurekaserverb:8762/eureka/ #把 eureka 集群中的所有 url 都填写了进来，也可以只写⼀台，因为各个 eureka server 可以同步注册表
instance:
  prefer-ip-address: true #使⽤ip注册
```

2. 消息消费者监听

```java
@EnableBinding(Sink.class)
public class MessageConsumerService {
    @StreamListener(Sink.INPUT)
    public void recevieMessages(Message<String> message) {
      System.out.println("=========接收到的消息： " + message);
    }
}
```

## Stream⾼级之⾃定义消息通道

Stream 内置了两种接⼝Source和Sink分别定义了 binding 为 “input” 的输⼊流和 “output” 的输出流，我们也可以⾃定义各种输⼊输出流（通道），但实际我们可以在我们的服务中使⽤多个binder、多个输⼊通道和输出通道，然⽽默认就带了⼀个input的输⼊通道和⼀个output的输出通道，怎么办？

我们是可以⾃定义消息通道的，学着Source和Sink的样⼦，给你的通道定义个⾃⼰的名字，多个输⼊通道和输出通道是可以写在⼀个类中的。

定义接⼝

```java
interface CustomChannel {
    String INPUT_LOG = "inputLog";
    String OUTPUT_LOG = "outputLog";
    @Input(INPUT_LOG)
    SubscribableChannel inputLog();
    @Output(OUTPUT_LOG)
    MessageChannel outputLog();
}
```

**如何使⽤？ **

1. 在 @EnableBinding 注解中，绑定⾃定义的接⼝
2. 使⽤ @StreamListener 做监听的时候，需要指定 CustomChannel.INPUT_LOG

```yaml
bindings:
  inputLog:
    destination: lagouExchange
  outputLog:
    destination: eduExchange
```

## Stream⾼级之⾃定义消息通道

如上我们的情况，消费者端有两个（消费同⼀个MQ的同⼀个主题），但是呢我们的业务场景中希望这个主题的⼀个Message只能被⼀个消费者端消费处理，此时我们就可以使⽤消息分组。

解决的问题：能解决消息重复消费问题

我们仅仅需要在服务消费者端设置 spring.cloud.stream.bindings.input.group 属性，多个消费者实例配置为同⼀个group名称（在同⼀个group中的多个消费者只有⼀个可以获取到消息并消费）。

![](https://secure2.wostatic.cn/static/h1sEiSjCchT7xMqf2Jpqfm/image.png)

# 常⻅问题及解决⽅案

常⻅问题及解决⽅案

本部分主要讲解 Eureka 服务发现慢的原因， Spring Cloud 超时设置问题。

如果你刚刚接触Eureka，对Eureka的设计和实现都不是很了解，可能就会遇到⼀些⽆法快速解决的问题，这些问题包括：新服务上线后，服务消费者不能访问到刚上线的新服务，需要过⼀段时间后才能访问？或是将服务下线后，服务还是会被调⽤到，⼀段时候后才彻底停⽌服务，访问前期会导致频繁报错？这些问题还会让你对Spring Cloud 产⽣严重的怀疑，这难道不是⼀个 Bug?

问题场景

上线⼀个新的服务实例，但是服务消费者⽆感知，过了⼀段时间才知道某⼀个服务实例下线了，服务消费者⽆感知，仍然向这个服务实例在发起请求这其实就是服务发现的⼀个问题，当我们需要调⽤服务实例时，信息是从注册中⼼Eureka获取的，然后通过Ribbon选择⼀个服务实例发起调⽤，如果出现调⽤不到或者下线后还可以调⽤的问题，原因肯定是服务实例的信息更新不及时导致的。

**Eureka 服务发现慢的原因 **

Eureka 服务发现慢的原因主要有两个，⼀部分是因为服务缓存导致的，另⼀部分是因为客户端缓存导致的。

![](https://secure2.wostatic.cn/static/xv1AdAHNwC5NXCee15hzaP/image.png)

- 服务端缓存
    
    服务注册到注册中⼼后，服务实例信息是存储在注册表中的，也就是内存中。但Eureka为了提⾼响应速度，在内部做了优化，加⼊了两层的缓存结构，将Client需要的实例信息，直接缓存起来，获取的时候直接从缓存中拿数据然后响应给 Client。
    
    - 第⼀层缓存是readOnlyCacheMap， readOnlyCacheMap是采⽤ConcurrentHashMap来存储数据的，主要负责定时与readWriteCacheMap进⾏数据同步，默认同步时间为 30 秒⼀次。
    - 第⼆层缓存是readWriteCacheMap， readWriteCacheMap采⽤Guava来实现缓存。缓存过期时间默认为180秒，当服务下线、过期、注册、状态变更等操作都会清除此缓存中的数据。
    
    Client获取服务实例数据时，会先从⼀级缓存中获取，如果⼀级缓存中不存在，再从⼆级缓存中获取，如果⼆级缓存也不存在，会触发缓存的加载，从存储层拉取数据到缓存中，然后再返回给 Client。
    
    Eureka 之所以设计⼆级缓存机制，也是为了提⾼ Eureka Server 的响应速度，缺点是缓存会导致 Client 获取不到最新的服务实例信息，然后导致⽆法快速发现新的服务和已下线的服务。
    
    了解了服务端的实现后，想要解决这个问题就变得很简单了，
    
    - 我们可以缩短只读缓存的更新时间（eureka.server.response-cache-update-interval-ms）让服务发现变得更加及时，或者直接将只读缓存关闭（eureka.server.use-read-onlyresponse-cache=false），多级缓存也导致C层⾯（数据⼀致性）很薄弱。
    - Eureka Server 中会有定时任务去检测失效的服务，将服务实例信息从注册表中移除，也可以将这个失效检测的时间缩短，这样服务下线后就能够及时从注册表中清除。
- 客户端缓存客户端缓存主要分为两块内容，⼀块是 Eureka Client 缓存，⼀块是Ribbon 缓存。
    
    - Eureka Client 缓存
        
        EurekaClient负责跟EurekaServer进⾏交互，在EurekaClient中的com.netflix.discovery.DiscoveryClient.initScheduledTasks() ⽅法中，初始化了⼀个 CacheRefreshThread 定时任务专⻔⽤来拉取 Eureka Server 的实例信息到本地。
        
        - 所以我们需要缩短这个定时拉取服务信息的时间间隔（eureka.client.registryFetchIntervalSeconds）来快速发现新的服务。
    - Ribbon 缓存Ribbon会从EurekaClient中获取服务信息， ServerListUpdater是Ribbon中负责服务实例更新的组件，默认的实现是PollingServerListUpdater，通过线程定时去更新实例信息。定时刷新的时间间隔默认是30秒，当服务停⽌或者上线后，这边最快也需要30秒才能将实例信息更新成最新的。
        
        - 我们可以将这个时间调短⼀点，⽐如 3 秒。刷新间隔的参数是通过 getRefreshIntervalMs ⽅法来获取的，⽅法中的逻辑也是从Ribbon 的配置中进⾏取值的。

将这些服务端缓存和客户端缓存的时间全部缩短后，跟默认的配置时间相⽐，快了很多。我们通过调整参数的⽅式来尽量加快服务发现的速度，但是还是不能完全解决报错的问题，间隔时间设置为3秒，也还是会有间隔。所以我们⼀般都会开启重试功能，当路由的服务出现问题时，可以重试到另⼀个服务来保证这次请求的成功。

**Spring Cloud 各组件超时 **

在SpringCloud中，应⽤的组件较多，只要涉及通信，就有可能会发⽣请求超时。那么如何设置超时时间？ 在 Spring Cloud 中，超时时间只需要重点关注 Ribbon 和Hystrix 即可。

Ribbon如果采⽤的是服务发现⽅式，就可以通过服务名去进⾏转发，需要配置Ribbon的超时。 Rbbon的超时可以配置全局的ribbon.ReadTimeout和ribbon.ConnectTimeout。也可以在前⾯指定服务名，为每个服务单独配置，⽐如user-service.ribbon.ReadTimeout。

user-service.ribbon.ReadTimeout。

其次是Hystrix的超时配置， Hystrix的超时时间要⼤于Ribbon的超时时间，因为Hystrix将请求包装了起来，特别需要注意的是，如果Ribbon开启了重试机制，⽐如重试3 次， Ribbon 的超时为 1 秒，那么 Hystrix 的超时时间应该⼤于 3 秒，否则就会出现 Ribbon 还在重试中，⽽ Hystrix 已经超时的现象。

HystrixHystrix全局超时配置就可以⽤default来代替具体的command名称。

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=3000

如果想对具体的 command 进⾏配置，那么就需要知道 command 名称的⽣成规则，才能准确的配置。

如果我们使⽤ @HystrixCommand 的话，可以⾃定义 commandKey。如果使⽤FeignClient的话，可以为FeignClient来指定超时时间：hystrix.command.UserRemoteClient.execution.isolation.thread.timeoutInMilliseconds = 3000

如果想对FeignClient中的某个接⼝设置单独的超时，可以在FeignClient名称后加上

具体的⽅法：

hystrix.command.UserRemoteClient#getUser(Long).execution.isolation.thread.timeoutInMilliseconds = 3000

FeignFeign本身也有超时时间的设置，如果此时设置了Ribbon的时间就以Ribbon的时间为准，如果没设置Ribbon的时间但配置了Feign的时间，就以Feign的时间为准。 Feign的时间同样也配置了连接超时时间（feign.client.config.服务名

称.connectTimeout）和读取超时时间（feign.client.config.服务名称.readTimeout）。

建议，我们配置Ribbon超时时间和Hystrix超时时间即可。

