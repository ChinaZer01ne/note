# 现有的日志框架

## 日志门面（接口层）

- JCL（Jakarta Commons Logging）
- slf4j（ Simple Logging Facade for Java）

## 日志实现

- JUL（java util logging）
- logback
- log4j
- log4j2

# 日志门面技术

## 什么是门面模式

借鉴JDBC的思想，为日志系统也提供一套门面，那么我们就可以面向这些接口规范来开发，避免了直接依赖具体的日志框架。这样我们的系统在日志中，就存在了日志的门面和日志的实现。

核心为**外部与一个子系统的通信必须通过一个统一的外观对象进行，使得子系统更易于使用**。其实就是一种面向接口编程的思想。

## 为什么使用门面模式

- 面向接口开发，不再依赖具体的实现类。减少代码耦合。依赖倒置原则
- 项目通过导入不同的日志实现类，可以灵活的切换日志实现框架
- 统一API，方便开发者学习和使用
- 统一配置便于项目日志的管理

# Slf4j

目前最常用的日志接口层。

![](sl4j.png)
## 入门

```xml
<!-- slf4j 日志门面 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.26</version>
</dependency>
<!-- slf4j 内置的简单实现-->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>1.7.21</version>
</dependency>
```

## 桥接旧的日志框架（Bridging）

通常，您依赖的某些组件依赖于SLF4J以外的日志记录API。您也可以假设这些组件在不久的将来不会切换到SLF4J。为了解决这种情况，SLF4J附带了几个桥接模块，这些模块将对log4j，JCL和java.util.logging API的调用重定向，就好像它们是对SLF4J API一样。

桥接解决的是项目中日志的遗留问题，当系统中存在之前的日志API，可以通过桥接转换到slf4j的实现

1. 先去除之前老的日志框架的依赖
2. 添加SLF4J提供的桥接组件
3. 为项目添加SLF4J的具体实现

![](slf4j桥接.png)

## 迁移的方式

如果我们要使用SLF4J的桥接器，替换原有的日志框架，那么我们需要做的第一件事情，就是删除掉原有项目中的日志框架的依赖。然后替换成SLF4J提供的桥接器。

```xml
<!-- log4j -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>log4j-over-slf4j</artifactId>
    <version>1.7.27</version>
</dependency>
<!-- jul -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jul-to-slf4j</artifactId>
    <version>1.7.27</version>
</dependency>
<!-- jcl -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
    <version>1.7.27</version>
</dependency>
```

## 注意问题

1. jcl-over-slf4j.jar和 slf4j-jcl.jar不能同时部署。前一个jar文件将导致JCL将日志系统的选择委托给SLF4J，后一个jar文件将导致SLF4J将日志系统的选择委托给JCL，从而导致无限循环。
2. log4j-over-slf4j.jar和slf4j-log4j12.jar不能同时出现
3. jul-to-slf4j.jar和slf4j-jdk14.jar不能同时出现
4. 所有的桥接都只对Logger日志记录器对象有效，如果程序中调用了内部的配置类或者是Appender,Filter等对象，将无法产生效果。

## SLF4J原理解析

1. SLF4J通过LoggerFactory加载日志具体的实现对象。
2. LoggerFactory在初始化的过程中，会通过performInitialization()方法绑定具体的日志实现。
3. 在绑定具体实现的时候，通过类加载器，加载`org/slf4j/impl/StaticLoggerBinder.class`
4. 所以，只要是一个日志实现框架，在org.slf4j.impl包中提供一个自己的StaticLoggerBinder类，在其中提供具体日志实现的LoggerFactory就可以被SLF4J所加载
# Logback

常用的日志实现层框架。

Logback是由log4j创始人设计的另一个开源日志组件，性能比log4j要好。

