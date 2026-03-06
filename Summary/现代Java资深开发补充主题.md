# 现代Java资深开发补充主题（2026）

## 当前评估（基于现有笔记）

- 结论：**基础能力已达资深面试主线要求**，但在“测试体系、安全合规、工程化交付、API治理”四块存在明显短板。
- 覆盖情况（主观评估）：
  - 语言与并发、JVM、数据库、中间件、分布式主线：`较完整`
  - 微服务治理与架构设计：`较完整`
  - 线上排障与稳定性：`中等偏上`
  - 测试工程化、安全合规、发布治理：`偏弱（已新增补漏专题）`
  - AI 工程化落地：`偏弱（已新增 [[AI工程化与Java落地]] 专题）`
- 建议：按 `[[复习分层导航]]` + `[[30天复习计划]]` 做复习；按 `[[新增模块学习计划]]` 学习未掌握模块，再并入复习体系。

## 目标

在你现有的 Java/Spring/中间件基础上，补齐“现代资深工程师”能力：  
不仅会写业务代码，还能做版本演进、稳定性治理、安全合规、交付效率和成本平衡。

## P0（优先补齐）

### 1) 平台版本与迁移策略（Java + Spring）
- 关键点
  - Java LTS 版本策略（17/21/25）与升级节奏。
  - Spring Boot 3/4、Spring Framework 6/7 的兼容关系与迁移边界。
  - 从 `javax.*` 到 `jakarta.*` 的影响面（依赖、注解、容器）。
- 面试输出
  - 你如何做“跨大版本升级”的灰度方案（双环境、分批、回滚、兼容性测试）。

### 2) Java 并发新模型（Loom）
- 关键点
  - Virtual Threads：适合 I/O 密集型线程/请求模型。
  - Structured Concurrency：并发任务生命周期管理与失败传播。
  - Scoped Values：替代部分 ThreadLocal 场景的上下文传递。
- 面试输出
  - 什么时候适合虚拟线程，什么时候仍然使用线程池。

### 3) Spring 现代化微服务栈
- 关键点
  - `OpenFeign + Spring Cloud LoadBalancer`（替代旧 Ribbon 思路）。
  - `Spring Cloud CircuitBreaker + Resilience4j`（熔断/限流/隔离）。
  - `Spring Security OAuth2 Resource Server + JWT`（统一鉴权）。
  - `Micrometer Observation + OpenTelemetry`（指标/链路统一模型）。
- 面试输出
  - 你如何从 Hystrix/Ribbon 旧栈迁到当前主流方案。

### 4) 云原生运行与弹性治理（Kubernetes）
- 关键点
  - `liveness/readiness/startup` 探针的差异与配置原则。
  - HPA 的指标驱动扩缩容与防抖配置。
  - 发布策略（滚动/灰度）与容量评估。
- 面试输出
  - 如何避免“错误探针导致雪崩重启”。

### 5) 可观测性与 SRE 基线
- 关键点
  - Logs/Metrics/Traces 三支柱落地。
  - 统一 TraceId/SpanId 传播。
  - SLI/SLO/Error Budget 与发布策略联动。
- 面试输出
  - 你如何定义一个核心接口的 SLO，并据此决策是否继续发布。

### 6) 分布式数据一致性进阶
- 关键点
  - Outbox + CDC（Debezium）替代“双写”。
  - 幂等键、去重表、重试补偿、死信处理。
  - 最终一致性链路的可观测性（消息延迟、堆积、失败率）。
- 面试输出
  - 订单/支付类场景如何保证“业务一致 + 可恢复”。

### 7) 安全与供应链安全
- 关键点
  - API 安全（OWASP API Top 10）。
  - 身份认证、授权边界、最小权限。
  - SBOM（CycloneDX）与依赖漏洞治理。
- 面试输出
  - 你如何把安全从“上线前扫描”前移到“研发流程内”。

## P1（加分项）

### 8) 性能工程体系
- 关键点
  - JMH 做微基准，避免“拍脑袋优化”。
  - JFR/Arthas/火焰图联合定位 CPU、锁竞争、内存热点。
  - 压测数据与容量模型联动（峰值、余量、降级阈值）。

### 9) 模块化架构治理
- 关键点
  - Spring Modulith 做单体内模块边界治理。
  - 通过架构测试（如 ArchUnit）防止层间越权依赖。

### 10) 大规模代码演进自动化
- 关键点
  - OpenRewrite 做批量 API 迁移、技术债收敛、规范统一。

### 11) AI工程化落地（新时代必学）
- 关键点
  - LLM 应用开发（Prompt、Tool Calling、结构化输出）。
  - Java 接入（Spring AI/LangChain4j）与治理（超时、限流、审计）。
  - RAG、评估、幻觉治理与安全防护。
- 参考
  - `[[AI工程化与Java落地]]`

## P2（按业务需要补充）

### 12) AOT/Native Image 与启动优化
- 关键点
  - Spring AOT、GraalVM Native Image 适用场景与限制。
  - JVM 模式与 Native 模式的取舍（启动、内存、构建时长、兼容性）。

### 13) 研发效能指标化
- 关键点
  - DORA 指标体系（部署频率、变更前置时间、变更失败率、恢复时间等）。
  - 指标用于识别瓶颈，而不是只做管理报表。

## 建议同步补到现有笔记

- `[[Java]]`：补“Java LTS 策略 + Loom 实战边界”。
- `[[Spring]]`：补“Security/OAuth2/JWT + Observability”。
- `[[MicroService Design]]`：补“Outbox + CDC + 幂等补偿”。
- `[[稳定性治理]]`：补“SLO/Error Budget + 发布门禁”。
- `[[压测]]`：补“压测指标口径 + 容量模型模板”。

## 官方参考（主）

- Java LTS 路线图：<https://www.oracle.com/java/technologies/java-se-support-roadmap.html>
- Virtual Threads（JEP 444）：<https://openjdk.org/jeps/444>
- Structured Concurrency（JEP 453）：<https://openjdk.org/jeps/453>
- Spring Boot 系统要求：<https://docs.spring.io/spring-boot/system-requirements.html>
- Spring Cloud OpenFeign：<https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html>
- Spring Cloud Circuit Breaker：<https://docs.spring.io/spring-cloud-circuitbreaker/reference/index.html>
- Spring Security JWT 资源服务：<https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html>
- Spring Boot Observability：<https://docs.spring.io/spring-boot/reference/actuator/observability.html>
- OpenTelemetry Java Agent：<https://opentelemetry.io/docs/zero-code/java/agent/>
- Kubernetes Probes：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- Kubernetes HPA：<https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/>
- Debezium：<https://debezium.io/documentation/reference/stable/index.html>
- OWASP API Top 10（2023）：<https://owasp.org/API-Security/editions/2023/en/0x11-t10/>
- CycloneDX（SBOM）：<https://cyclonedx.org/>
- JMH：<https://openjdk.org/projects/code-tools/jmh/>
- DORA 指标演进：<https://dora.dev/guides/dora-metrics/history/>
