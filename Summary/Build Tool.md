# Maven

## setting.xml配置

### 配置优先级

- `~/.m2/setting.xml`
- `安装目录/conf/setting.xml`

### 具体配置

* `localRepository` : 定义jar包下载位置
* `mirros`：镜像
* ...

## pom.xml详解

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd ">
    <!-- 父项目的坐标。如果项目中没有规定某个元素的值，那么父项目中的对应值即为项目的默认值。坐标包括group ID，artifact ID和 version。 -->
    <parent>
          <!-- 被继承的父项目的全球唯一标识符 -->
        <groupId>xxx</groupId>
        <!-- 被继承的父项目的构件标识符 -->
        <artifactId>xxx</artifactId>
        <!-- 被继承的父项目的版本 -->
        <version>xxx</version>
        <!-- 父项目的pom.xml文件的相对路径。相对路径允许你选择一个不同的路径。默认值是../pom.xml。
             Maven首先在构建当前项目的地方寻找父项目的pom，
             其次在文件系统的这个位置（relativePath位置），
             然后在本地仓库，最后在远程仓库寻找父项目的pom。
              -->
        <relativePath>xxx</relativePath>
    </parent>
    <!-- 声明项目描述符遵循哪一个POM模型版本。模型本身的版本很少改变，虽然如此，但它仍然是必不可少的，
         这是为了当Maven引入了新的特性或者其他模型变更的时候，确保稳定性。 -->
    <modelVersion> 4.0.0 </modelVersion>
    <!-- 项目的全球唯一标识符，通常使用全限定的包名区分该项目和其他项目。并且构建时生成的路径也是由此生成， 
         如com.mycompany.app生成的相对路径为：/com/mycompany/app -->
    <groupId>xxx</groupId>
    <!-- 构件的标识符，它和group ID一起唯一标识一个构件。换句话说，你不能有两个不同的项目拥有同样的artifact ID
         和groupID；在某个特定的group ID下，artifact ID也必须是唯一的。构件是项目产生的或使用的一个东西，Maven
         为项目产生的构件包括：JARs，源码，二进制发布和WARs等。 -->
    <artifactId>xxx</artifactId>
    <!-- 项目产生的构件类型，例如jar、war、ear、pom。插件可以创建他们自己的构件类型，所以前面列的不是全部构件类型 -->
    <packaging> jar </packaging>
    <!-- 项目当前版本，格式为:主版本.次版本.增量版本-限定版本号 -->
    <version> 1.0-SNAPSHOT </version>
    <!-- 项目的名称, Maven产生的文档用 -->
    <name> xxx-maven </name>
    <!-- 项目主页的URL, Maven产生的文档用 -->
    <url> http://maven.apache.org </url>
    <!-- 项目的详细描述, Maven 产生的文档用。 当这个元素能够用HTML格式描述时（例如，CDATA中的文本会被解析器忽略，
         就可以包含HTML标签）， 不鼓励使用纯文本描述。如果你需要修改产生的web站点的索引页面，你应该修改你自己的
         索引页文件，而不是调整这里的文档。 -->
    <description> A maven project to study maven. </description>
    <!-- 描述了这个项目构建环境中的前提条件。 -->
    <prerequisites>
        <!-- 构建该项目或使用该插件所需要的Maven的最低版本 -->
        <maven></maven>
    </prerequisites>
    <!-- 项目的问题管理系统(Bugzilla, Jira, Scarab,或任何你喜欢的问题管理系统)的名称和URL，本例为 jira -->
    <issueManagement>
        <!-- 问题管理系统（例如jira）的名字， -->
        <system> jira </system>
        <!-- 该项目使用的问题管理系统的URL -->
        <url> http://jira.baidu.com/xxx </url>
    </issueManagement>
    <!-- 项目持续集成信息 -->
    <ciManagement>
        <!-- 持续集成系统的名字，例如continuum -->
        <system></system>
        <!-- 该项目使用的持续集成系统的URL（如果持续集成系统有web接口的话）。 -->
        <url></url>
        <!-- 构建完成时，需要通知的开发者/用户的配置项。包括被通知者信息和通知条件（错误，失败，成功，警告） -->
        <notifiers>
            <!-- 配置一种方式，当构建中断时，以该方式通知用户/开发者 -->
            <notifier>
                <!-- 传送通知的途径 -->
                <type></type>
                <!-- 发生错误时是否通知 -->
                <sendOnError></sendOnError>
                <!-- 构建失败时是否通知 -->
                <sendOnFailure></sendOnFailure>
                <!-- 构建成功时是否通知 -->
                <sendOnSuccess></sendOnSuccess>
                <!-- 发生警告时是否通知 -->
                <sendOnWarning></sendOnWarning>
                <!-- 不赞成使用。通知发送到哪里 --><address></address>
                <!-- 扩展配置项 -->
                <configuration></configuration>
            </notifier>
        </notifiers>
    </ciManagement>
    <!-- 项目创建年份，4位数字。当产生版权信息时需要使用这个值。 -->
    <inceptionYear />
    <!-- 项目相关邮件列表信息 -->
    <mailingLists>
        <!-- 该元素描述了项目相关的所有邮件列表。自动产生的网站引用这些信息。 -->
        <mailingList>
            <!-- 邮件的名称 -->
            <name> Demo </name>
            <!-- 发送邮件的地址或链接，如果是邮件地址，创建文档时，mailto: 链接会被自动创建 -->
            <post> xxx@126.com </post>
            <!-- 订阅邮件的地址或链接，如果是邮件地址，创建文档时，mailto: 链接会被自动创建 -->
            <subscribe> xxx@126.com </subscribe>
            <!-- 取消订阅邮件的地址或链接，如果是邮件地址，创建文档时，mailto: 链接会被自动创建 -->
            <unsubscribe> xxx@126.com </unsubscribe>
            <!-- 你可以浏览邮件信息的URL -->
            <archive> http:/hi.baidu.com/xxx/demo/dev/</archive>
        </mailingList>
    </mailingLists>
    <!-- 项目开发者列表 -->
    <developers>
        <!-- 某个项目开发者的信息 -->
        <developer>
            <!-- SCM里项目开发者的唯一标识符 -->
            <id> HELLO WORLD </id>
            <!-- 项目开发者的全名 -->
            <name> xxx </name>
            <!-- 项目开发者的email -->
            <email> xxx@126.com </email>
            <!-- 项目开发者的主页的URL -->
            <url></url>
            <!-- 项目开发者在项目中扮演的角色，角色元素描述了各种角色 -->
            <roles>
                <role> Project Manager </role>
                <role> Architect </role>
            </roles>
            <!-- 项目开发者所属组织 -->
            <organization> demo </organization>
            <!-- 项目开发者所属组织的URL -->
            <organizationUrl> http://hi.baidu.com/xxx </organizationUrl>
            <!-- 项目开发者属性，如即时消息如何处理等 -->
            <properties>
                <dept> No </dept>
            </properties>
            <!-- 项目开发者所在时区， -11到12范围内的整数。 -->
            <timezone> -5 </timezone>
        </developer>
    </developers>
    <!-- 项目的其他贡献者列表 -->
    <contributors>
        <!-- 项目的其他贡献者。参见developers/developer元素 -->
        <contributor>
            <!-- 项目贡献者的全名 -->
            <name></name>
            <!-- 项目贡献者的email -->
            <email></email>
            <!-- 项目贡献者的主页的URL -->
            <url></url>
            <!-- 项目贡献者所属组织 -->
            <organization></organization>
            <!-- 项目贡献者所属组织的URL -->
            <organizationUrl></organizationUrl>
            <!-- 项目贡献者在项目中扮演的角色，角色元素描述了各种角色 -->
            <roles>
                <role> Project Manager </role>
                <role> Architect </role>
            </roles>
            <!-- 项目贡献者所在时区， -11到12范围内的整数。 -->
            <timezone></timezone>
            <!-- 项目贡献者属性，如即时消息如何处理等 -->
            <properties>
                <dept> No </dept>
            </properties>
        </contributor>
    </contributors>
    <!-- 该元素描述了项目所有License列表。 应该只列出该项目的license列表，不要列出依赖项目的 license列表。
         如果列出多个license，用户可以选择它们中的一个而不是接受所有license。 -->
    <licenses>
        <!-- 描述了项目的license，用于生成项目的web站点的license页面，其他一些报表和validation也会用到该元素。 -->
        <license>
            <!-- license用于法律上的名称 -->
            <name> Apache 2 </name>
            <!-- 官方的license正文页面的URL -->
            <url> http://www.baidu.com/xxx/LICENSE-2.0.txt </url>
            <!-- 项目分发的主要方式： 
                    repo，可以从Maven库下载 
                    manual， 用户必须手动下载和安装依赖 -->
            <distribution> repo </distribution>
            <!-- 关于license的补充信息 -->
            <comments> A business-friendly OSS license  </comments>
        </license>
    </licenses>
    <!-- SCM(Source Control Management)标签允许你配置你的代码库，供Maven web站点和其它插件使用。 -->
    <scm>
        <!-- SCM的URL,该URL描述了版本库和如何连接到版本库。欲知详情，请看SCMs提供的URL格式和列表。该连接只读。 -->
        <connection>scm:svn:http://svn.baidu.com/xxx/maven/xxx/xxx-maven2-trunk(dao-trunk) </connection>
        <!-- 给开发者使用的，类似connection元素。即该连接不仅仅只读 -->
        <developerConnection> scm:svn:http://svn.baidu.com/xxx/maven/xxx/dao-trunk
        </developerConnection>
        <!-- 当前代码的标签，在开发阶段默认为HEAD -->
        <tag></tag>
        <!-- 指向项目的可浏览SCM库（例如ViewVC或者Fisheye）的URL。 -->
        <url> http://svn.baidu.com/xxx </url>
    </scm>
    <!-- 描述项目所属组织的各种属性。Maven产生的文档用 -->
    <organization>
        <!-- 组织的全名 -->
        <name> demo </name>
        <!-- 组织主页的URL -->
        <url> http://www.baidu.com/xxx </url>
    </organization>
    <!-- 构建项目需要的信息 -->
    <build>
        <!-- 该元素设置了项目源码目录，当构建项目的时候，构建系统会编译目录里的源码。该路径是相对
             于pom.xml的相对路径。 -->
        <sourceDirectory></sourceDirectory>
        <!-- 该元素设置了项目脚本源码目录，该目录和源码目录不同：绝大多数情况下，该目录下的内容会
             被拷贝到输出目录(因为脚本是被解释的，而不是被编译的)。 -->
        <scriptSourceDirectory></scriptSourceDirectory>
        <!-- 该元素设置了项目单元测试使用的源码目录，当测试项目的时候，构建系统会编译目录里的源码。
             该路径是相对于pom.xml的相对路径。 -->
        <testSourceDirectory></testSourceDirectory>
        <!-- 被编译过的应用程序class文件存放的目录。 -->
        <outputDirectory></outputDirectory>
        <!-- 被编译过的测试class文件存放的目录。 -->
        <testOutputDirectory></testOutputDirectory>
        <!-- 使用来自该项目的一系列构建扩展 -->
        <extensions>
            <!-- 描述使用到的构建扩展。 -->
            <extension>
                <!-- 构建扩展的groupId -->
                <groupId></groupId>
                <!-- 构建扩展的artifactId -->
                <artifactId></artifactId>
                <!-- 构建扩展的版本 -->
                <version></version>
            </extension>
        </extensions>
        <!-- 当项目没有规定目标（Maven2 叫做阶段）时的默认值 -->
        <defaultGoal></defaultGoal>
        <!-- 这个元素描述了项目相关的所有资源路径列表，例如和项目相关的属性文件，这些资源被包含在
             最终的打包文件里。 -->
        <resources>
            <!-- 这个元素描述了项目相关或测试相关的所有资源路径 -->
            <resource>
                <!-- 描述了资源的目标路径。该路径相对target/classes目录（例如${project.build.outputDirectory}）。
                     举个例子，如果你想资源在特定的包里(org.apache.maven.messages)，你就必须该元素设置为
                    org/apache/maven/messages。然而，如果你只是想把资源放到源码目录结构里，就不需要该配置。 -->
                <targetPath></targetPath>
                <!-- 是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，文件在filters元素
                     里列出。 -->
                <filtering></filtering>
                <!-- 描述存放资源的目录，该路径相对POM路径 -->
                <directory></directory>
                <!-- 包含的模式列表，例如**/*.xml. -->
                <includes>
                    <include></include>
                </includes>
                <!-- 排除的模式列表，例如**/*.xml -->
                <excludes>
                    <exclude></exclude>
                </excludes>
            </resource>
        </resources>
        <!-- 这个元素描述了单元测试相关的所有资源路径，例如和单元测试相关的属性文件。 -->
        <testResources>
            <!-- 这个元素描述了测试相关的所有资源路径，参见build/resources/resource元素的说明 -->
            <testResource>
                <!-- 描述了测试相关的资源的目标路径。该路径相对target/classes目录（例如${project.build.outputDirectory}）。
                     举个例子，如果你想资源在特定的包里(org.apache.maven.messages)，你就必须该元素设置为
                    org/apache/maven/messages。然而，如果你只是想把资源放到源码目录结构里，就不需要该配置。 -->
                <targetPath></targetPath>
                <!-- 是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，文件在filters元素
                     里列出。 -->
                <filtering></filtering>
                <!-- 描述存放测试相关的资源的目录，该路径相对POM路径 -->
                <directory></directory>
                <!-- 包含的模式列表，例如**/*.xml. -->
                <includes>
                    <include></include>
                </includes>
                <!-- 排除的模式列表，例如**/*.xml -->
                <excludes>
                    <exclude></exclude>
                </excludes>
            </testResource>
        </testResources>
        <!-- 构建产生的所有文件存放的目录 -->
        <directory></directory>
        <!-- 产生的构件的文件名，默认值是${artifactId}-${version}。 -->
        <finalName></finalName>
        <!-- 当filtering开关打开时，使用到的过滤器属性文件列表 -->
        <filters></filters>
        <!-- 子项目可以引用的默认插件信息。该插件配置项直到被引用时才会被解析或绑定到生命周期。给定插件的任何本
             地配置都会覆盖这里的配置 -->
        <pluginManagement>
            <!-- 使用的插件列表 。 -->
            <plugins>
                <!-- plugin元素包含描述插件所需要的信息。 -->
                <plugin>
                    <!-- 插件在仓库里的group ID -->
                    <groupId></groupId>
                    <!-- 插件在仓库里的artifact ID -->
                    <artifactId></artifactId>
                    <!-- 被使用的插件的版本（或版本范围） -->
                    <version></version>
                    <!-- 是否从该插件下载Maven扩展（例如打包和类型处理器），由于性能原因，只有在真需要下载时，该
                         元素才被设置成enabled。 -->
                    <extensions>true/false</extensions>
                    <!-- 在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。 -->
                    <executions>
                        <!-- execution元素包含了插件执行需要的信息 -->
                        <execution>
                            <!-- 执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标 -->
                            <id></id>
                            <!-- 绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段 -->
                            <phase></phase>
                            <!-- 配置的执行目标 -->
                            <goals></goals>
                            <!-- 配置是否被传播到子POM -->
                            <inherited>true/false</inherited>
                            <!-- 作为DOM对象的配置 -->
                            <configuration></configuration>
                        </execution>
                    </executions>
                    <!-- 项目引入插件所需要的额外依赖 -->
                    <dependencies>
                        <!-- 参见dependencies/dependency元素 -->
                        <dependency></dependency>
                    </dependencies>
                    <!-- 任何配置是否被传播到子项目 -->
                    <inherited>true/false</inherited>
                    <!-- 作为DOM对象的配置 -->
                    <configuration></configuration>
                </plugin>
            </plugins>
        </pluginManagement>
        <!-- 该项目使用的插件列表 。 -->
        <plugins>
            <!-- plugin元素包含描述插件所需要的信息。 -->
            <plugin>
                <!-- 插件在仓库里的group ID -->
                <groupId></groupId>
                <!-- 插件在仓库里的artifact ID -->
                <artifactId></artifactId>
                <!-- 被使用的插件的版本（或版本范围） -->
                <version></version>
                <!-- 是否从该插件下载Maven扩展（例如打包和类型处理器），由于性能原因，只有在真需要下载时，该
                     元素才被设置成enabled。 -->
                <extensions>true/false</extensions>
                <!-- 在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。 -->
                <executions>
                    <!-- execution元素包含了插件执行需要的信息 -->
                    <execution>
                        <!-- 执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标 -->
                        <id></id>
                        <!-- 绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段 -->
                        <phase></phase>
                        <!-- 配置的执行目标 -->
                        <goals></goals>
                        <!-- 配置是否被传播到子POM -->
                        <inherited>true/false</inherited>
                        <!-- 作为DOM对象的配置 -->
                        <configuration></configuration>
                    </execution>
                </executions>
                <!-- 项目引入插件所需要的额外依赖 -->
                <dependencies>
                    <!-- 参见dependencies/dependency元素 -->
                    <dependency></dependency>
                </dependencies>
                <!-- 任何配置是否被传播到子项目 -->
                <inherited>true/false</inherited>
                <!-- 作为DOM对象的配置 -->
                <configuration></configuration>
            </plugin>
        </plugins>
    </build>
    <!-- 在列的项目构建profile，如果被激活，会修改构建处理 -->
    <profiles>
        <!-- 根据环境参数或命令行参数激活某个构建处理 -->
        <profile>
            <!-- 构建配置的唯一标识符。即用于命令行激活，也用于在继承时合并具有相同标识符的profile。 -->
            <id></id>
            <!-- 自动触发profile的条件逻辑。Activation是profile的开启钥匙。profile的力量来自于它能够
                 在某些特定的环境中自动使用某些特定的值；这些环境通过activation元素指定。activation元
                 素并不是激活profile的唯一方式。 -->
            <activation>
                <!-- profile默认是否激活的标志 -->
                <activeByDefault>true/false</activeByDefault>
                <!-- 当匹配的jdk被检测到，profile被激活。例如，1.4激活JDK1.4，1.4.0_2，而!1.4激活所有版本
                     不是以1.4开头的JDK。 -->
                <jdk>jdk版本，如:1.7</jdk>
                <!-- 当匹配的操作系统属性被检测到，profile被激活。os元素可以定义一些操作系统相关的属性。 -->
                <os>
                    <!-- 激活profile的操作系统的名字 -->
                    <name> Windows XP </name>
                    <!-- 激活profile的操作系统所属家族(如 'windows') -->
                    <family> Windows </family>
                    <!-- 激活profile的操作系统体系结构 -->
                    <arch> x86 </arch>
                    <!-- 激活profile的操作系统版本 -->
                    <version> 5.1.2600 </version>
                </os>
                <!-- 如果Maven检测到某一个属性（其值可以在POM中通过${名称}引用），其拥有对应的名称和值，Profile
                     就会被激活。如果值字段是空的，那么存在属性名称字段就会激活profile，否则按区分大小写方式匹
                     配属性值字段 -->
                <property>
                    <!-- 激活profile的属性的名称 -->
                    <name> mavenVersion </name>
                    <!-- 激活profile的属性的值 -->
                    <value> 2.0.3 </value>
                </property>
                <!-- 提供一个文件名，通过检测该文件的存在或不存在来激活profile。missing检查文件是否存在，如果不存在则激活 
                     profile。另一方面，exists则会检查文件是否存在，如果存在则激活profile。 -->
                <file>
                    <!-- 如果指定的文件存在，则激活profile。 -->
                    <exists> /usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/
                    </exists>
                    <!-- 如果指定的文件不存在，则激活profile。 -->
                    <missing> /usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/
                    </missing>
                </file>
            </activation>
            <!-- 构建项目所需要的信息。参见build元素 -->
            <build>
                <defaultGoal />
                <resources>
                    <resource>
                        <targetPath></targetPath>
                        <filtering></filtering>
                        <directory></directory>
                        <includes>
                            <include></include>
                        </includes>
                        <excludes>
                            <exclude></exclude>
                        </excludes>
                    </resource>
                </resources>
                <testResources>
                    <testResource>
                        <targetPath></targetPath>
                        <filtering></filtering>
                        <directory></directory>
                        <includes>
                            <include></include>
                        </includes>
                        <excludes>
                            <exclude></exclude>
                        </excludes>
                    </testResource>
                </testResources>
                <directory></directory>
                <finalName></finalName>
                <filters></filters>
                <pluginManagement>
                    <plugins>
                        <!-- 参见build/pluginManagement/plugins/plugin元素 -->
                        <plugin>
                            <groupId></groupId>
                            <artifactId></artifactId>
                            <version></version>
                            <extensions>true/false</extensions>
                            <executions>
                                <execution>
                                    <id></id>
                                    <phase></phase>
                                    <goals></goals>
                                    <inherited>true/false</inherited>
                                    <configuration></configuration>
                                </execution>
                            </executions>
                            <dependencies>
                                <!-- 参见dependencies/dependency元素 -->
                                <dependency></dependency>
                            </dependencies>
                            <goals></goals>
                            <inherited>true/false</inherited>
                            <configuration></configuration>
                        </plugin>
                    </plugins>
                </pluginManagement>
                <plugins>
                    <!-- 参见build/pluginManagement/plugins/plugin元素 -->
                    <plugin>
                        <groupId></groupId>
                        <artifactId></artifactId>
                        <version></version>
                        <extensions>true/false</extensions>
                        <executions>
                            <execution>
                                <id></id>
                                <phase></phase>
                                <goals></goals>
                                <inherited>true/false</inherited>
                                <configuration></configuration>
                            </execution>
                        </executions>
                        <dependencies>
                            <!-- 参见dependencies/dependency元素 -->
                            <dependency></dependency>
                        </dependencies>
                        <goals></goals>
                        <inherited>true/false</inherited>
                        <configuration></configuration>
                    </plugin>
                </plugins>
            </build>
            <!-- 模块（有时称作子项目） 被构建成项目的一部分。列出的每个模块元素是指向该模块的目录的
                 相对路径 -->
            <modules>
                <!--子项目相对路径-->
                <module></module>
            </modules>
            <!-- 发现依赖和扩展的远程仓库列表。 -->
            <repositories>
                <!-- 参见repositories/repository元素 -->
                <repository>
                    <releases>
                        <enabled></enabled>
                        <updatePolicy></updatePolicy>
                        <checksumPolicy></checksumPolicy>
                    </releases>
                    <snapshots>
                        <enabled></enabled>
                        <updatePolicy></updatePolicy>
                        <checksumPolicy></checksumPolicy>
                    </snapshots>
                    <id></id>
                    <name></name>
                    <url></url>
                    <layout></layout>
                </repository>
            </repositories>
            <!-- 发现插件的远程仓库列表，这些插件用于构建和报表 -->
            <pluginRepositories>
                <!-- 包含需要连接到远程插件仓库的信息.参见repositories/repository元素 -->
                <pluginRepository>
                    <releases>
                        <enabled></enabled>
                        <updatePolicy></updatePolicy>
                        <checksumPolicy></checksumPolicy>
                    </releases>
                    <snapshots>
                        <enabled></enabled>
                        <updatePolicy></updatePolicy>
                        <checksumPolicy></checksumPolicy>
                    </snapshots>
                    <id></id>
                    <name></name>
                    <url></url>
                    <layout></layout>
                </pluginRepository>
            </pluginRepositories>
            <!-- 该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。它们自动从项目定义的
                 仓库中下载。要获取更多信息，请看项目依赖机制。 -->
            <dependencies>
                <!-- 参见dependencies/dependency元素 -->
                <dependency></dependency>
            </dependencies>
            <!-- 不赞成使用. 现在Maven忽略该元素. -->
            <reports></reports>
            <!-- 该元素包括使用报表插件产生报表的规范。当用户执行“mvn site”，这些报表就会运行。 在页面导航栏能看
                 到所有报表的链接。参见reporting元素 -->
            <reporting></reporting>
            <!-- 参见dependencyManagement元素 -->
            <dependencyManagement>
                <dependencies>
                    <!-- 参见dependencies/dependency元素 -->
                    <dependency></dependency>
                </dependencies>
            </dependencyManagement>
            <!-- 参见distributionManagement元素 -->
            <distributionManagement></distributionManagement>
            <!-- 参见properties元素 -->
            <properties /> </profile>
    </profiles>
    <!-- 模块（有时称作子项目） 被构建成项目的一部分。列出的每个模块元素是指向该模块的目录的相对路径 -->
    <modules>
        <!--子项目相对路径-->
        <module></module>
    </modules>
    <!-- 发现依赖和扩展的远程仓库列表。 -->
    <repositories>
        <!-- 包含需要连接到远程仓库的信息 -->
        <repository>
            <!-- 如何处理远程仓库里发布版本的下载 -->
            <releases>
                <!-- true或者false表示该仓库是否为下载某种类型构件（发布版，快照版）开启。 -->
                <enabled></enabled>
                <!-- 该元素指定更新发生的频率。Maven会比较本地POM和远程POM的时间戳。这里的选项是：always（一直），
                     daily（默认，每日），interval：X（这里X是以分钟为单位的时间间隔），或者never（从不）。 -->
                <updatePolicy></updatePolicy>
                <!-- 当Maven验证构件校验文件失败时该怎么做：ignore（忽略），fail（失败），或者warn（警告）。 -->
                <checksumPolicy></checksumPolicy>
            </releases>
            <!-- 如何处理远程仓库里快照版本的下载。有了releases和snapshots这两组配置，POM就可以在每个单独的仓库中，
                 为每种类型的构件采取不同的策略。例如，可能有人会决定只为开发目的开启对快照版本下载的支持。参见repositories/repository/releases元素 -->
            <snapshots>
                <enabled></enabled>
                <updatePolicy></updatePolicy>
                <checksumPolicy></checksumPolicy>
            </snapshots>
            <!-- 远程仓库唯一标识符。可以用来匹配在settings.xml文件里配置的远程仓库 -->
            <id> xxx-repository-proxy </id>
            <!-- 远程仓库名称 -->
            <name> xxx-repository-proxy </name>
            <!-- 远程仓库URL，按protocol://hostname/path形式 -->
            <url> http://192.168.1.169:9999/repository/
            </url>
            <!-- 用于定位和排序构件的仓库布局类型-可以是default（默认）或者legacy（遗留）。Maven 2为其仓库提供了一个默认
                 的布局；然而，Maven 1.x有一种不同的布局。我们可以使用该元素指定布局是default（默认）还是legacy（遗留）。 -->
            <layout> default </layout>
        </repository>
    </repositories>
    <!-- 发现插件的远程仓库列表，这些插件用于构建和报表 -->
    <pluginRepositories>
        <!-- 包含需要连接到远程插件仓库的信息.参见repositories/repository元素 -->
        <pluginRepository></pluginRepository>
    </pluginRepositories>
    <!-- 该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。它们自动从项目定义的仓库中下载。
         要获取更多信息，请看项目依赖机制。 -->
    <dependencies>
        <dependency>
            <!-- 依赖的group ID -->
            <groupId> org.apache.maven </groupId>
            <!-- 依赖的artifact ID -->
            <artifactId> maven-artifact </artifactId>
            <!-- 依赖的版本号。 在Maven 2里, 也可以配置成版本号的范围。 -->
            <version> 3.8.1 </version>
            <!-- 依赖类型，默认类型是jar。它通常表示依赖的文件的扩展名，但也有例外。一个类型可以被映射成另外一个扩展
                 名或分类器。类型经常和使用的打包方式对应，尽管这也有例外。一些类型的例子：jar，war，ejb-client和test-jar。
                 如果设置extensions为 true，就可以在plugin里定义新的类型。所以前面的类型的例子不完整。 -->
            <type> jar </type>
            <!-- 依赖的分类器。分类器可以区分属于同一个POM，但不同构建方式的构件。分类器名被附加到文件名的版本号后面。例如，
                 如果你想要构建两个单独的构件成JAR，一个使用Java 1.4编译器，另一个使用Java 6编译器，你就可以使用分类器来生
                 成两个单独的JAR构件。 -->
            <classifier></classifier>
            <!-- 依赖范围。在项目发布过程中，帮助决定哪些构件被包括进来。欲知详情请参考依赖机制。 
                - compile ：默认范围，用于编译 
                - provided：类似于编译，但支持你期待jdk或者容器提供，类似于classpath 
                - runtime: 在执行时需要使用 
                - test: 用于test任务时使用 
                - system: 需要外在提供相应的元素。通过systemPath来取得 
                - systemPath: 仅用于范围为system。提供相应的路径 
                - optional: 当项目自身被依赖时，标注依赖是否传递。用于连续依赖时使用 -->
            <scope> test </scope>
            <!-- 仅供system范围使用。注意，不鼓励使用这个元素，并且在新的版本中该元素可能被覆盖掉。该元素为依赖规定了文件
                 系统上的路径。需要绝对路径而不是相对路径。推荐使用属性匹配绝对路径，例如${java.home}。 -->
            <systemPath></systemPath>
            <!-- 当计算传递依赖时， 从依赖构件列表里，列出被排除的依赖构件集。即告诉maven你只依赖指定的项目，不依赖项目的
                 依赖。此元素主要用于解决版本冲突问题 -->
            <exclusions>
                <exclusion>
                    <artifactId> spring-core </artifactId>
                    <groupId> org.springframework </groupId>
                </exclusion>
            </exclusions>
            <!-- 可选依赖，如果你在项目B中把C依赖声明为可选，你就需要在依赖于B的项目（例如项目A）中显式的引用对C的依赖。
                 可选依赖阻断依赖的传递性。 -->
            <optional> true </optional>
        </dependency>
    </dependencies>
    <!-- 不赞成使用. 现在Maven忽略该元素. -->
    <reports></reports>
    <!-- 该元素描述使用报表插件产生报表的规范。当用户执行“mvn site”，这些报表就会运行。 在页面导航栏能看到所有报表的链接。 -->
    <reporting>
        <!-- true，则，网站不包括默认的报表。这包括“项目信息”菜单中的报表。 -->
        <excludeDefaults />
        <!-- 所有产生的报表存放到哪里。默认值是${project.build.directory}/site。 -->
        <outputDirectory />
        <!-- 使用的报表插件和他们的配置。 -->
        <plugins>
            <!-- plugin元素包含描述报表插件需要的信息 -->
            <plugin>
                <!-- 报表插件在仓库里的group ID -->
                <groupId></groupId>
                <!-- 报表插件在仓库里的artifact ID -->
                <artifactId></artifactId>
                <!-- 被使用的报表插件的版本（或版本范围） -->
                <version></version>
                <!-- 任何配置是否被传播到子项目 -->
                <inherited>true/false</inherited>
                <!-- 报表插件的配置 -->
                <configuration></configuration>
                <!-- 一组报表的多重规范，每个规范可能有不同的配置。一个规范（报表集）对应一个执行目标 。例如，
                     有1，2，3，4，5，6，7，8，9个报表。1，2，5构成A报表集，对应一个执行目标。2，5，8构成B报
                     表集，对应另一个执行目标 -->
                <reportSets>
                    <!-- 表示报表的一个集合，以及产生该集合的配置 -->
                    <reportSet>
                        <!-- 报表集合的唯一标识符，POM继承时用到 -->
                        <id></id>
                        <!-- 产生报表集合时，被使用的报表的配置 -->
                        <configuration></configuration>
                        <!-- 配置是否被继承到子POMs -->
                        <inherited>true/false</inherited>
                        <!-- 这个集合里使用到哪些报表 -->
                        <reports></reports>
                    </reportSet>
                </reportSets>
            </plugin>
        </plugins>
    </reporting>
    <!-- 继承自该项目的所有子项目的默认依赖信息。这部分的依赖信息不会被立即解析,而是当子项目声明一个依赖
        （必须描述group ID和artifact ID信息），如果group ID和artifact ID以外的一些信息没有描述，则通过
            group ID和artifact ID匹配到这里的依赖，并使用这里的依赖信息。 -->
    <dependencyManagement>
        <dependencies>
            <!-- 参见dependencies/dependency元素 -->
            <!-- scope: import
            它只使用在dependencyManagement中，我们知道maven和java只能单继承，作用是管理依赖包的版本，一般用来保持当前项目的所有依赖版本统一。
            例如：项目中有很多的子项目，并且都需要保持依赖版本的统一，以前的做法是创建一个父类来管理依赖的版本，所有的子类继承自父类，这样就会导致父项目的pom.xml非常大，而且子项目不能再继承其他项目。
            import为我们解决了这个问题，可以把dependencyManagement放到一个专门用来管理依赖版本的pom中，然后在需要用到该依赖配置的pom中使用scope import就可以引用配置。-->
            <dependency></dependency>
        </dependencies>
    </dependencyManagement>
    <!-- 项目分发信息，在执行mvn deploy后表示要发布的位置。有了这些信息就可以把网站部署到远程服务器或者
         把构件部署到远程仓库。 -->
    <distributionManagement>
        <!-- 部署项目产生的构件到远程仓库需要的信息 -->
        <repository>
            <!-- 是分配给快照一个唯一的版本号（由时间戳和构建流水号）？还是每次都使用相同的版本号？参见
                 repositories/repository元素 -->
            <uniqueVersion />
            <id> xxx-maven2 </id>
            <name> xxx maven2 </name>
            <url> file://${basedir}/target/deploy
            </url>
            <layout></layout>
        </repository>
        <!-- 构件的快照部署到哪里？如果没有配置该元素，默认部署到repository元素配置的仓库，参见
             distributionManagement/repository元素 -->
        <snapshotRepository>
            <uniqueVersion />
            <id> xxx-maven2 </id>
            <name> xxx-maven2 Snapshot Repository
            </name>
            <url> scp://svn.baidu.com/xxx:/usr/local/maven-snapshot
            </url>
            <layout></layout>
        </snapshotRepository>
        <!-- 部署项目的网站需要的信息 -->
        <site>
            <!-- 部署位置的唯一标识符，用来匹配站点和settings.xml文件里的配置 -->
            <id> xxx-site </id>
            <!-- 部署位置的名称 -->
            <name> business api website </name>
            <!-- 部署位置的URL，按protocol://hostname/path形式 -->
            <url> scp://svn.baidu.com/xxx:/var/www/localhost/xxx-web
            </url>
        </site>
        <!-- 项目下载页面的URL。如果没有该元素，用户应该参考主页。使用该元素的原因是：帮助定位
             那些不在仓库里的构件（由于license限制）。 -->
        <downloadUrl />
        <!-- 如果构件有了新的group ID和artifact ID（构件移到了新的位置），这里列出构件的重定位信息。 -->
        <relocation>
            <!-- 构件新的group ID -->
            <groupId></groupId>
            <!-- 构件新的artifact ID -->
            <artifactId></artifactId>
            <!-- 构件新的版本号 -->
            <version></version>
            <!-- 显示给用户的，关于移动的额外信息，例如原因。 -->
            <message></message>
        </relocation>
        <!-- 给出该构件在远程仓库的状态。不得在本地项目中设置该元素，因为这是工具自动更新的。有效的值
             有：none（默认），converted（仓库管理员从Maven 1 POM转换过来），partner（直接从伙伴Maven 
             2仓库同步过来），deployed（从Maven 2实例部署），verified（被核实时正确的和最终的）。 -->
        <status></status>
    </distributionManagement>
    <!-- 以值替代名称，Properties可以在整个POM中使用，也可以作为触发条件（见settings.xml配置文件里
         activation元素的说明）。格式是<name>value</name>。 -->
    <properties>
        <name>value</name>
    </properties>