官方网站：[https://logback.qos.ch/index.html](https://logback.qos.ch/index.html)

Logback主要分为三个模块：

* `logback-core`：其它两个模块的基础模块
* `logback-classic`：它是log4j的一个改良版本，同时它完整实现了slf4j API
* `logback-access`：访问模块与Servlet容器集成提供通过Http来访问日志的功能

## logback入门

引入依赖

```xml
<!-- slf4j 日志门面 -->
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>1.7.26</version>
</dependency>

<!-- logback-classic -->
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.2.3</version>
  <scope>test</scope>
</dependency>
```

## logback配置

logback会依次读取以下类型配置文件：

- `logback.groovy`
- `logback-test.xml`
- `logback.xml` 
- 如果均不存在会采用默认配置

## logback组件之间的关系

* `Logger`：日志的记录器，把它关联到应用的对应的context上后，主要用于存放日志对象，也可以定义日志类型、级别。
* `Appender`：用于指定日志输出的目的地，目的地可以是控制台、文件、数据库等等。
* `Layout`：负责把事件转换成字符串，格式化的日志信息的输出。在logback中Layout对象被封装在encoder中。

## 输出到控制台

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!--
    配置集中管理属性
    我们可以直接改属性的 value 值
    格式：${name}
  -->

  <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度 %msg：日志消息，%n是换行符-->
  <property name="pattern" value="[%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} %c %M %L [%thread] %m%n"/>

  <!--
  日志输出格式：
    %-5level
    %d{yyyy-MM-dd HH:mm:ss.SSS}日期
    %c类的完整名称
    %M为method
    %L为行号
    %thread线程名称
    %m或者%msg为信息
    %n换行
    -->

  <!--Appender: 设置日志信息的去向,常用的有以下几个 
  ch.qos.logback.core.ConsoleAppender (控制台)
  ch.qos.logback.core.rolling.RollingFileAppender (文件大小到达指定尺 寸的时候产生一个新文件)
  ch.qos.logback.core.FileAppender (文件)
    -->

      <!--控制台日志输出的 appender-->
      <appender name="console" class="ch.qos.logback.core.ConsoleAppender">

        <!--控制输出流对象 默认 System.out 改为 System.err-->
        <target>System.out</target>

        <!--日志消息格式配置-->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
          <!-- 引入上文配置的日志格式 -->
          <pattern>${pattern}</pattern>
        </encoder>
      </appender>

      <!--
      用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。 
      <loger>仅有一个name属性，一个可选的level和一个可选的addtivity属性 name:用来指定受此logger约束的某一个包或者具体的某一个类。 
      level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF， 如果未设置此属性，那么当前logger将会继承上级的级别。
      additivity: 是否向上级loger传递打印信息。默认是true。 
      <logger>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个 logger
      --> 

      <!--
      也是<logger>元素，但是它是根logger。默认debug level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF， 
      <root>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个 logger。 
      -->

      <!--root logger 配置-->
      <!-- 打开所有的位置级别 -->
      <root level="ALL">、
        <!-- 将上文定义的console控制台Appender添加进根目录 -->
        <appender-ref ref="console"/>
      </root>

</configuration>
```

## FileAppender配置

```XML
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--
        配置集中管理属性
        我们可以直接改属性的 value 值
        格式：${name}
    -->
    <property name="pattern" value="[%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} %c %M %L [%thread] %m%n"/>
    <!--
    日志输出格式：
        %-5level
        %d{yyyy-MM-dd HH:mm:ss.SSS}日期
        %c类的完整名称
        %M为method
        %L为行号
        %thread线程名称
        %m或者%msg为信息
        %n换行
      -->
    <!--定义日志文件保存路径属性-->
    <property name="log_dir" value="src/main/java/com/leex/logs"/>

    <!--控制台日志输出的 appender-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <!--控制输出流对象 默认 System.out 改为 System.err-->
        <target>System.out</target>
        <!--日志消息格式配置-->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 引入上文配置的日志格式 -->
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!--日志文件输出的 appender-->
    <appender name="file" class="ch.qos.logback.core.FileAppender">
        <!--日志文件保存路径-->
        <file>${log_dir}/logback.log</file>
        <!--日志消息格式配置-->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!--日志拆分和归档压缩的 appender 对象-->
    <appender name="rollFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--日志文件保存路径-->
        <!--<file>${log_dir}/roll_logback.log</file>-->
        <!--日志消息格式配置-->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
        <!--指定拆分规则-->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--按照时间和压缩格式声明拆分的文件名-->
            <!-- %d{yyyy-MM-dd}表示按日拆分,-号分割 ；
                log%i 表示拆分的文件 .log1.gz   .log2.gz
                   .gz .zip表示压缩包-->
            <fileNamePattern>${log_dir}/rolling.%d{yyyy-MM-dd-hh-mm-ss}.log%i.zip</fileNamePattern>
            <!--按照文件大小拆分-->
            <maxFileSize>1MB</maxFileSize>
        </rollingPolicy>
    </appender>

    <!--root logger 配置-->
    <!-- 打开所有的位置级别 -->
    <root level="ALL">
        <!-- 将上文定义的console控制台Appender添加进根目录 -->
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
        <appender-ref ref="rollFile"/>
    </root>
</configuration>

```

## Filter和异步日志配置

过滤器：在`appender`标签中加入`filter`过滤器标签，appender使用过滤器

```xml
<!--控制台日志输出的 appender-->
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    .......
    <!--日志级别过滤器-->
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <!-- 配置过滤的日志级别 -->
        <level>ERROR</level>
        <!-- 大于配置的日志级别则放行 -->
        <onMatch>ACCEPT</onMatch>
        <!-- 小于配置的日志级别则放行 -->
        <onMismatch>DENY</onMismatch>
    </filter>
</appender>
```

异步日志：相当于在`appender`和`logger`之间加一层`appender`作为异步操作

```xml
<!--异步日志-->
<appender name="async" class="ch.qos.logback.classic.AsyncAppender">
    <!--指定某个具体的 appender-->
    <appender-ref ref="rollFile"/>
</appender>
```

## logback-access配置

`logback-access`模块与Servlet容器（如Tomcat和Jetty）集成，以提供HTTP访问日志功能。我们可以使用logback-access模块来替换tomcat的访问日志。

1. 将`logback-access.jar`与`logback-core.jar`复制到`$TOMCAT_HOME/lib/`目录下
2. 修改`$TOMCAT_HOME/conf/server.xml`中的Host元素中添加：

```xml
<Valve className="ch.qos.logback.access.tomcat.LogbackValve" />
```

3. logback默认会在`$TOMCAT_HOME/conf`下查找文件`logback-access.xml`

```XML
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- always a good activate OnConsoleStatusListener -->
  <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener"/>
  <property name="LOG_DIR" value="${catalina.base}/logs"/>
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/access.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
          <fileNamePattern>access.%d{yyyy-MM-dd}.log.zip</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <!-- 访问日志的格式 -->
            <pattern>combined</pattern>
        </encoder>
    </appender>
    <appender-ref ref="FILE"/>
</configuration>

```