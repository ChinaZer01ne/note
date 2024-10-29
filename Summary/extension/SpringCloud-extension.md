
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

- url为空，调用`loadBalance`方法

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
