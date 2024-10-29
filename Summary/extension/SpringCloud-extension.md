
## Eureka核⼼源码剖析

### Eureka Server启动过程

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

### Eureka Server服务接⼝暴露策略

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

### Eureka Server服务注册接⼝（接受客户端注册服务）

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

### Eureka Server服务续约接⼝（接受客户端续约）

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

### Eureka Client注册服务

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

### Eureka Client下架服务

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

### Eureka Client⼼跳续约

参考前面

## @LoadBalanced源码剖析

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

## Feign源码分析

### `@EnableFeignClients` 

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
```

### `FeignClientsRegistrar`

`@EnableFeignClients`类中导入了一个类`FeignClientsRegistrar`

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

### `registerDefaultConfiguration`方法

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

### `registerFeignClients(metadata,  registry);`方法

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

### `loadBalance`方法

url为空，调用`loadBalance`方法

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

`org.springframework.cloud.openfeign.HystrixTargeter#target`

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

在`ReflectiveFeign`中创建了代理对象

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