</project>
```

### 其中的节点详情

#### 一般信息节点

##### issueManagement项目问题管理系统

```XML
<!-- 项目的问题管理系统(Bugzilla, Jira, Scarab,或任何你喜欢的问题管理系统)的名称和URL，本例为 jira -->
<issueManagement>
  <!-- 问题管理系统（例如jira）的名字， -->
  <system> jira </system>
  <!-- 该项目使用的问题管理系统的URL -->
  <url> [http://jira.baidu.com/xxx](http://jira.baidu.com/xxx) </url>
</issueManagement>

```

##### ciManagement项目持续集成信息

```xml
<!-- 项目持续集成信息 -->
  <ciManagement>
    <!-- 持续集成系统的名字，例如continuum -->
    <system></system>
    <!-- 该项目使用的持续集成系统的URL（如果持续集成系统有web接口的话）。 -->
    <url></url>
    <!-- 构建完成时，需要通知的开发者/用户的配置项。包括被通知者信息和通知条件（错误，失败，成功，警告） -->
    <notifiers>
      <!-- 配置一种方式，当构建中断时，以该方式通知用户/开发者 -->
      <notifier>
        <!-- 传送通知的途径 -->
        <type></type>
        <!-- 发生错误时是否通知 -->
        <sendOnError></sendOnError>
        <!-- 构建失败时是否通知 -->
        <sendOnFailure></sendOnFailure>
        <!-- 构建成功时是否通知 -->
        <sendOnSuccess></sendOnSuccess>
        <!-- 发生警告时是否通知 -->
        <sendOnWarning></sendOnWarning>
        <!-- 不赞成使用。通知发送到哪里 --><address></address>
        <!-- 扩展配置项 -->
        <configuration></configuration>
      </notifier>
    </notifiers>
  </ciManagement>
```

##### mailingLists项目相关邮件列表信息

```xml
<!-- 项目相关邮件列表信息 -->
  <mailingLists>
    <!-- 该元素描述了项目相关的所有邮件列表。自动产生的网站引用这些信息。 -->
    <mailingList>
      <!-- 邮件的名称 -->
      <name> Demo </name>
      <!-- 发送邮件的地址或链接，如果是邮件地址，创建文档时，mailto: 链接会被自动创建 -->
      <post> [xxx@126.com](mailto:xxx@126.com) </post>
      <!-- 订阅邮件的地址或链接，如果是邮件地址，创建文档时，mailto: 链接会被自动创建 -->
      <subscribe> [xxx@126.com](mailto:xxx@126.com) </subscribe>
      <!-- 取消订阅邮件的地址或链接，如果是邮件地址，创建文档时，mailto: 链接会被自动创建 -->
      <unsubscribe> [xxx@126.com](mailto:xxx@126.com) </unsubscribe>
      <!-- 你可以浏览邮件信息的URL -->
      <archive> http:/hi.baidu.com/xxx/demo/dev/
      </archive>
    </mailingList>
  </mailingLists>
```

##### developers项目开发者列表

```xml
<!-- 项目开发者列表 -->
  <developers>
    <!-- 某个项目开发者的信息 -->
    <developer>
      <!-- SCM里项目开发者的唯一标识符 -->
      <id> HELLO WORLD </id>
      <!-- 项目开发者的全名 -->
      <name> xxx </name>
      <!-- 项目开发者的email -->
      <email> [xxx@126.com](mailto:xxx@126.com) </email>
      <!-- 项目开发者的主页的URL -->
      <url></url>
      <!-- 项目开发者在项目中扮演的角色，角色元素描述了各种角色 -->
      <roles>
        <role> Project Manager </role>
        <role> Architect </role>
      </roles>
      <!-- 项目开发者所属组织 -->
      <organization> demo </organization>
      <!-- 项目开发者所属组织的URL -->
      <organizationUrl> [http://hi.baidu.com/xxx](http://hi.baidu.com/xxx) </organizationUrl>
      <!-- 项目开发者属性，如即时消息如何处理等 -->
      <properties>
        <dept> No </dept>
      </properties>
      <!-- 项目开发者所在时区， -11到12范围内的整数。 -->
      <timezone> -5 </timezone>
    </developer>
  </developers>
```

##### contributors项目的其他贡献者列表

```xml
<!-- 项目的其他贡献者列表 -->
  <contributors>
    <!-- 项目的其他贡献者。参见developers/developer元素 -->
    <contributor>
      <!-- 项目贡献者的全名 -->
      <name></name>
      <!-- 项目贡献者的email -->
      <email></email>
      <!-- 项目贡献者的主页的URL -->
      <url></url>
      <!-- 项目贡献者所属组织 -->
      <organization></organization>
      <!-- 项目贡献者所属组织的URL -->
      <organizationUrl></organizationUrl>
      <!-- 项目贡献者在项目中扮演的角色，角色元素描述了各种角色 -->
      <roles>
        <role> Project Manager </role>
        <role> Architect </role>
      </roles>
      <!-- 项目贡献者所在时区， -11到12范围内的整数。 -->
      <timezone></timezone>
      <!-- 项目贡献者属性，如即时消息如何处理等 -->
      <properties>
        <dept> No </dept>
      </properties>
    </contributor>
  </contributors>
```

##### licenses许可证列表

```xml
<!-- 该元素描述了项目所有License列表。 应该只列出该项目的license列表，不要列出依赖项目的 license列表。
  如果列出多个license，用户可以选择它们中的一个而不是接受所有license。 -->
  <licenses>
    <!-- 描述了项目的license，用于生成项目的web站点的license页面，其他一些报表和validation也会用到该元素。 -->
    <license>
      <!-- license用于法律上的名称 -->
      <name> Apache 2 </name>
      <!-- 官方的license正文页面的URL -->
      <url> [http://www.baidu.com/xxx/LICENSE-2.0.txt](http://www.baidu.com/xxx/LICENSE-2.0.txt)
      </url>
      <!-- 项目分发的主要方式： 
        repo，可以从Maven库下载 
        manual， 用户必须手动下载和安装依赖 -->
      <distribution> repo </distribution>
      <!-- 关于license的补充信息 -->
      <comments> A business-friendly OSS license
      </comments>
    </license>
  </licenses>
```

##### scm和organization

```xml
<!-- SCM(Source Control Management)标签允许你配置你的代码库，供Maven web站点和其它插件使用。 -->
  <scm>
    <!-- SCM的URL,该URL描述了版本库和如何连接到版本库。欲知详情，请看SCMs提供的URL格式和列表。该连接只读。 -->
    <connection> scm:svn:[http://svn.baidu.com/xxx/maven/xxx/xxx-maven2-trunk(dao-trunk)](http://svn.baidu.com/xxx/maven/xxx/xxx-maven2-trunk(dao-trunk))
    </connection>
    <!-- 给开发者使用的，类似connection元素。即该连接不仅仅只读 -->
    <developerConnection> scm:svn:[http://svn.baidu.com/xxx/maven/xxx/dao-trunk](http://svn.baidu.com/xxx/maven/xxx/dao-trunk)
    </developerConnection>
    <!-- 当前代码的标签，在开发阶段默认为HEAD -->
    <tag></tag>
    <!-- 指向项目的可浏览SCM库（例如ViewVC或者Fisheye）的URL。 -->
    <url> [http://svn.baidu.com/xxx](http://svn.baidu.com/xxx) </url>
  </scm>
  <!-- 描述项目所属组织的各种属性。Maven产生的文档用 -->
  <organization>
    <!-- 组织的全名 -->
    <name> demo </name>
    <!-- 组织主页的URL -->
    <url> [http://www.baidu.com/xxx](http://www.baidu.com/xxx) </url>
  </organization>
```

#### 重要的信息

##### build 构建项目需要的信息

```xml
<!-- 构建项目需要的信息 -->

  <build>

    <!-- 该元素设置了项目源码目录，当构建项目的时候，构建系统会编译目录里的源码。该路径是相对

      于pom.xml的相对路径。 -->

    <sourceDirectory></sourceDirectory>

    <!-- 该元素设置了项目脚本源码目录，该目录和源码目录不同：绝大多数情况下，该目录下的内容会

      被拷贝到输出目录(因为脚本是被解释的，而不是被编译的)。 -->

    <scriptSourceDirectory></scriptSourceDirectory>

    <!-- 该元素设置了项目单元测试使用的源码目录，当测试项目的时候，构建系统会编译目录里的源码。

      该路径是相对于pom.xml的相对路径。 -->

    <testSourceDirectory></testSourceDirectory>

    <!-- 被编译过的应用程序class文件存放的目录。 -->

    <outputDirectory></outputDirectory>

    <!-- 被编译过的测试class文件存放的目录。 -->

    <testOutputDirectory></testOutputDirectory>

    <!-- 使用来自该项目的一系列构建扩展 -->

    <extensions>

      <!-- 描述使用到的构建扩展。 -->

      <extension>

        <!-- 构建扩展的groupId -->

        <groupId></groupId>

        <!-- 构建扩展的artifactId -->

        <artifactId></artifactId>

        <!-- 构建扩展的版本 -->

        <version></version>

      </extension>

    </extensions>

    <!-- 当项目没有规定目标（Maven2 叫做阶段）时的默认值 -->

    <defaultGoal></defaultGoal>

    <!-- 这个元素描述了项目相关的所有资源路径列表，例如和项目相关的属性文件，这些资源被包含在

      最终的打包文件里。 -->

    <resources>

      <!-- 这个元素描述了项目相关或测试相关的所有资源路径 -->

      <resource>

        <!-- 描述了资源的目标路径。该路径相对target/classes目录（例如${project.build.outputDirectory}）。

          举个例子，如果你想资源在特定的包里(org.apache.maven.messages)，你就必须该元素设置为

          org/apache/maven/messages。然而，如果你只是想把资源放到源码目录结构里，就不需要该配置。 -->

        <targetPath></targetPath>

        <!-- 是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，文件在filters元素

          里列出。 -->

        <filtering></filtering>

        <!-- 描述存放资源的目录，该路径相对POM路径 -->

        <directory></directory>

        <!-- 包含的模式列表，例如**/*.xml. -->

        <includes>

          <include></include>

        </includes>

        <!-- 排除的模式列表，例如**/*.xml -->

        <excludes>

          <exclude></exclude>

        </excludes>

      </resource>

    </resources>

    <!-- 这个元素描述了单元测试相关的所有资源路径，例如和单元测试相关的属性文件。 -->

    <testResources>

      <!-- 这个元素描述了测试相关的所有资源路径，参见build/resources/resource元素的说明 -->

      <testResource>

        <!-- 描述了测试相关的资源的目标路径。该路径相对target/classes目录（例如${project.build.outputDirectory}）。

          举个例子，如果你想资源在特定的包里(org.apache.maven.messages)，你就必须该元素设置为

          org/apache/maven/messages。然而，如果你只是想把资源放到源码目录结构里，就不需要该配置。 -->

        <targetPath></targetPath>

        <!-- 是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，文件在filters元素

          里列出。 -->

        <filtering></filtering>

        <!-- 描述存放测试相关的资源的目录，该路径相对POM路径 -->

        <directory></directory>

        <!-- 包含的模式列表，例如**/*.xml. -->

        <includes>

          <include></include>

        </includes>

        <!-- 排除的模式列表，例如**/*.xml -->

        <excludes>

          <exclude></exclude>

        </excludes>

      </testResource>

    </testResources>

    <!-- 构建产生的所有文件存放的目录 -->

    <directory></directory>

    <!-- 产生的构件的文件名，默认值是${artifactId}-${version}。 -->

    <finalName></finalName>

    <!-- 当filtering开关打开时，使用到的过滤器属性文件列表 -->

    <filters></filters>

    <!-- 子项目可以引用的默认插件信息。该插件配置项直到被引用时才会被解析或绑定到生命周期。给定插件的任何本

      地配置都会覆盖这里的配置 -->

    <pluginManagement>

      <!-- 使用的插件列表 。 -->

      <plugins>

        <!-- plugin元素包含描述插件所需要的信息。 -->

        <plugin>

          <!-- 插件在仓库里的group ID -->

          <groupId></groupId>

          <!-- 插件在仓库里的artifact ID -->

          <artifactId></artifactId>

          <!-- 被使用的插件的版本（或版本范围） -->

          <version></version>

          <!-- 是否从该插件下载Maven扩展（例如打包和类型处理器），由于性能原因，只有在真需要下载时，该

            元素才被设置成enabled。 -->

          <extensions>true/false</extensions>

          <!-- 在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。 -->

          <executions>

            <!-- execution元素包含了插件执行需要的信息 -->

            <execution>

              <!-- 执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标 -->

              <id></id>

              <!-- 绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段 -->

              <phase></phase>

              <!-- 配置的执行目标 -->

              <goals></goals>

              <!-- 配置是否被传播到子POM -->

              <inherited>true/false</inherited>

              <!-- 作为DOM对象的配置 -->

              <configuration></configuration>

            </execution>

          </executions>

          <!-- 项目引入插件所需要的额外依赖 -->

          <dependencies>

            <!-- 参见dependencies/dependency元素 -->

            <dependency></dependency>

          </dependencies>

          <!-- 任何配置是否被传播到子项目 -->

          <inherited>true/false</inherited>

          <!-- 作为DOM对象的配置 -->

          <configuration></configuration>

        </plugin>

      </plugins>

    </pluginManagement>

    <!-- 该项目使用的插件列表 。 -->

    <plugins>

      <!-- plugin元素包含描述插件所需要的信息。 -->

      <plugin>

        <!-- 插件在仓库里的group ID -->

        <groupId></groupId>

        <!-- 插件在仓库里的artifact ID -->

        <artifactId></artifactId>

        <!-- 被使用的插件的版本（或版本范围） -->

        <version></version>

        <!-- 是否从该插件下载Maven扩展（例如打包和类型处理器），由于性能原因，只有在真需要下载时，该

          元素才被设置成enabled。 -->

        <extensions>true/false</extensions>

        <!-- 在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。 -->

        <executions>

          <!-- execution元素包含了插件执行需要的信息 -->

          <execution>

            <!-- 执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标 -->

            <id></id>

            <!-- 绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段 -->

            <phase></phase>

            <!-- 配置的执行目标 -->

            <goals></goals>

            <!-- 配置是否被传播到子POM -->

            <inherited>true/false</inherited>

            <!-- 作为DOM对象的配置 -->

            <configuration></configuration>

          </execution>

        </executions>

        <!-- 项目引入插件所需要的额外依赖 -->

        <dependencies>

          <!-- 参见dependencies/dependency元素 -->

          <dependency></dependency>

        </dependencies>

        <!-- 任何配置是否被传播到子项目 -->

        <inherited>true/false</inherited>

        <!-- 作为DOM对象的配置 -->

        <configuration></configuration>

      </plugin>

    </plugins>

  </build>
```

##### profiles 构建规则

个人认为可以看作一个pom中规则的集合，符合设定条件则激活，或者手动激活

```xml
<!-- 在列的项目构建profile，如果被激活，会修改构建处理 -->

  <profiles>

    <!-- 根据环境参数或命令行参数激活某个构建处理 -->

    <profile>

      <!-- 构建配置的唯一标识符。即用于命令行激活，也用于在继承时合并具有相同标识符的profile。 -->

      <id></id>

      <!-- 自动触发profile的条件逻辑。Activation是profile的开启钥匙。profile的力量来自于它能够

        在某些特定的环境中自动使用某些特定的值；这些环境通过activation元素指定。activation元

        素并不是激活profile的唯一方式。 -->

      <activation>

        <!-- profile默认是否激活的标志 -->

        <activeByDefault>true/false</activeByDefault>

        <!-- 当匹配的jdk被检测到，profile被激活。例如，1.4激活JDK1.4，1.4.0_2，而!1.4激活所有版本

          不是以1.4开头的JDK。 -->

        <jdk>jdk版本，如:1.7</jdk>

        <!-- 当匹配的操作系统属性被检测到，profile被激活。os元素可以定义一些操作系统相关的属性。 -->

        <os>

          <!-- 激活profile的操作系统的名字 -->

          <name> Windows XP </name>

          <!-- 激活profile的操作系统所属家族(如 'windows') -->

          <family> Windows </family>

          <!-- 激活profile的操作系统体系结构 -->

          <arch> x86 </arch>

          <!-- 激活profile的操作系统版本 -->

          <version> 5.1.2600 </version>

        </os>

        <!-- 如果Maven检测到某一个属性（其值可以在POM中通过${名称}引用），其拥有对应的名称和值，Profile

          就会被激活。如果值字段是空的，那么存在属性名称字段就会激活profile，否则按区分大小写方式匹

          配属性值字段 -->

        <property>

          <!-- 激活profile的属性的名称 -->

          <name> mavenVersion </name>

          <!-- 激活profile的属性的值 -->

          <value> 2.0.3 </value>

        </property>

        <!-- 提供一个文件名，通过检测该文件的存在或不存在来激活profile。missing检查文件是否存在，如果不存在则激活 

          profile。另一方面，exists则会检查文件是否存在，如果存在则激活profile。 -->

        <file>

          <!-- 如果指定的文件存在，则激活profile。 -->

          <exists> /usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/

          </exists>

          <!-- 如果指定的文件不存在，则激活profile。 -->

          <missing> /usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/

          </missing>

        </file>

      </activation>

      <!-- 构建项目所需要的信息。参见build元素 -->

      <build>

        ...

      </build>

      <!-- 模块（有时称作子项目） 被构建成项目的一部分。列出的每个模块元素是指向该模块的目录的

        相对路径 -->

      <modules>

        <!--子项目相对路径-->

        <module></module>

      </modules>

      <!-- 发现依赖和扩展的远程仓库列表。 -->

      <repositories>

        <!-- 参见repositories/repository元素 -->

      </repositories>

      <!-- 发现插件的远程仓库列表，这些插件用于构建和报表 -->

      <pluginRepositories>

        <!-- 包含需要连接到远程插件仓库的信息.参见repositories/repository元素 -->

      </pluginRepositories>

      <!-- 该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。它们自动从项目定义的仓库中下载。要获取更多信息，请看项目依赖机制。 -->

      <dependencies>

        <!-- 参见dependencies/dependency元素 -->

        <dependency></dependency>

      </dependencies>

      <!-- 不赞成使用. 现在Maven忽略该元素. -->

      <reports></reports>

      <!-- 该元素包括使用报表插件产生报表的规范。当用户执行“mvn site”，这些报表就会运行。 在页面导航栏能看

        到所有报表的链接。参见reporting元素 -->

      <reporting></reporting>

      <!-- 参见dependencyManagement元素 -->

      <dependencyManagement>

        <dependencies>

          <!-- 参见dependencies/dependency元素 -->

          <dependency></dependency>

        </dependencies>

      </dependencyManagement>

      <!-- 参见distributionManagement元素 -->

      <distributionManagement></distributionManagement>

      <!-- 参见properties元素 -->

      <properties /> </profile>

  </profiles>
```

##### modules模块：略

##### repositories 远程仓库设置

```xml
<!-- 发现依赖和扩展的远程仓库列表。 -->

  <repositories>

    <!-- 包含需要连接到远程仓库的信息 -->

    <repository>

      <!-- 如何处理远程仓库里发布版本的下载 -->

      <releases>

        <!-- true或者false表示该仓库是否为下载某种类型构件（发布版，快照版）开启。 -->

        <enabled></enabled>

        <!-- 该元素指定更新发生的频率。Maven会比较本地POM和远程POM的时间戳。这里的选项是：always（一直），

          daily（默认，每日），interval：X（这里X是以分钟为单位的时间间隔），或者never（从不）。 -->

        <updatePolicy></updatePolicy>

        <!-- 当Maven验证构件校验文件失败时该怎么做：ignore（忽略），fail（失败），或者warn（警告）。 -->

        <checksumPolicy></checksumPolicy>

      </releases>

      <!-- 如何处理远程仓库里快照版本的下载。有了releases和snapshots这两组配置，POM就可以在每个单独的仓库中，

        为每种类型的构件采取不同的策略。例如，可能有人会决定只为开发目的开启对快照版本下载的支持。参见repositories/repository/releases元素 -->

      <snapshots>

        <enabled></enabled>

        <updatePolicy></updatePolicy>

        <checksumPolicy></checksumPolicy>

      </snapshots>

      <!-- 远程仓库唯一标识符。可以用来匹配在settings.xml文件里配置的远程仓库 -->

      <id> xxx-repository-proxy </id>

      <!-- 远程仓库名称 -->

      <name> xxx-repository-proxy </name>

      <!-- 远程仓库URL，按protocol://hostname/path形式 -->

      <url> [http://192.168.1.169:9999/repository/](http://192.168.1.169:9999/repository/)

      </url>

      <!-- 用于定位和排序构件的仓库布局类型-可以是default（默认）或者legacy（遗留）。Maven 2为其仓库提供了一个默认

        的布局；然而，Maven 1.x有一种不同的布局。我们可以使用该元素指定布局是default（默认）还是legacy（遗留）。 -->

      <layout> default </layout>

    </repository>

  </repositories>
```

##### pluginRepositories插件仓库

```xml
<!-- 发现插件的远程仓库列表，这些插件用于构建和报表 -->

  <pluginRepositories>

    <!-- 包含需要连接到远程插件仓库的信息.参见repositories/repository元素 -->

    <pluginRepository></pluginRepository>

  </pluginRepositories>
```

##### dependencies依赖

```xml
<!-- 该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。它们自动从项目定义的仓库中下载。要获取更多信息，请看项目依赖机制。 -->

  <dependencies>

    <dependency>

      <!-- 依赖的group ID -->

      <groupId> org.apache.maven </groupId>

      <!-- 依赖的artifact ID -->

      <artifactId> maven-artifact </artifactId>

      <!-- 依赖的版本号。 在Maven 2里, 也可以配置成版本号的范围。 -->

      <version> 3.8.1 </version>

      <!-- 依赖类型，默认类型是jar。它通常表示依赖的文件的扩展名，但也有例外。一个类型可以被映射成另外一个扩展名或分类器。类型经常和使用的打包方式对应，尽管这也有例外。一些类型的例子：jar，war，ejb-client和test-jar。如果设置extensions为 true，就可以在plugin里定义新的类型。所以前面的类型的例子不完整。 -->

      <type> jar </type>

      <!-- 依赖的分类器。分类器可以区分属于同一个POM，但不同构建方式的构件。分类器名被附加到文件名的版本号后面。例如，如果你想要构建两个单独的构件成JAR，一个使用Java 1.4编译器，另一个使用Java 6编译器，你就可以使用分类器来生成两个单独的JAR构件。 -->

      <classifier></classifier>

      <!-- 依赖范围。在项目发布过程中，帮助决定哪些构件被包括进来。欲知详情请参考依赖机制。 

        - compile ：默认范围，用于编译 

        - provided：类似于编译，但支持你期待jdk或者容器提供，类似于classpath 

        - runtime: 在执行时需要使用 

        - test: 用于test任务时使用 

        - system: 需要外在提供相应的元素。通过systemPath来取得 

        - systemPath: 仅用于范围为system。提供相应的路径 

        - optional: 当项目自身被依赖时，标注依赖是否传递。用于连续依赖时使用 -->

      <scope> test </scope>

      <!-- 仅供system范围使用。注意，不鼓励使用这个元素，并且在新的版本中该元素可能被覆盖掉。该元素为依赖规定了文件系统上的路径。需要绝对路径而不是相对路径。推荐使用属性匹配绝对路径，例如${java.home}。 -->

      <systemPath></systemPath>

      <!-- 当计算传递依赖时， 从依赖构件列表里，列出被排除的依赖构件集。即告诉maven你只依赖指定的项目，不依赖项目的依赖。此元素主要用于解决版本冲突问题 -->

      <exclusions>

        <exclusion>

          <artifactId> spring-core </artifactId>

          <groupId> org.springframework </groupId>

        </exclusion>

      </exclusions>

      <!-- 可选依赖，如果你在项目B中把C依赖声明为可选，你就需要在依赖于B的项目（例如项目A）中显式的引用对C的依赖。 可选依赖阻断依赖的传递性。 -->

      <optional> true </optional>

    </dependency>

  </dependencies>
```

##### reporting 报表产生的规范

```xml
<!-- 该元素描述使用报表插件产生报表的规范。当用户执行“mvn site”，这些报表就会运行。 在页面导航栏能看到所有报表的链接。 -->

  <reporting>

    <!-- true，则，网站不包括默认的报表。这包括“项目信息”菜单中的报表。 -->

    <excludeDefaults />

    <!-- 所有产生的报表存放到哪里。默认值是${project.build.directory}/site。 -->

    <outputDirectory />

    <!-- 使用的报表插件和他们的配置。 -->

    <plugins>

      <!-- plugin元素包含描述报表插件需要的信息 -->

      <plugin>

        <!-- 报表插件在仓库里的group ID -->

        <groupId></groupId>

        <!-- 报表插件在仓库里的artifact ID -->

        <artifactId></artifactId>

        <!-- 被使用的报表插件的版本（或版本范围） -->

        <version></version>

        <!-- 任何配置是否被传播到子项目 -->

        <inherited>true/false</inherited>

        <!-- 报表插件的配置 -->

        <configuration></configuration>

        <!-- 一组报表的多重规范，每个规范可能有不同的配置。一个规范（报表集）对应一个执行目标 。例如，有1，2，3，4，5，6，7，8，9个报表。1，2，5构成A报表集，对应一个执行目标。2，5，8构成B报表集，对应另一个执行目标 -->

        <reportSets>

          <!-- 表示报表的一个集合，以及产生该集合的配置 -->

          <reportSet>

            <!-- 报表集合的唯一标识符，POM继承时用到 -->

            <id></id>

            <!-- 产生报表集合时，被使用的报表的配置 -->

            <configuration></configuration>

            <!-- 配置是否被继承到子POMs -->

            <inherited>true/false</inherited>

            <!-- 这个集合里使用到哪些报表 -->

            <reports></reports>

          </reportSet>

        </reportSets>

      </plugin>

    </plugins>

  </reporting>
```

##### dependencyManagement依赖管理

```xml
<!-- 继承自该项目的所有子项目的默认依赖信息。这部分的依赖信息不会被立即解析,而是当子项目声明一个依赖（必须描述group ID和artifact ID信息），如果group ID和artifact ID以外的一些信息没有描述，则通过group ID和artifact ID匹配到这里的依赖，并使用这里的依赖信息。 -->

  <dependencyManagement>

    <dependencies>

      <!-- 参见dependencies/dependency元素 -->

      <dependency></dependency>

    </dependencies>

  </dependencyManagement>
```

##### distributionManagement项目分发信息

```xml
<!-- 项目分发信息，在执行mvn deploy后表示要发布的位置。有了这些信息就可以把网站部署到远程服务器或者

  把构件部署到远程仓库。 -->

  <distributionManagement>

    <!-- 部署项目产生的构件到远程仓库需要的信息 -->

    <repository>

      <!-- 是分配给快照一个唯一的版本号（由时间戳和构建流水号）？还是每次都使用相同的版本号？参见

        repositories/repository元素 -->

      <uniqueVersion />

      <id> xxx-maven2 </id>

      <name> xxx maven2 </name>

      <url> file://${basedir}/target/deploy

      </url>

      <layout></layout>

    </repository>

    <!-- 构件的快照部署到哪里？如果没有配置该元素，默认部署到repository元素配置的仓库，参见

      distributionManagement/repository元素 -->

    <snapshotRepository>

      <uniqueVersion />

      <id> xxx-maven2 </id>

      <name> xxx-maven2 Snapshot Repository

      </name>

      <url> scp://svn.baidu.com/xxx:/usr/local/maven-snapshot

      </url>

      <layout></layout>

    </snapshotRepository>

    <!-- 部署项目的网站需要的信息 -->

    <site>

      <!-- 部署位置的唯一标识符，用来匹配站点和settings.xml文件里的配置 -->

      <id> xxx-site </id>

      <!-- 部署位置的名称 -->

      <name> business api website </name>

      <!-- 部署位置的URL，按protocol://hostname/path形式 -->

      <url> scp://svn.baidu.com/xxx:/var/www/localhost/xxx-web

      </url>

    </site>

    <!-- 项目下载页面的URL。如果没有该元素，用户应该参考主页。使用该元素的原因是：帮助定位

      那些不在仓库里的构件（由于license限制）。 -->

    <downloadUrl />

    <!-- 如果构件有了新的group ID和artifact ID（构件移到了新的位置），这里列出构件的重定位信息。 -->

    <relocation>

      <!-- 构件新的group ID -->

      <groupId></groupId>

      <!-- 构件新的artifact ID -->

      <artifactId></artifactId>

      <!-- 构件新的版本号 -->

      <version></version>

      <!-- 显示给用户的，关于移动的额外信息，例如原因。 -->

      <message></message>

    </relocation>

    <!-- 给出该构件在远程仓库的状态。不得在本地项目中设置该元素，因为这是工具自动更新的。有效的值

      有：none（默认），converted（仓库管理员从Maven 1 POM转换过来），partner（直接从伙伴Maven 

      2仓库同步过来），deployed（从Maven 2实例部署），verified（被核实时正确的和最终的）。 -->

    <status></status>

  </distributionManagement>
```

##### properties配置参数

```xml
<!-- 以值替代名称，Properties可以在整个POM中使用，也可以作为触发条件（见settings.xml配置文件里

  activation元素的说明）。格式是<name>value</name>。 -->

<properties>

  <name>value</name>

</propies>
```

1. groupId: 分组名
2. artifactId: 功能名
3. version: 版本
4. packaging: 打包方式：jar、war、pom、maven-plugin，默认jar
5. dependenceManagement：
    - 只能出现在父pom里面（虽然他也可以在子pom里面，但我们不这样写）
    - 统一版本号
    - 声明（子pom中用到再引用）
6. dependency：
    - Type：默认jar，即引入一个特定的jar包。那么为什么还会有type为pom呢?当我们需要引入很多jar包的时候会导致pom.xml过大，我们可以想到的一种解决方案是定义一个父项目，但是父项目只有一个，也有可能导致父项目的pom.xml文件过大。这个时候我们引进来一个type为pom，意味着我们可以将所有的jar包打包成一个pom，然后我们依赖了pom，即可以下载下来所有依赖的jar包
        
    - scope：解决了什么时候会用和会不会打包到项目中的问题
        
        a）compile：编译的时候会用，打包，例如spring-core
        
        b）test：测试的时候会用，不打包，例如junit
        
        c）provided：编译的时候会用，不打包，例如servlet
        
        d）runtime：运行时会用，打包。例如jdbc驱动实现
        
        e）system：本地jar，不在maven仓库里，配合 `systemPath`使用，指定jar包位置。例如阿里的鱼卡短信调用
        
        f）依赖传递
        
        ```
        mvn dependency:tree > d.txt 查看包依赖
        ```
        

|第二依赖范围👉<br />第一依赖范围👇|complie|test|provided|runtime|
|---|---|---|---|---|
|compile|compile|-|-|runtime|
|test|test|-|-|test|
|provided|provided|-|provided|provided|
|runtime|runtime|-|-|runtime|

## 生命周期 Lifecycle/Phases/Goals

A Build Lifecycle is Made Up of Phases

A Build Phases is Made Up of Plugin Goals

### clean：

```
pre-clean:

clean:

post-clean:
```

### default:

```
complie:

package:

install:

deploy:

...
```

### site:

```
pre-site；

site：

post-site:

site-deploy:
```

![Maven生命周期](https://secure2.wostatic.cn/static/bJJrWNtxRZYY8BzLnHiQ76/Maven%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

## 版本管理：

a）1.0-SNAPSHOT（约定由于配置，不稳定版）

b）刷新本地仓库

```
1）从repository删除

2）mvn clean package -U（强制拉一次）
```

c）主版本号.此版本号.增量版本号-<里程碑版本>

## 常用命令

* compile
* clean 删除 target/
* test 运行test case junit/testNG
* package 打包
* install 把项目 install 到 local repository
* deploy 把本地的jar发布到私服上面去

## 常用插件

i. [https://maven.apache.org/plugins/](https://maven.apache.org/plugins/)

ii. [http://www.mojohaus.org/plugins.html](http://www.mojohaus.org/plugins.html)

iii. findbugs 静态代码检查

iv. versions 统一升级版本号 1. mvn versions:set -DnewVersion=1.1

v. source 打包源代码

vi. assembly 打包zip、war

vii. tomcat7

## 自定义插件

[https://maven.apache.org/guides/plugin/guide-java-plugin-development.html](https://maven.apache.org/guides/plugin/guide-java-plugin-development.html)

a）新建Maven项目

b）打包方式：maven-plugin（ `<packaging>maven-plugin</packaging>`）

c）引入以下两个插件

```xml
<dependencies>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-plugin-api</artifactId>
            <version>3.5.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugin-tools</groupId>
            <artifactId>maven-plugin-annotations</artifactId>
            <version>3.5</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
```

```java
@Mojo(name = "lion-plugin", defaultPhase = LifecyclePhase.PACKAGE)
public class LionMojo extends AbstractMojo {

    public void execute() throws MojoExecutionException, MojoFailureException {
        System.out.println("lion plugin");
    }
}
```

```xml
<plugin>
    <groupId>com.zer01ne</groupId>
    <artifactId>lion-plugin</artifactId>
    <version>1.0-SNAPSHOT</version>
    <executions>
        <execution>
            <phase>clean</phase>  <!--clean阶段运行这个插件-->
            <goals>
                <goal>lion-plugin</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```xml
<plugin>
    <groupId>com.zer01ne</groupId>
    <artifactId>lion-plugin</artifactId>
    <version>1.0-SNAPSHOT</version>
    <configuration>  <!--这里就是使用插件时的传值操作-->
        <msg>This is massage!</msg>
        <options>  <!-- 需要与插件中定义的属性名一致-->
            <param>one</param>  <!-- 这个标签可以随便写，不一定时param-->
            <param>two</param>
            <param>three</param>
        </options>
    </configuration>
    <executions>
        <execution>
            <phase>clean</phase>
            <goals>
                <goal>lion-plugin</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

传值问题参见：

[https://maven.apache.org/guides/plugin/guide-java-plugin-development.html](https://maven.apache.org/guides/plugin/guide-java-plugin-development.html)

## profile多环境配置

使用场景,多环境的配置，dev/test/pro

例如工程目录：

![1528780694248](https://secure2.wostatic.cn/static/mDh4ZcJjZL3AuqgTtAtFCA/%E5%B7%A5%E7%A8%8B%E7%9B%AE%E5%BD%95.png)

```xml
<profiles>
    <!--dev环境-->
    <profile>
        <id>dev</id>
        <properties>
            <profiles.active>dev</profiles.active>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
     <!--test环境-->
    <profile>
        <id>test</id>
        <properties>
            <profiles.active>test</profiles.active>
        </properties>
    </profile>
     <!--pro环境-->
    <profile>
        <id>pro</id>
        <properties>
            <profiles.active>pro</profiles.active>
        </properties>
    </profile>
</profiles>
<build>
    <resources>
        <resource>
            <!--排除所有的配置文件-->
            <directory>${basedir}/src/main/resources</directory>
            <excludes>
                <exclude>conf/**</exclude>
            </excludes>
        </resource>
        <resource>
            <!--加载指定配置文件-->
            <directory>src/main/resources/conf/${profiles.active}</directory>
        </resource>
    </resources>
```

## 仓库

a) 下载

b) 安装 解压

c) 使用[http://books.sonatype.com/nexus-book/reference3/index.html](http://books.sonatype.com/nexus-book/reference3/index.html)

i. [http://192.168.1.6:8081/nexus](http://192.168.1.6:8081/nexus)

ii. admin/admin123

d) 发布

i. pom.xml 配置

```xml
<distributionManagement>
     <!--如果版本不是snapshot，走这个-->
    <repository>
        <id>nexus-releases</id>  <!--与setting中的仓库id一致-->
        <name>Nexus Releases Repository</name>
        <url>http://192.168.1.127:8081/repository/maven-releases/</url>    <!--仓库地址-->
    </repository>
    <!--如果版本是snapshot，走这个-->
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <name>Nexus Snapshot Repository</name>
        <url>http://192.168.1.127:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

```xml
<server>
      <id>nexus-releases</id>  <!--与pom中的id一致-->
      <username>admin</username>
      <password>admin123</password>
    </server>
     <server>
      <id>nexus-snapshots</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
  </servers>
```

e）从私服下载jar

第一种方式：配置mirror

第二种方式：setting文件的profile属性

## 13.archetype模板化

a） 生成一个archetype

```
i、生成archetype

`mvn archetype:create-from-project`

ii、进入以下目录

`cd /target/generated-sources/archetype`

iii、安装到本地仓库

`mvn install`

iiii、如果想要发布到私服

`mvn deploy`

注意如果想发布，那么当前文件夹的pom文件必须包含一下属性
```

```xml
<distributionManagement>
    <repository>
      <id>nexus-releases</id>
      <name>Nexus Releases Repository</name>
      <url>http://192.168.1.127:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
      <id>nexus-snapshots</id>
      <name>Nexus Snapshot Repository</name>
      <url>http://192.168.1.127:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
  </distributionManagement>
```

## 14.springboot与maven整合

当出现找不到类的问题时，需要将项目安装到本地仓库。

如果直接运行`mvn install`，maven会将springboot的一个启动jar文件 安装到本地仓库，这个jar是我们项目中不能用的。如果别的模块引用了该模块，会出现找不到类或者包的问题。

比如：dao模块依赖了pojo模块，如果将由springboot构建的pojo模块安装到本地仓库，当我们再安装dao的时候，就会出现找不到pojo的问题。

解决方法：

1、执行`mvn package`

2、进入target目录，里面有个original文件

3、执行`mvn install:install-file -Dfile=xxx.original -DgroupId="your groupId" -DartifactId="your artifactId" -Dversion="your version" -Dpackaging=jar`

## 15.遇到的问题

1、依赖jar包的时候没有触发传递依赖。

我对接阿里大于的时候，引入`com.aliyun:aliyun-java-sdk-core:3.7.1`的时候，应该会把他需要的包同时依赖过来，但是没有。

原因：本地仓库里的``com.aliyun:aliyun-java-sdk-core:3.7.1`没有把pom给下下来（阿里云的私服上没有给出pom），本地下下来的是lastupdated后缀的。

解决：从down下来的jar中把pom文件内容拷贝给了本地仓库的pom文件。

g）依赖仲裁

i）最短路径原则，例如jar版本依赖，会依赖版本树中最近的版本

ii）加载顺序原则，pom加载的顺序

iii）exclusion，解决jar包冲突，排除

d）添加 `@Mojo`注解，name表示goal名称，defaultPhase默认运行阶段。

继承 `AbstractMojo`类，实现 `execute`方法。

e）`mvn clean install`将插件安装到本地仓库就可以使用了。

f）插件只有挂载到某个Phase（阶段）或者直接双击插件才能运行。

g）插件类也可以进行传值

```java
@Parameter
 private String msg;  //传值
Parameter
 private List<String> options;//传值
 public void execute() throws MojoExecutionException, MojoFailureException {
     System.out.println("lion plugin " + msg);  //使用
     System.out.println("lion plugin " + options);  //使用
 }
```

可以使用`mvn intall -P “环境id”`

多环境配置还可以在 maven 的 setting.xml 文件中配置，应用场景：可以配置在家工作和在公司工作的两套环境，与第一种方式应用场景不太一样。

ii.setting.xml配置

b) 从生成的archetype创建新的项目 (命令的方式)

i、`mvn archetype:generate -DarchetypeCatalog=local`

ii、选择使用的archetype

iii、按步操作，略


# Git

### Git和Svn的区别

![](https://secure2.wostatic.cn/static/2yerFVaf9EuWBsKvKkmSC/Git%E5%92%8CSvn%E7%9A%84%E5%8C%BA%E5%88%AB.png?auth_key=1702732012-P8SZmwyCyz8bQCMP4VSEH-0-d5662d69abd46796ddb51fbfe84d19bd)

Svn是集中式的，存储的是变化。中心化的。

Git是分布式的，存储的是完整的文件，去中心化，和区块链的思想相似。

Git保证完整性。

### 集中式版本控制（svn）和分布式版本控制的区别

1、集中式版本控制:(svn是这种形式)

有一个包含所有版本文件的单个服务器和一个数字(版本号),众多客户端从这个server上去检出文件(只是文件,本地没有仓库的概念)。

但是集中式的版本控制，有个严重的缺陷。就是中央服务器的单点故障。如果服务宕机一个小时，在这期间，没有任何人可以在正在工作的版本上很好的合 作或者去保存某一个版本的改变。另外如果中央数据库的磁盘坏了，并且可能没有保存备份，那么将丢失所有的东西。你失去了绝对一切 - 除了单一的任何人的快照恰好有在本地计算机上项目的整个历史。当然本地的版本控制系统也有相同的问题。虽然，你能够把每个人的本地代码，进行合并得到一个相对完整的版本，但是当你把这个相对完整的版本重新部署到服务器的新仓库时，将会丢失所有的历史版本包括日志。

2、分布式版本控制：（git是这种形式，GIT跟SVN一样有自己的集中式版本库或服务器）

这是在分布式版本控制系统（DVCSs）步在DVCS（如GIT中），客户端不只是检查出文件的最新快照：他们完全镜像的存储库（本地有仓库，这就是分布式的意义）。因此，如果出现上述问题，任何客户机库的可复制备份到服务器，以恢复它。每一个克隆确实是所有数据的完整备份(除了没有push的代码，这个也是理所当然的)。

那么针对于本地版本控制系统，和集中式版本控制系统的最严重的缺陷，就被分布式版本控制系统解决了。

#### 区别

1、git是分布式的scm,svn是集中式的。(最核心)

2、git是每个历史版本都存储完整的文件,便于恢复,svn是存储差异文件,历史版本不可恢复。(核心)

3、git可离线完成大部分操作,svn则不能。

4、git有着更优雅的分支和合并实现。

5、git有着更强的撤销修改和修改历史版本的能力

6、git速度更快,效率更高。

基于以上区别,git有了很明显的优势,特别在于它具有的本地仓库。

## 概念

工作区：

暂存区：通过add命令，让文件进入暂存区

本地版本库：通过commit命令，文件进入本地版本库

远程版本库：通过push命令，文件进入远程版本库

**git文件状态：已修改，已暂存，已提交。**

## 用户设置

a）`git config --global user.username 'xxx'`

b）`git config --global user.email 'xxx'`

> `git config --systemgit config --local`

对于`user.name`和`user.email`来说，有3个地方可以设置,查找顺序最近原则3，2，1.

1、`/etc/gitconfig`（几乎不会使用），针对于操作系统，`git config --system`

2、`~/.gitconfig`（很常用），针对用用户，`git config --global`

3、`.git/config`文件中，针对于特定项目的，`git config --local`

## git常用命令

```PowerShell
# 初始化git仓库
git init
# 创建git裸库
git init --bare
# 文件进入stage
git add <file>
# 让文件从暂存区回到工作区
git reset HEAD <file>
# 让commit文件变成未追踪的状态
git rm --cached <file> 
# 删除文件并且将其放入暂存区
git rm <file>
# 修改的文件还原到未修改状态，内容找不回了。  丢弃相对于暂存区中，文件最后一次变更
# 如果文件已经放到暂存区中，该命令是无效的
git checkout -- <file>
# 进入本地仓库，-m指定提交信息
git commit
# 修正最近一条的提交信息
git commit amend -m "修正"
# 修改提交的用户
git commit amend -reset-author
# 放到暂存区，并且提交
git commit -am [提交信息]
# 到远端仓库
git push
# 查看当前仓库状态。会提示那些文件发生修改，哪些内容需要add&commit。
git status
# 日志
git log
# 前三条
git log -3
# 日志列表
git log pretty=oneline
# 图形化显示日志
git log --graph
# 其他形式显示日志
git log --graph --abbrev-commit
git log --graph --pretty=oneline --abbrev-commit
# 帮助文档
git config --help
git help config
# 配置别名，如 branch 用 br来替换，reset HEAD 用 unstage 替换，这些被写入到~/.gitconfig中
git config --global alias.br branch
git config --global alias.unstage 'reset HEAD'
# 用 ！表示一个外部命令
git config --global alias.ui '!gitk'
```

## .gitignore文件

```.properties
# 忽略所有以d结尾的文件
*.d
# 忽略当前文件的mydir文件下的所有文件
mydir/
# 忽略根目录下test.txt文件
/test.txt
# 忽略所有目录下test.txt文件，**代表所有层次
/**/test.txt
```

## 分支

### 常用命令

```PowerShell
# 查看分支，-r显示所有远程分支，-a显示所有本地分支和远程分支
git branch
# 创建分支 
git brach [分支名]
# 分支重命名
git branch -m [old branch name] [new branch name]
# 当前分支最新的提交信息
git branch -v
# 删除分支
git branch -d [分支名]
# 强制删除，没合并分支的情况也会被删除分支
git branch -D [分支名]
# 切换分支 
# 如果此时有未提交的修改，是无法切换分支的，这时候就可以用`git stash`进行暂存
git checkout [分支名]
# 创建新分支，并且切换到该分支
git checkout -b [分支名]
# 在最近的两个分支上进行来回切换
git checkout -
# 合并分支，比如想要把dev分支的修改合并到master中，就要在master分支上执行 git merge dev 命令
git merge [分支名]
# 合并分支，不使用fast-forward模式，这种合并方式会产生一次新的提交
git merge --no-ff [分支名]

# 其他切换分支命令
# 切换到指定id的版本上，此时的HEAD是游离状态的，如果在此分支做了修改，需要提交才能再切换分支
git checkout [commit id]
# 如果HEAD是游离状态，切换分支后，提交也是游离状态，可以使用下面来新建分支保存那个状态,commit id 代表你上次checkout切换的分支id
git branch <new-branch-name> [commit id]

```

### 概念原理

HEAD：指向当前分支

master：指向提交

1、如下图所示，版本的每一次提交（commit），git都将它们根据提交的时间点串联成一条线。刚开始是只有一条时间线，即master分支，HEAD指向的是当前分支的当前版本。

![img](https://secure2.wostatic.cn/static/cRh7dYiXm4GgrLZG1av3NU/HEAD%E5%92%8Cmaster.png?auth_key=1702732012-okcnzzz8USYyr5H85cWBz8-0-f6d2d1c27d52f9dd5f5193e58c605575)

2、当创建了新分支，比如dev分支（通过命令git branch dev完成），git新建一个指针dev，dev=master，dev指向master指向的版本，然后切换到dev分支（通过命令git checkout dev完成），把HEAD指针指向dev，如下图。

![img](https://secure2.wostatic.cn/static/jG3yiQ8dwLM2uMJPSYefUv/%E5%88%87%E6%8D%A2%E5%88%86%E6%94%AF.png?auth_key=1702732012-4vyrYHSvBugUmDZqjaqrqp-0-83257f69531ff1ed4c28b2e5e03cfba5)

3、在dev分支上编码开发时，都是在dev上进行指针移动，比如在dev分支上commit一次，dev指针往前移动一步，但是master指针没有变，如下：

![img](https://secure2.wostatic.cn/static/w1EswCYySobZoM1MM5zTWN/%E5%88%86%E6%94%AF%E4%BF%AE%E6%94%B9.png?auth_key=1702732012-fnNQHehz83RbVmZ8tX8mjR-0-8e01d773453565d1e0eab9f15d3008df)

4、当我们完成了dev分支上的工作，要进行分支合并，把dev分支的内容合并到master分支上（通过首先切换到master分支，git branch master，然后合并git merge dev命令完成）。其内部的原理，其实就是先把HEAD指针指向master，再把master指针指向现在的dev指针指向的内容。如下图。

![img](https://secure2.wostatic.cn/static/w4GhuE35Gno9hvRTKqhHDe/%E5%90%88%E5%B9%B6%E5%88%86%E6%94%AF.png?auth_key=1702732012-fzAgvjCPUcAUryPef7m2u7-0-a322cd20c0e95b02ffc575d00971aed2)

5、当合并分支的时候出现冲突（confict），比如在dev分支上commit了一个文件file1，同时在master分支上也提交了该文件file1，修改的地方不同（比如都修改了同一个语句），那么合并的时候就有可能出现冲突，如下图所示。

![img](https://secure2.wostatic.cn/static/mpq3GVAhjbm2JwJqY8ZaPf/%E5%88%86%E6%94%AF%E5%86%B2%E7%AA%81.png?auth_key=1702732012-vK6nSQj9fFygqeW7z2T1HX-0-ca3a3a368f892db9a0b9e03d89354929)

这时候执行git merge dev命令，git会默认执行合并，但是要手动解决下冲突，然后在master上git add并且git commit，现在git分支的结构如下图。

![img](https://secure2.wostatic.cn/static/8xE6CQCdzEXp8ZGXRoBwud/%E5%90%88%E5%B9%B6%E5%86%B2%E7%AA%81.png?auth_key=1702732012-L2t4ZNor4sFFJEdL1NG8U-0-a4a5d35e47a9dcac2b1daef4370d302e)

在master上合并成功后，切换到dev分支上，将master合并到dev上，会fast-forward，直接合并成功，因为master已经比dev快了一个版本了，上图绿色线条表示的地方。

6、合并完成后，就可以删除掉dev分支（通过git branch -d dev命令完成）。

![img](https://secure2.wostatic.cn/static/gE6XjBN6NJ7XN5Ra2khMD8/%E5%88%A0%E9%99%A4%E5%88%86%E6%94%AF.png?auth_key=1702732012-2Rv8nbS3Ef8GmdB7HdJSD8-0-0e912237ba5bb5797cae9b1ba75700f7)

[详细内容请看此博客](https://blog.csdn.net/zl1zl2zl3/article/details/52637737)

## 回退版本

### 常用命令

```PowerShell
# 回退到上个版本
git reset --hard HEAD^
# 回退到上上个版本（回退两个版本）
git reset --hard HEAD^^
# 如果回退了，又想回去，commit id不需要写全
git reset --hard [commit id]
# 如果你回退之后，又想回去，但不知道commit id了，就需要查看里是的操作日志信息
git reflog
# 回退到之前的第2个提交
git reset --hard HEAD~2
# 回退到指定的提交
git reset --hard [commit id]
```

## 现场保存

当在一个分支做了一些修改，提交后，切换到另一个分支做了一些修改，修改未完成，没有提交，此时想切换分支就无法切换了，这就需要下面的命令了。

### 常用命令

```PowerShell
# 将当前的工作情况保存
git stash
# 查看保存的现场列表 
git stash list
# 恢复之前的保存的状态，并把该条状态信息删除
git stash pop
# 恢复之前的保存的状态，不删除该条状态信息，你可能需要手动删除git stash drop stash@{0}
git stash apply
# 恢复指定的保存状态
git stash apply stash@{1}
# 删除某条状态信息
git stash drop stash@{0}
```

## 标签

标签有两种：轻量级标签、带有附注的标签

轻量级标签是保存了当前提交sha-1值。（一层指向）

带有附注的标签是一个对象，有自己的sha-1值，然后指向当前提交的sha-1值（两层）

### 常用命令

```PowerShell
# 创建轻量级标签
git tag [标签名]
# 创建带有附注的标签
git tag -a [标签名] -m [附注]
# 查看存在的标签
git tag
# 查找标签，支持*通配符
git tag -l [关键词]
# 删除标签
git tag -d [标签名]
# 将标签推送到远程
git push origin v1.0
# 删除远程标签
git push origin --delete tag [标签名]
git push origin :refs/tags/[标签名]
```

## 文件比较

**主要是三个区域的相互比较：工作区、暂存区、版本库。**

```PowerShell
# 比较暂存区与工作区文件之间的差别
git diff
# 工作区和最新的提交的差别
git diff HEAD
# 最新的提交和暂存区的差别
git diff --cached
# 和指定commit id的差别
git diff [commit id]
git diff --cached [commit id]
```

## 远程

### 概念

`origin/master`：远程分支，这个分支对应了远程仓库的`master`分支。

### 常用命令

```PowerShell
# 从远程仓库克隆，
git clone [地址] 
git clone [地址] [自定义仓库名称]
# 拉取，pull = fetch + merge
git pull
# 推送
git push
## 完整写法
git push origin src:dest
# 查看远程仓库的信息
git remote
# 列出所有的远程仓库的别名（有可能你本地仓库关联了Github、Gitlib多个远程仓库）
git remote show
# 显示远程仓库详情
git remote show [远程仓库别名]
# 查看所有分支
git branch -a
# 查看远程分支
git branch -r
# 查看所有分支及最后一次提交
git branch -av
# 关联远程仓库
git remote add origin https://github.com/ChinaZer01ne/utils.git
# 将本地的分支和远程分支进行关联了
git branch --set-upstream-to=origin/<branch>
# 合并两个独立的仓库
git pull origin master --allow-unrelated-histories
# 把本地分支推送到远程，当远程没有对应分支的时候，执行
git push --set-upstream origin [分支名]
# 当本地没有与远程对应的分支，如果向追踪远程分支，执行 如：git checkout -b dev origin/dev
git checkout -b [分支名] [远程分支名]
## 也可以用这个，只是默认名对应了远程分支
git checkout --track [远程分支名]
# 删除远程分支
git push origin :[要删除的目标分支]
git push origin --delete [要删除的目标分支]
# 恢复删除的远程分支，如果忽略两个参数，就是把当前分支推送到远程，是同名的
git push --set-upstream origin [源分支]:[目标分支]
## 创建同名的
git push origin [目标分支]
## 创建不同名的
git push origin HEAD:[新分支名]
# 如果想重命名远程分支，只能先删掉远程分支，然后重新推送本地分支
略
# 如果远程分支被删掉了，可能你需要在本地把远程分支的引用删除，可以执行
git remote prune origin
# 查看远程分支日志
git log origin/[远程分支名]
git log remotes/origin/[远程分支名]
git log refs/remotes/origin/[远程分支名]
# 如果你想创建一个远程分支的引用，如果没有本地分支对应，那么他是一个stale状态
git fetch origin [远程分支名]:refs/remotes/origin/[自定义引用名]
# 如果是本地没有分支和本地创建的远程分支引用对应那么，可以执行
git checkout --track origin/[你的远程分支的引用名]
## 或者（全名）
git checkout --track refs/remotes/origin/[你的远程分支的引用名]
```

### 如何将项目分享到Github上

1、如果Github上已经新建了一个空的仓库，那么我们只需要将本地仓库和远程仓库关联起来就可以了。

```PowerShell
# 关联远程仓库
git remote add origin [仓库地址]
# 将本地的master与远程做关联
# 新版本不推荐
git push -u origin [分支名]
# 新版本推荐
git push --set-upstream origin [分支名]
```

如果远程和本地仓库是两个独立的项目（仓库名不一致）的话，尝试这两个命令。

```PowerShell
# 将本地的分支和远程分支进行关联了
git branch --set-upstream-to=origin/<branch>
# 合并两个独立的仓库
git pull origin master -–allow-unrelated-histories
```

如果你想使用`SSH`的形式的仓库地址，就要`在仓库setting的Deploy keys`先配置`SSH密钥`

```PowerShell
# 执行该命令生成公钥私钥，默认在~/.ssh文件夹下
ssh-keygen
```

将生成的公钥配置到Github上（`~/.ssh/id_rsa`）,然后执行关联仓库的操作（回到步骤1）

## submodule

场景：

我们有一个A项目（git版本管理），A项目依赖B模块的代码，B模块是单独的git仓库管理的。我们需要将B模块的git仓库变成A模块的仓库的一个子模块，形成一种父子关系。

```PowerShell
# 将子模块拉取到当前面模块的xxx文件（这个文件夹不要自己创建，git会自动生成）中
# 如 git submodule add https://github.com/ChinaZer01ne/git_child.git submodule
git submodule add [子模块的地址] [当前模块的xx文件夹,可选参数]
# 拉取更新，你可以进入单独的子模块执行，git pull操作，也可以
## 在根目录就可以拉取所有的更新，包括子模块的更新
git submodule foreach git pull
# 但我们克隆一个依赖submodule的git项目时，克隆下来，submodule模块里面内容是空的，我们需要进入子模块进行初始化和更新操作。
git submodule init
git submodule update --recursive
## 另一种方法一条命令解决
git clone https://github.com/ChinaZer01ne/git_parent.git --recursive
# 删除submodule，没有提供直接的命令，不过我们可以使用删除然后提交的方式
略
```

弊端：

因为父模块包含了子模块，所以父模块就可以修改子模块，然后推送，这就会出现一些问题。因此就出现了`git subtree`命令，来替代`submodule`，这也是git官方推荐使用的。

## subtree

```PowerShell
# 在父模块中关联子模块， 如git remote add subtree-origin  https://github.com/ChinaZer01ne/git_subtree_child.git
git remote add [别名] [地址] 
# 添加子模块 git subtree add --prefix=subtree subtree-origin master --squash，--squash参数表示是否将子模块的提交合并成一次提交，这个参数如果使用那么以后pull的时候一定要都加上，如果没有使用，那么以后都不要使用，否则会出现一些莫名其妙的问题
git subtree add --prefix=[拉取到当前模块的位置] [拉取的地址别名] [拉取的分支] 
# 在父模块中拉取，如git subtree pull --prefix=subtree subtree-origin master
git subtree pull --prefix=[拉取到当前模块的位置] [拉取的地址别名] [拉取的分支] 
# 如果在父模块修改了子模块的数据，可以通过git push推到父模块，通过如下命令，推送到子模块git subtree push --prefix=subtree subtree-origin master
git subtree push --prefix=[push的文件] [push的地址别名] [push的分支] 
```

## cherry-pick

场景：

项目开发中有很多分支，开发的时候发现，自己应该在dev分支上开发，但不小心到了master做了文件的提交，如何把自己在master上的提交应用到dev上？

```PowerShell
# commit id 是你想应用的提交
# 把master的提交应用到dev上，就在dev分支上执行git cherry-pick [commit id]，这个commit id是master上的你想修改的提交
# commit id 最好是从你想修改的第一次提交开始，慢慢追加，如果你直接选择最后的提交，那么必须经历个merge的过程，并且一些日志
git cherry-pick [commit id]
# 将master分支的文件提交应用到dev上之后，你可能想让master分支恢复之前的状态，那么先切换到之前状态的master分支上（游离态），commit id表示之前状态的分支
git checkout [commit id]
# 然后删除master分支
git branch -D [分支名]
# 然后从游离态新建切换到master分支
git checkout -b [分支名]
# 然后重新关联master分支
git branch --set-upstream-to=[远程分支引用]
```

## rebase

不要在与其他人共享的分支上进行操作。不要对master分支执行rebase，否则会引起很多问题。

一般来说，执行rebase的分支都是自己的本地分支，没有推送到远程版本库。

```PowerShell
# 如果想把test上的修改变到dev上，那么就在test分支执行，git rebase dev，相当于把test新修改的基础变为dev
git rebase [要变到的分支名]
# 取消变基操作
git rebase --abort
# 可以忽略掉当前补丁（从最早的提交开始）
git rebase --skip
# 如果有冲突，解决然后添加，然后执行下面命令，表示继续rebase操作
git rebase --continue
```

## 其他

1、git的提交id（commit id）是一个摘要值，是通过sha1计算出来的。

2、如果新创建一个文件夹mydir，如果mydir中没有文件，git是不识别的。

3、每个提交都有自己的parent指针，指向上一次提交的commit id。

4、`git checkout -- <file>` ：丢弃相对于暂存区中，文件最后一次变更

`git reset HEAD <file>` ：让文件从暂存区回到工作区。

如果文件已经放到暂存区中。那么`git checkout -- <file>` 是无效的。

5、使用`git blame <file>`命令查看文件的修改历史。

6、 `git remote show [仓库别名]`显示仓库详情

`Fetch URL`：版本拉取的地址。

`Push URL`：版本推送的地址。

7、`git push`完成了两件事情：①推送到远程②改变`origin/master`和当前分支版本一致

8、git是不会让我们直接修改`origin/master`的，当我们使用`git checkout origin/master` ，会切换到`origin/master`对应的最新的提交上，这个HEAD游离状态的。

9、`fast-forward`：快进，说明没有冲突。

10、`vi`删除的命令`dd`删除一行，`2,4d`：表示删除第二行到第四行。

变基操作会生成一条路径，merge会表现为多条路径。


# Jenkins