# 什么是 Spring cloud

> 构建分布式系统复杂和容易出错。Spring Cloud 为最常见的分布式系统模式提供了一种简单且易于接受的编程模型，帮助开发人员构建有弹性的、可靠的、协调的应用程序。Spring Cloud 构建于 Spring Boot 之上，使得开发者很容易入手并快速应用于生产中。

我所理解的Spring Cloud就是微服务系统架构的一站式解决方案，在平时我们构建微服务的过程中需要做如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等操作，而 Spring Cloud 为我们提供了一套简易的编程模型，使我们能在 Spring Boot 的基础上轻松地实现微服务项目的构建。

# Spring Cloud 解决什么问题

Spring Cloud 规范及实现意图要解决的问题其实就是微服务架构实施过程中存在的⼀些问题，⽐如微服务架构中的服务注册发现问题、⽹络问题（⽐如熔断场景）、统⼀认证安全授权问题、负载均衡问题、链路追踪等问题。

# Spring Cloud 架构

如前所述， Spring Cloud是⼀个微服务相关规范，这个规范意图为搭建微服务架构提供⼀站式服务， 采⽤组件（框架）化机制定义⼀系列组件，各类组件针对性的处理微服务中的特定问题，这些组件共同来构成Spring Cloud微服务技术栈。

## Spring Cloud 核⼼组件

Spring Cloud ⽣态圈中的组件，按照发展可以分为第⼀代 Spring Cloud组件和第⼆代 Spring Cloud组件。

||||
|---|---|---|
|第⼀代 Spring Cloud（Netflix， SCN）|第⼆代 Spring Cloud（主要就是 Spring Cloud Alibaba， SCA）||
|注册中⼼|Netflix Eureka|阿⾥巴巴 Nacos|
|客户端负载均衡|Netflix Ribbon|阿⾥巴巴 Dubbo LB、 Spring Cloud Loadbalancer|
|熔断器|Netflix Hystrix|阿⾥巴巴 Sentinel|
|⽹关|Netflix Zuul：性能⼀般，未来将退出Spring Cloud ⽣态圈|官⽅ Spring Cloud Gateway|
|配置中⼼|官⽅ Spring Cloud Config|阿⾥巴巴 Nacos、携程 Apollo|
|服务调⽤|Netflix Feign|阿⾥巴巴 Dubbo RPC|
|消息驱动|官⽅ Spring Cloud Stream||
|链路追踪|官⽅ Spring Cloud Sleuth/Zipkin||
|||阿⾥巴巴 seata 分布式事务⽅案|

# Spring Cloud 与 Dubbo 对⽐

Dubbo是阿⾥巴巴公司开源的⼀个⾼性能优秀的服务框架，基于RPC调⽤，对于⽬前使⽤率较⾼的Spring Cloud Netflix来说，它是基于HTTP的，所以效率上没有Dubbo⾼，但问题在于Dubbo体系的组件不全，不能够提供⼀站式解决⽅案，⽐如服务注册与发现需要借助于Zookeeper等实现，⽽Spring Cloud Netflix则是真正的提供了⼀站式服务化解决⽅案，且有Spring⼤家族背景。前些年， Dubbo使⽤率⾼于SpringCloud，但⽬前Spring Cloud在服务化/微服务解决⽅案中已经有了⾮常好的发展趋势。

# Spring Cloud 与 Spring Boot 的关系

Spring Cloud 只是利⽤了Spring Boot 的特点，让我们能够快速的实现微服务组件开发，否则不使⽤Spring Boot的话，我们在使⽤Spring Cloud时，每⼀个组件的相关Jar包都需要我们⾃⼰导⼊配置以及需要开发⼈员考虑兼容性等各种情况。所以Spring Boot是我们快速把Spring Cloud微服务技术应⽤起来的⼀种⽅式。

[第⼀代 Spring Cloud 核⼼组件](https://www.wolai.com/7wfNdYwfVKFn7Tb9ePcxsH)

[第二代 Spring Cloud 核心组件 (SCA)](https://www.wolai.com/94k2HVB3ibturayzY2wWiA)