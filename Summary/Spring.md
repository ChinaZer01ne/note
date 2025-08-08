#flashcards 
# Spring初始化流程
Spring的初始化流程主要是执行`AbstractApplicationContext`类的`refresh`方法。
- refresh  
    - prepareRefresh  
    - obtainFreshBeanFactory  
        * createBeanFactory：创建容器DefaultListableBeanFactory
        * loadBeanDefinitions：加载配置文件
            - loadBeanDefinitions ：加载成Document  
            - registerBeanDefinitions ：注册BeanDefinition
                - 以BeanDefinition的形式注册到容器`DefaultListableBeanFactory#beanDefinitionMap`中  
                    `BeanDefinitionNames`：beanName  
                    `BeanDefinitionMap`: beanName -> BeanDefinition
    - prepareBeanFactory：
        - 主要是属性值的设置，比如SPEL，Aware接口，监听器
    - postProcessBeanFactory  
    - invokeBeanFactoryPostProcessors：执行[BeanFactoryPostProcessor](#Spring有哪些重要的BeanFactoryPostProcessor？)
    - registerBeanPostProcessors：创建所有BeanPostProcessor  
    - initMessageSource：国际化相关  
    - initApplicationEventMulticaster：发布事件初始化相关  
    - onRefresh  
    - registerListeners：注册事件监听器  
    - finishBeanFactoryInitialization  
        - getBean方法：创建初始化Bean  
	        - [FactoryBean](#什么是FactoryBean？)处理
		        - 所有的FactoryBean需要延迟创建的单例对象（getObject方法创建的）都保存在factoryBeanObjectCache中，原始的FactoryBean对象在一级缓存中
			- 普通Bean处理  
				- resolveBeforeInstantiation  
					- 会执行InstantiationAwareBeanPostProcessor该类方法可以做扩展来提早初始化bean，而不用一定执行doCreateBean，默认空实现  
					- AOP在这个地方创建代理对象  
				- doCreateBean  
					- createBeanInstance：创建对象
					- applyMergedBeanDefinitionPostProcessors：注解解析
						- 通过执行`MergedBeanDefinitionPostProcessor`的子类`postProcessMergedBeanDefinition`方法来解析注解，提前保存到BeanDefinition
							- `CommonAnnotationBeanPostProcessor`  
								- @PostConstruct  
								- @PreDestroy  
								- @Resource  
							- `AutowiredAnnotationBeanPostProcessor`  
								- @Autowired  
								- @Value  
								- @Inject  
					- populateBean：注入属性（解决循环依赖问题）  
						- 执行实例化的后置处理：
							- `InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`  
						- 根据配置文件的autowire属性注入  
							- autowireByType
							- autowireByName
						- @Resource、@Autowire注解标识的属性注入 
							- 将之前MergedBeanDefinitionPostProcessor处理的注解字段，在此处进行赋值操作，主要是调用`InstantiationAwareBeanPostProcessor#postProcessProperties`方法  
						- 配置文件的property属性注入  
							- applyPropertyValues
					- initializeBean：初始化
						- Aware接口回调
							- `BeanNameAware`
							- `BeanClassLoaderAware`  
							- `BeanFactoryAware`  
						- 执行[BeanPostProcessor#postProcessBeforeInitialization](#Spring有哪些重要的BeanPostProcessor？)  
						- 初始化
							- `InitializingBean#afterPropertiesSet`  
							- `init-method`  
						- 执行[BeanPostProcessor#postProcessAfterInitialization](#Spring有哪些重要的BeanPostProcessor？)  
        - Bean初始化后的回调处理  
    - finishRefresh：发布ContextRefreshedEvent事件

![](spring流程.png)
# Spring Bean的生命周期
1. **实例化（Instantiation）**：[创建对象](#Spring创建Bean有哪些方式？)
2. **属性赋值（Population）**：主要是在`populateBean`方法中，在属性赋值阶段，Spring将相应的属性值设置到 Bean 实例中。这包括基于 XML 配置、注解的属性注入。
3. **初始化（Initialization）**：在初始化阶段，Spring 主要执行`InitializingBean#afterPropertiesSet()` 方法，`BeanPostProcessor`以及配置的 `init-method` 属性的指定的初始化方法。
4. **使用（In Use）**：在初始化完成后，Bean 就处于可用状态。
5. **销毁（Destruction）**：在容器关闭或显式销毁 Bean 的时候，会触发 Bean 的销毁阶段。主要执行`DisposableBean#destroy() `方法，以及配置的`destroy-method` 属性指定的销毁方法。
# Spring创建Bean有哪些方式？

 1. 实现InstantiationAwareBeanPostProcessor  
 2. FactoryBean#getObject  
 3. 反射  
 4. factory-method  
 5. supplier

# Autowired和Resource区别

`@Autowired` 是先根据类型（byType）查找，如果存在多个 Bean 再根据名称（byName）进行查找，它的具体查找流程如下：

![](autowired注入方式.png)

`@Resource` 是先根据名称（byName）查找，如果（byType）查找不到，再根据类型进行查找，它的具体流程如下图所示：

![](resource注入方式.png)
# FactoryBean与BeanFactory
## 什么是BeanFactory？::
BeanFactory是Spring框架中的一个关键接口，用于管理和获取Bean对象。它是Spring IoC（Inversion of Control，控制反转）容器的核心部分之一。BeanFactory负责创建、配置和管理Bean的生命周期。

## 什么是FactoryBean？::
FactoryBean是是一种特殊类型的Bean，用于创建其他Bean实例。与普通的Bean不同，FactoryBean本身是一个工厂（Factory），用于生成其他Bean的实例。

FactoryBean接口定义了一个方法**getObject()**，该方法返回由FactoryBean创建的Bean实例。通过实现FactoryBean接口，开发人员可以自定义Bean的创建过程，并在需要时进行一些特殊的处理。例如，可以在创建Bean之前或之后执行一些自定义逻辑，或者返回不同的Bean实例，甚至返回原型（Prototype）作用域的Bean。

**如果一个Bean实现了FactoryBean接口，Spring容器将调用该Bean的getObject()方法获取实际的Bean实例，而不是直接返回FactoryBean本身**。这使得开发人员能够通过FactoryBean实现更灵活和可定制的Bean创建过程。

FactoryBean包含三个方法  
* getObject：延迟初始化的对象  
* isSingleton：延迟对象是不是单例的，单例的会单独放到缓存  
* getObjectType：获取对象类型

# BeanFactory和ApplicationContext 

**基本区别**

BeanFactory：BeanFacotry是Spring中最原始的Factory，里面最低层的接口，提供了最简单的容器的功能，只提供了实例化对象和拿对象的功能。它没有AOP功能、Web应用功能等等。

ApplicationContext：应用上下文，继承BeanFactory接口（因而提供BeanFactory所有的功能），ApplicationContext以一种更面向框架的方式工作以及对上下文进行分层和实现继承。它是Spring的一各更高级的容器，提供了更多的有用的功能：

- 国际化（MessageSource）（ApplicationContext.getMessage()拿到国际化消息）
- 访问资源，如URL和文件（ResourceLoader） （ApplicationContext acxt =new ClassPathXmlApplicationContext("/applicationContext.xml");能直接读取文件内容）
- 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层
- 消息发送、响应机制（ApplicationEventPublisher）
- AOP（拦截器）

**两者装载bean的区别**

BeanFactory在启动的时候不会去实例化Bean，只有从容器中拿Bean的时候才会去实例化；（它只去加载Bean的定义信息，显示调用getBean()才会真正去实例化），这样懒加载的优点是：启动快，启动时占用资源少。

ApplicationContext在启动的时候就把所有的Bean全部实例化了。它还可以为Bean配置`lazy-init=true`来让Bean延迟实例化（所有的单例、非懒加载的Bean都会容器启动时候立马实例化）；

立马加载好的优点有：

- 启动时都初始化完成了，所以运行时会更快。
- 我们能在系统启动的时候，尽早的发现系统中的配置问题 （因为启动时就得实例化处理）
- 可以（建议）把费时的操作放到系统启动中完成（比如初始化本地缓存、获取连接池的链接等等操作

# Spring Bean是单例模式吗？

不是，Spring中同一个类可以指定不同的名字，使用`@Bean`创建多次，并且都不是同一个对象。
# Spring循环依赖::
## 什么是Spring循环依赖问题？

当初始化A对象的时候，要给A填充属性，如果A对象有一个引用对象B，就回去容器中查找B对象，如果有B则直接赋值，如果没有B则会去创建B对象，如果B依赖了A对象或者B依赖的其他引用对象里间接引用了A对象，由于A对象没有初始化完成或者说不是完整对象，就会再去创建A对象，所以导致了循环依赖。  

其根本原因是因为创建的对象但没完成初始化的对象不会提前暴露出来。

## Spring如何解决循环依赖问题？::

Spring设计了三级缓存来解决循环依赖问题。在`DefaultSingletonBeanRegistry`类中用三个Map来表示三级缓存。
* 一级缓存：singletonObjects  
* 二级缓存：earlySingletonObjects  
* 三级缓存：singletonFactories  

假设场景：A依赖B，B依赖A。

循环依赖对象的赋值顺序 ：当A要赋值B时，B不存在，进而去创建B，创建B的时候A在三级缓存中，B就能拿到A，B成为完整对象会放到一级缓存,回到A赋值的地方就能从一级缓存拿到B对象。
## 为什么用三级缓存而不用二级缓存？::

> 可能就是设计如此，或许是为了三级缓存的职责划分更合理。

## 循环依赖与代理对象会产生什么问题？

场景设定为A依赖B，B依赖A的场景
### 普通对象与循环依赖
通过二级缓存 + 提前暴露的方式可以解决普通对象的循环依赖问题。

### 代理对象与循环依赖

如果A依赖的是B的代理对象，B依赖的是A的代理对象，会出现什么情况？
* 二级缓存的场景，由于提前暴露的是A原始对象（A先创建，二级缓存存放半成品A，然后去创建B，B依赖A，则填充A的原始对象，B完成实例化，创建B的代理对象，放进二级缓存，返回赋值给A），所以在属性填充的时候B赋值的是A的原始对象，但最终我们需要的是A的代理对象。
* 三级缓存如何解决？
	* 通过三级缓存存放FactoryBean，延迟对应创建
	* 当B需要A的时候，会提前创建A的代理对象，赋值给B
	* 创建完成B之后，回到创建A的流程
	* 属性装配是在A原始对象的基础上进行装配的（代理对象持有原始对象）

我们通过二级缓存 + 提前创建代理对象的方式是否可以实现？
好像不可以，如果提前创建代理对象，那么在属性注入的时候，操作的是代理对象，没法给代理对象赋值。我们只能将属性赋值给原始对象（代理对象持有原始对象），我们需要原始对象。

我们通过去掉一级缓存的方式，将原先从二级缓存转移到以及缓存的地方去掉，把完成品的对象放在二级缓存中，是否可以？
完全可以。相当于用二级缓存代替了一级缓存，不影响Bean创建的流程。

> 在代理对象的场景：原始对象好像只有属性装配的时候有用，最后缓存放的还是代理对象。
## 不能解决的循环依赖
* 原型模式
* 构造器注入方式
	* 对象都创建不出来
# 对AOP的理解
* 面向切面编程
* 需要动态代理
	* JDK
	* CGLIB
> 本质是通过创建代理对象将一些公共逻辑作为类的增强处理，这些公共逻辑叫切面，这面有前置，后置，环绕等等，都是基于动态代理的。
# Spring AOP 原理
* obtainFreshBeanFactory
	* loadBeanDefinitions：解析配置文件的`aop`相关标签
* registerBeanPostProcessors
	* AspectJAwareAdvisorAutoProxyCreator：配置文件相关基础类
	* AnnotationAwareAspectJAutoProxyCreator：注解相关
* finishBeanFactoryInitialization
	* getBean
		* resolveBeforeInstantiation
			* AOP在这个地方提前创建代理相关对象
				* `AspectJPointcutAdvisor#0`
					* `AspectJAroundAdvice`/`AspectJAfterAdvice`/`AspectJAfterReturningAdvice`/`AspectJAfterThrowingAdvice`/`AspectJMethodBeforeAdvice`
						* `MethodLocatingFactoryBean`
						* `AspectJExpressionPointcut`
						* `SimpleBeanFactoryAwareAspectInstanceFactory`
			* 同样会解析`@Aspect`注解修饰的类
		* 执行`BeanPostProcessor#postProcessAfterInitialization`
			* `wrapIfNecessary`
			* 如果目标对象需要被代理，则匹配相应的advisor，创建代理对象
> 创建代理对象的两个位置：
> 1. 如果循环依赖需要赋值代理对象，导致代理对象需要提前创建，getSingleton#singletonFactory.getObject()#getEarlyBeanReference
> 2. BeanPostProcessor#postProcessAfterInitialization根据必要性创建，如果代理对象没有提前创建，此处可能创建，AbstractAutoProxyCreator#postProcessAfterInitialization#wrapIfNecessary，没有循环依赖的时候会在这创建，spring开发者应该更倾向于在这创建，而不是提前创建

# Spring事务失效的场景

* 方法内自调用
	* this对象不是代理对象
* 方法是private的
	* spring事务会基于cglib来进行aop，cglib是通过重写子类的方式生成代理对象，由于方法是private，导致子类无法重写方法
* 方法是final
	* 同private
* 单独的线程调用方法
	* spring会从ThreadLocal中获取连接对象，如果新开启了线程，由于ThreadLocal的隔离，会导致拿不到连接，进而新创建一个来连接，这个新连接是自动提交的模式，无法回滚
* 异常被catch
* 类没有被spring管理
* 数据库不支持事务
# Spring中支持的常用数据库事务传播属性和事务隔离级别

## 事务的传播行为

> 当事务方法被另一个事务方法调用时，必须指定事务应该如何传播，列如方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行，事务传播的行为有传播属性指定，Spring定义了7中类传播行为

| 传播属性         | 描述                                                      |
| ------------ | ------------------------------------------------------- |
| REQUIRED     | **(默认)** 如果有事务在运行，当前的方法就在这个事务内运行，否则就启动一个新的事务，并在自己的事务内运行 |
| REQUIRED_NEW | **(常用)** 当前方法必须启动事务，并在它自己的事务内运行，如果有事务正在运行，应该将他挂起        |
| SUPPORTS     | 如果有事务在运行，当前的方法就在这个事务内运行，否则他可以不运行在事务中                    |
| NOT_SUPPORTE | 当前的方法不应该运行在事务中，如果有运行的事务，将他挂起                            |
| MANDATORY    | 当前的方法必须运行在事务内部，如果没有正在运行的事务，就抛出异常                        |
| NEVER        | 当前方法不应该运行在事务中，如果有运行的事务，就抛出异常                            |
| NESTED       | 如果有事务在运行，当前的方法就应该在这个事物的嵌套事务内运行，否则，就启动一个新的事务，并在它自己的事务内运行 |
|              |                                                         |

**事务传播属性可以在@Transactional注解的propagation属性中定义**

## 事务隔离级别

### 数据库事务并发问题
​ 假设现在有两个事务：Transaction01和Transaction02并发执行。

**1) 脏读**
> ①Transaction01将某条记录的AGE值从20修改为30。
> 
> ②Transaction02读取了Transaction01更新后的值：30。
> 
> ③Transaction01回滚，AGE值恢复到了20。
> 
> ④Transaction02读取到的30就是一个无效的值。

**2) 不可重复读**

> ①Transaction01读取了AGE值为20。
>
> ②Transaction02将AGE值修改为30。
>
> ③Transaction01再次读取AGE值为30，和第一次读取不一致。

**3) 幻读**
> ①Transaction01读取了STUDENT表中的一部分数据。
> 
> ②Transaction02向STUDENT表中插入了新的行。
> 
>③Transaction01读取了STUDENT表时，多出了一些行。

### 隔离级别
> 数据库系统必须具有隔离并发运行各个事务的能力，使它们不会相互影响，避免各种并发问题。一个事务与其他事务隔离的程度称为隔离级别。SQL标准中规定了多种事务隔离级别，不同隔离级别对应不同的干扰程度，隔离级别越高，数据一致性就越好，但并发性越弱。

- **读未提交**：READ UNCOMMITTED
允许`Transaction01`读取`Transaction02`未提交的修改。
![img_5.png](img_5.png)
- **读已提交(Oracle默认隔离级别)**：READ COMMITTED
​ 要求Transaction01只能读取`Transaction02`已提交的修改。
![img_6.png](Image%2Fimg_6.png)
如上图所示，在事务进行读写的时候，加入了锁（读锁和写锁，锁的相容性矩阵），这样在`Transaction01`进行修改的时候，加入了写锁（排他锁）直到事务提交释放锁，这样就保证了`Transaction02`就无法读取该值，解决了脏读的问题。
- ![img_7.png](Image%2Fimg_7.png)
  但是读已提交仍然会存在不可重复读的问题，如上图所示。使得`Transaction02` 前后两次读取的同一变量的值不一样。那怎么办呢，就到我们不可重复读的情况。
- **可重复读(MySQL的默认事务隔离级别)**：REPEATABLE READ
​ 确保Transaction01可以多次从一个字段中读取到相同的值，即Transaction01执行期间禁止其它事务对这个字段进行更新。也就是简单点说，操作读完提交事务才释放锁，而不是一读完就释放锁。
  MySQL会利用MVCC机制让其读到更新之前的值，也就第一次读到的值。即使事务巍提交之前，有另外的事务将其值修改，也是读到旧值。
![img_8.png](Image%2Fimg_8.png)
- **可串行化**：SERIALIZABLE
​ 确保Transaction01可以多次从一个表中读取到相同的行，在Transaction01执行期间，禁止其它事务对这个表进行添加、更新、删除操作。可以避免任何并发问题，但性能十分低下。
如果不是可串行化的话，会出现读取存在的行是10行，但是事务没提交，这个时候锁住的是存在的行，这个时候插入行并提交了的话，就会产生后面读取的行数不对了。出现幻读了。
![img_9.png](Image%2Fimg_9.png)
解决方案就是使用可串行化，把整个表给锁住了。
![img_10.png](Image%2Fimg_10.png)

**m面试题测试**
```text
//1.请简单介绍Spring支持的常用数据库事务传播属性和事务隔离级别？

/**
 * 事务的属性：
 * 	1.★propagation：用来设置事务的传播行为
 * 		事务的传播行为：一个方法运行在了一个开启了事务的方法中时，当前方法是使用原来的事务还是开启一个新的事务
 * 		-Propagation.REQUIRED：默认值，使用原来的事务
 * 		-Propagation.REQUIRES_NEW：将原来的事务挂起，开启一个新的事务
 * 	2.★isolation：用来设置事务的隔离级别
 * 		-Isolation.REPEATABLE_READ：可重复读，MySQL默认的隔离级别
 * 		-Isolation.READ_COMMITTED：读已提交，Oracle默认的隔离级别，开发时通常使用的隔离级别
 */

```
# Spring有哪些重要的BeanFactoryPostProcessor？::
* `ConfigurationClassPostProcessor`
	* 是比较重要的`BeanFactoryPostProcessor`，它和SpringBoot的自动装配息息相关。他负责`Configuration`、`Import`、`ImportResource`、`Component`、`ComponentScan`、`Bean`等注解的处理，和SpringBoot的自动装配息息相关，此类还会对配置类进行代理操作，解决@Bean 的单例问题
# invokeBeanFactoryPostProcessors方法中BeanFactoryPostProcessor的执行顺序？::
* 大顺序上先处理`BeanDefinitionRegistryPostProcessor`，再处理`BeanFactoryPostProcessor`  
* 每种类型处理的小顺序上先处理实现`PriorityOrdered`的PostProcessor，再处理实现`Ordered`的PostProcessor，再处理没实现顺序接口的PostProcessor

# Spring有哪些重要的BeanPostProcessor？::
* `ApplicationContextAwareProcessor` ：执行一些Aware接口的回调
* `CommonAnnotationBeanPostProcessor`：`@PostConstruct`方法执行 
* `AbstractAutoProxyCreator`：AOP相关

# SpringBoot自动装配原理
* Spring: ConfigurationClassPostProcessor -> @Import 
	* 注入Import的类
* SpringBoot:  
	* @SpringBootApplication -> 
		* @EnableAutoConfiguration -> 
			* @Improt(**AutoConfigurationImportSelector.class**)
				* 读取spring.factories下的配置
* spring.factories -> EnableAutoConfiguration配置

# 如何优化SpringBoot的启动速度？

* 设置bean的懒加载
* 创建扫描索引`@Index`以及`spring-context-indexer`
* 减少`@ComponentScan`、`@SpringBootApplication`扫描类的范围，以及其他组件的扫描范围，比如swagger、feign等等。
* 关闭SpringBoot的JMX监控，`spirng.jmx.enabled=false`
* 设置JVM参数 `-noverify`，不对类进行验证
* AOP切面尽量不要使用注解，这会导致启动时扫描全部方法
* 关闭endpoint的一些监控功能
* 排除项目多余的依赖jar

以上方法可以提高20%-30%的启动速度。

* 升级JDK17
* 升级SpringBoot3，支持本地化特性，将程序编译成可执行镜像，显著提升启动速度

以上方法可以提高200%-300%的启动速度。

# SpringBoot读取配置的方式
* @Value
* @Configuration#prefix
* Environment#getProperty
* @PropertySource
* PropertySourcesPlaceholderConfigurer
* Pproperties
> 前三种较常用，后三种主要获取外部配置文件

# Springboot解决跨域问题

跨域问题是由于浏览器同源策略（协议、域名、端口不一致）导致的，只有前端会存在。

如何解决：
* JSONP
	* 前后端都要增加代码，耦合性高
* CORS
	* 低版本浏览器不支持
* Nginx
	* 配置转发
# Spring MVC的执行流程
1. 客户端发起请求：
>客户端通过浏览器等方式发送 HTTP 请求到 Spring MVC 应用程序。
2. 前端控制器（DispatcherServlet）处理请求：
>前端控制器是 Spring MVC 的核心组件，它接收所有的请求并将其分发给相应的处理器（Controller）进行处理。DispatcherServlet 根据配置文件中的 URL 映射规则，将请求转发给对应的处理器。
3. 处理器映射器（Handler Mapping）查找处理器：
>处理器映射器负责根据请求的 URL 信息，将请求映射到相应的处理器（Controller）。它根据配置文件中的映射规则，确定要调用的处理器(其中包含拦截器链（HandlerInterceptor）和目标handler)。
4. 处理器执行业务逻辑：
>处理器（Controller）接收请求并执行相应的业务逻辑。它可能会调用业务逻辑层（Service）来处理请求，并根据请求结果准备模型数据(HandlerInterceptor前置方法 -> handler -> HandlerInterceptor后置方法)。
5. 视图解析器（View Resolver）解析视图：
>视图解析器负责将逻辑视图名解析为实际的视图对象。它根据配置文件中的规则，将逻辑视图名解析为具体的视图对象，如 JSP、Thymeleaf 模板等。
6. 视图渲染：
>解析得到的视图对象负责渲染模型数据，生成最终的响应结果。它会将模型数据填充到视图模板中，并生成 HTML 或其他格式的响应内容。
7. 响应返回给客户端：
>最终生成的响应结果通过响应对象传递给前端控制器（DispatcherServlet），然后通过网络传输返回给客户端。

