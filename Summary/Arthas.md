## 概述

### Arthas（阿尔萨斯） 能为你做什么？

Arthas 是Alibaba开源的Java诊断工具，深受开发者喜爱。当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：
- 1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
- 2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
- 3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
- 4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
- 5. 是否有一个全局视角来查看系统的运行状况？
- 6. 有什么办法可以监控到JVM的实时运行状态？
- 7. 怎么快速定位应用的热点，生成火焰图？
## [安装与卸载]()
### 安装

- 下载arthas-boot.jar，然后用java -jar的方式启动：
	```sh
	curl -O https://alibaba.github.io/arthas/arthas-boot.jar
	java -jar arthas-boot.jar
	```

- 如果下载速度比较慢，可以使用aliyun的镜像：
    ```sh
    java -jar arthas-boot.jar --repo-mirror aliyun --use-http
    ```
### 卸载

```sh
rm -rf ~/.arthas/
rm -rf ~/logs/arthas
```

  
## 快速入门

- 1. 执行一个jar包
- 2. 通过arthas来粘附，并且进行几种常用的操作
- 3. 通过一个案例快速入门

### 1. 准备代码

> 以下是一个简单的Java程序，每隔一秒生成一个随机数，再执行质因数分解，并打印出分解结果。代码 的内容不用理会这不是现在关注的点。

```java
public class MathGame {
    private static Random random = new Random();
    //用于统计生成的不合法变量的个数
    public int illegalArgumentCount = 0;

    public static void main(String[] args) throws InterruptedException {
        MathGame game = new MathGame();
        //死循环，每过 1 秒调用 1 次下面的方法(不是开启一个线程)
        while (true) {
            game.run();
            TimeUnit.SECONDS.sleep(1);
        }
    }

    //分解质因数
    public void run() throws InterruptedException {
        try {
            //随机生成 1 万以内的整数
            int number = random.nextInt() / 10000;
            //调用方法进行质因数分解
            List<Integer> primeFactors = primeFactors(number);
            //打印结果
            print(number, primeFactors);
        } catch (Exception e) {
            System.out.println(String.format("illegalArgumentCount:%3d, ",
                    illegalArgumentCount) + e.getMessage());
        }
    }

    //打印质因数分解的结果
    public static void print(int number, List<Integer> primeFactors) {
        StringBuffer sb = new StringBuffer(number + "=");
        for (int factor : primeFactors) {
            sb.append(factor).append('*');
        }
        if (sb.charAt(sb.length() - 1) == '*') {
            sb.deleteCharAt(sb.length() - 1);
        }
        System.out.println(sb);
    }

    //计算number的质因数分解
    public List<Integer> primeFactors(int number) {
        //如果小于 2 ，则抛出异常，并且计数加 1
        if (number < 2) {
            illegalArgumentCount++;
            throw new IllegalArgumentException("number is: " + number + ", need >= 2");
        }
        //用于保存每个质数
        List<Integer> result = new ArrayList<Integer>();
        //分解过程，从 2 开始看能不能整除
        int i = 2;
        while (i <= number) { //如果i大于number就退出循环
            //能整除，则i为一个因数，number为整除的结果再继续从 2 开始
            if (number % i == 0) {
                result.add(i);
                number = number / i;
                i = 2;
            } else {
                i++; //否则i++
            }
        }
        return result;
    }
}
```

### 2. 启动 Demo

```shell
curl -O https://arthas.aliyun.com/math-game.jar

java -jar math-game.jar
```
### 3. 启动 arthas

在命令行下面执行（使用和目标进程一致的用户启动，否则可能 attach 失败）：

```SHELL
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

- 执行该程序的用户需要和目标进程具有相同的权限。比如以`admin`用户来执行：`sudo su admin && java -jar arthas-boot.jar` 或 `sudo -u admin -EH java -jar arthas-boot.jar`。
- 如果 attach 不上目标进程，可以查看`~/logs/arthas/` 目录下的日志。
- 如果下载速度比较慢，可以使用 aliyun 的镜像：`java -jar arthas-boot.jar --repo-mirror aliyun --use-http`
- `java -jar arthas-boot.jar -h` 打印更多参数信息。

选择应用 java 进程：

```bash
$ $ java -jar arthas-boot.jar
* [1]: 35542
  [2]: 71560 math-game.jar
```

`math-game`进程是第 2 个，则输入 2，再输入`回车/enter`。Arthas 会 attach 到目标进程上，并输出日志：

```shell
[INFO] Try to attach process 71560
[INFO] Attach process 71560 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'


wiki: https://arthas.aliyun.com/doc
version: 3.0.5.20181127201536
pid: 71560
time: 2018-11-28 19:16:24

$
```

### 4. 通过浏览器连接 arthas

Arthas目前支持Web Console，用户在attach成功之后，可以直接访问：`http://127.0.0.1:3658/`。可以填入IP，远程连接其它机器上的arthas。

默认情况下，arthas只listen 127.0.0.1，所以如果想从远程连接，则可以使用 `--target-ip`参数指定listen的IP

### 5. dashboard 仪表板

输入dashboard(仪表板)，按回车`/enter`，会展示当前进程的信息，按`ctrl+c`可以中断执行。

- 1. 第一部分是显示JVM中运行的所有线程：所在线程组，优先级，线程的状态，CPU的占用率，是否是后台进程等
- 2. 第二部分显示的JVM内存的使用情况
- 3. 第三部分是操作系统的一些信息和Java版本号
```sh
$ dashboard
ID     NAME                   GROUP          PRIORI STATE  %CPU    TIME   INTERRU DAEMON
17     pool-2-thread-1        system         5      WAITIN 67      0:0    false   false
27     Timer-for-arthas-dashb system         10     RUNNAB 32      0:0    false   true
11     AsyncAppender-Worker-a system         9      WAITIN 0       0:0    false   true
9      Attach Listener        system         9      RUNNAB 0       0:0    false   true
3      Finalizer              system         8      WAITIN 0       0:0    false   true
2      Reference Handler      system         10     WAITIN 0       0:0    false   true
4      Signal Dispatcher      system         9      RUNNAB 0       0:0    false   true
26     as-command-execute-dae system         10     TIMED_ 0       0:0    false   true
13     job-timeout            system         9      TIMED_ 0       0:0    false   true
1      main                   main           5      TIMED_ 0       0:0    false   false
14     nioEventLoopGroup-2-1  system         10     RUNNAB 0       0:0    false   false
18     nioEventLoopGroup-2-2  system         10     RUNNAB 0       0:0    false   false
23     nioEventLoopGroup-2-3  system         10     RUNNAB 0       0:0    false   false
15     nioEventLoopGroup-3-1  system         10     RUNNAB 0       0:0    false   false

Memory             used   total max    usage GC
heap               32M    155M  1820M  1.77% gc.ps_scavenge.count  4
ps_eden_space      14M    65M   672M   2.21% gc.ps_scavenge.time(m 166
ps_survivor_space  4M     5M    5M           s)
ps_old_gen         12M    85M   1365M  0.91% gc.ps_marksweep.count 0
nonheap            20M    23M   -1           gc.ps_marksweep.time( 0
code_cache         3M     5M    240M   1.32% ms)

Runtime
os.name                Mac OS X
os.version             10.13.4
java.version           1.8.0_162
java.home              /Library/Java/JavaVir
                       tualMachines/jdk1.8.0
                       _162.jdk/Contents/Hom
                       e/jre
```

### 6. 通过 thread 命令来获取到 `math-game` 进程的 Main Class

`thread 1`会打印线程 ID 1 的栈，通常是 main 函数的线程。

```sh
$ thread 1 | grep 'main('
    at demo.MathGame.main(MathGame.java:17)
```

### 7. 通过 jad 来反编译 Main Class
```sh
$ jad demo.MathGame
```

```java
ClassLoader:
+-sun.misc.Launcher$AppClassLoader@3d4eac69
  +-sun.misc.Launcher$ExtClassLoader@66350f69

Location:
/tmp/math-game.jar

/*
 * Decompiled with CFR 0_132.
 */
package demo;

import java.io.PrintStream;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class MathGame {
    private static Random random = new Random();
    private int illegalArgumentCount = 0;

    public static void main(String[] args) throws InterruptedException {
        MathGame game = new MathGame();
        do {
            game.run();
            TimeUnit.SECONDS.sleep(1L);
        } while (true);
    }

    public void run() throws InterruptedException {
        try {
            int number = random.nextInt();
            List<Integer> primeFactors = this.primeFactors(number);
            MathGame.print(number, primeFactors);
        }
        catch (Exception e) {
            System.out.println(String.format("illegalArgumentCount:%3d, ", this.illegalArgumentCount) + e.getMessage());
        }
    }

    public static void print(int number, List<Integer> primeFactors) {
        StringBuffer sb = new StringBuffer("" + number + "=");
        Iterator<Integer> iterator = primeFactors.iterator();
        while (iterator.hasNext()) {
            int factor = iterator.next();
            sb.append(factor).append('*');
        }
        if (sb.charAt(sb.length() - 1) == '*') {
            sb.deleteCharAt(sb.length() - 1);
        }
        System.out.println(sb);
    }

    public List<Integer> primeFactors(int number) {
        if (number < 2) {
            ++this.illegalArgumentCount;
            throw new IllegalArgumentException("number is: " + number + ", need >= 2");
        }
        ArrayList<Integer> result = new ArrayList<Integer>();
        int i = 2;
        while (i <= number) {
            if (number % i == 0) {
                result.add(i);
                number /= i;
                i = 2;
                continue;
            }
            ++i;
        }
        return result;
    }
}

Affect(row-cnt:1) cost in 970 ms.
```
### 8. watch

通过watch命令来查看`demo.MathGame#primeFactors`函数的返回值：

```sh
$ watch demo.MathGame primeFactors returnObj
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 107 ms.
ts=2018-11-28 19:22:30; [cost=1.715367ms] result=null
ts=2018-11-28 19:22:31; [cost=0.185203ms] result=null
ts=2018-11-28 19:22:32; [cost=19.012416ms] result=@ArrayList[
    @Integer[5],
    @Integer[47],
    @Integer[2675531],
]
ts=2018-11-28 19:22:33; [cost=0.311395ms] result=@ArrayList[
    @Integer[2],
    @Integer[5],
    @Integer[317],
    @Integer[503],
    @Integer[887],
]
ts=2018-11-28 19:22:34; [cost=10.136007ms] result=@ArrayList[
    @Integer[2],
    @Integer[2],
    @Integer[3],
    @Integer[3],
    @Integer[31],
    @Integer[717593],
]
ts=2018-11-28 19:22:35; [cost=29.969732ms] result=@ArrayList[
    @Integer[5],
    @Integer[29],
    @Integer[7651739],
]
```

### 9. 退出 arthas

- 如果只是退出当前的连接，可以用`quit`或者`exit`命令。Attach到目标进程上的arthas还会继续运行，端口会保持开放，下次连接时可以直接连接上。
- 如果想完全退出arthas，可以执行`stop`命令。

## 基础命令
- [base64](https://arthas.gitee.io/doc/base64.html) - base64 编码转换，和 linux 里的 base64 命令类似
- [cat](https://arthas.gitee.io/doc/cat.html) - 打印文件内容，和 linux 里的 cat 命令类似
- [cls](https://arthas.gitee.io/doc/cls.html) - 清空当前屏幕区域
- [echo](https://arthas.gitee.io/doc/echo.html) - 打印参数，和 linux 里的 echo 命令类似
- [grep](https://arthas.gitee.io/doc/grep.html) - 匹配查找，和 linux 里的 grep 命令类似
- [help](https://arthas.gitee.io/doc/help.html) - 查看命令帮助信息
- [history](https://arthas.gitee.io/doc/history.html) - 打印命令历史
- [keymap](https://arthas.gitee.io/doc/keymap.html) - Arthas 快捷键列表及自定义快捷键
- [pwd](https://arthas.gitee.io/doc/pwd.html) - 返回当前的工作目录，和 linux 命令类似
- [quit](https://arthas.gitee.io/doc/quit.html) - 退出当前 Arthas 客户端，其他 Arthas 客户端不受影响
- [reset](https://arthas.gitee.io/doc/reset.html) - 重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类
- [session](https://arthas.gitee.io/doc/session.html) - 查看当前会话的信息
- [stop](https://arthas.gitee.io/doc/stop.html) - 关闭 Arthas 服务端，所有 Arthas 客户端全部退出
- [tee](https://arthas.gitee.io/doc/tee.html) - 复制标准输入到标准输出和指定的文件，和 linux 里的 tee 命令类似
- [version](https://arthas.gitee.io/doc/version.html) - 输出当前目标 Java 进程所加载的 Arthas 版本号

## Arthas 命令行快捷键

|快捷键说明|命令说明|
|---|:-:|
|ctrl + a|跳到行首|
|ctrl + e|跳到行尾|
|ctrl + f|向前移动一个单词|
|ctrl + b|向后移动一个单词|
|键盘左方向键|光标向前移动一个字符|
|键盘右方向键|光标向后移动一个字符|
|键盘下方向键|下翻显示下一个命令|
|键盘上方向键|上翻显示上一个命令|
|ctrl + h|向后删除一个字符|
|ctrl + shift + /|向后删除一个字符|
|ctrl + u|撤销上一个命令，相当于清空当前行|
|ctrl + d|删除当前光标所在字符|
|ctrl + k|删除当前光标到行尾的所有字符|
|ctrl + i|自动补全，相当于敲TAB|
|ctrl + j|结束当前行，相当于敲回车|
|ctrl + m|结束当前行，相当于敲回车|

- 任何时候 tab 键，会根据当前的输入给出提示
- 命令后敲 - 或 -- ，然后按 tab 键，可以展示出此命令具体的选项
- ctrl + c: 终止当前命令
- ctrl + z: 挂起当前命令，后续可以 bg/fg 重新支持此命令，或 kill 掉
- ctrl + a: 回到行首
- ctrl + e: 回到行尾
## 命令
### jvm相关
#### dashboard

> dashboard 显示当前系统的实时数据面板，按q或ctrl+c退出 

##### 使用参考
```sh
$ dashboard
ID   NAME                           GROUP           PRIORITY   STATE     %CPU      DELTA_TIME TIME      INTERRUPTE DAEMON
-1   C2 CompilerThread0             -               -1         -         1.55      0.077      0:8.684   false      true
53   Timer-for-arthas-dashboard-07b system          5          RUNNABLE  0.08      0.004      0:0.004   false      true
22   scheduling-1                   main            5          TIMED_WAI 0.06      0.003      0:0.287   false      false
-1   C1 CompilerThread0             -               -1         -         0.06      0.003      0:2.171   false      true
-1   VM Periodic Task Thread        -               -1         -         0.03      0.001      0:0.092   false      true
49   arthas-NettyHttpTelnetBootstra system          5          RUNNABLE  0.02      0.001      0:0.156   false      true
16   Catalina-utility-1             main            1          TIMED_WAI 0.0       0.000      0:0.029   false      false
-1   G1 Young RemSet Sampling       -               -1         -         0.0       0.000      0:0.019   false      true
17   Catalina-utility-2             main            1          WAITING   0.0       0.000      0:0.025   false      false
34   http-nio-8080-ClientPoller     main            5          RUNNABLE  0.0       0.000      0:0.016   false      true
23   http-nio-8080-BlockPoller      main            5          RUNNABLE  0.0       0.000      0:0.011   false      true
-1   VM Thread                      -               -1         -         0.0       0.000      0:0.032   false      true
-1   Service Thread                 -               -1         -         0.0       0.000      0:0.006   false      true
-1   GC Thread#5                    -               -1         -         0.0       0.000      0:0.043   false      true
Memory                     used     total    max      usage    GC
heap                       36M      70M      4096M    0.90%    gc.g1_young_generation.count   12
g1_eden_space              6M       18M      -1       33.33%                                  86
g1_old_gen                 30M      50M      4096M    0.74%    gc.g1_old_generation.count     0
g1_survivor_space          491K     2048K    -1       24.01%   gc.g1_old_generation.time(ms)  0
nonheap                    66M      69M      -1       96.56%
codeheap_'non-nmethods'    1M       2M       5M       22.39%
metaspace                  46M      47M      -1       98.01%
Runtime
os.name                                                        Mac OS X
os.version                                                     10.15.4
java.version                                                   15
java.home                                                      /Library/Java/JavaVirtualMachines/jdk-15.jdk/Contents/Home
systemload.average                                             10.68
processors                                                     8
uptime                                                         272s

```

##### 数据说明

- ID: Java级别的线程ID，注意这个ID不能跟jstack中的nativeID一一对应
- NAME: 线程名
- GROUP: 线程组名
- PRIORITY: 线程优先级, 1~10之间的数字，越大表示优先级越高
- STATE: 线程的状态
- CPU%: 线程消耗的cpu占比，采样100ms，将所有线程在这100ms内的cpu使用量求和，再算出每个线程的cpu使用占比。
- TIME: 线程运行总时间，数据格式为分：秒
- INTERRUPTED: 线程当前的中断位状态
- DAEMON: 是否是daemon线程
###### JVM 内部线程

Java 8 之后支持获取 JVM 内部线程 CPU 时间，这些线程只有名称和 CPU 时间，没有 ID 及状态等信息（显示 ID 为-1）。 通过内部线程可以观测到 JVM 活动，如 GC、JIT 编译等占用 CPU 情况，方便了解 JVM 整体运行状况。

- 当 JVM 堆(heap)/元数据(metaspace)空间不足或 OOM 时，可以看到 GC 线程的 CPU 占用率明显高于其他的线程。
- 当执行`trace/watch/tt/redefine`等命令后，可以看到 JIT 线程活动变得更频繁。因为 JVM 热更新 class 字节码时清除了此 class 相关的 JIT 编译结果，需要重新编译。

JVM 内部线程包括下面几种：

- JIT 编译线程: 如 `C1 CompilerThread0`, `C2 CompilerThread0`
- GC 线程: 如`GC Thread0`, `G1 Young RemSet Sampling`
- 其它内部线程: 如`VM Periodic Task Thread`, `VM Thread`, `Service Thread`

##### 截图展示
![](dashboard.png)
#### thread

> thread 查看当前 JVM 的线程堆栈信息

##### 参数说明
| 参数名称          | 参数说明                             |
| ------------- | -------------------------------- |
| _id_          | 线程 id                            |
| [n:]          | 指定最忙的前 N 个线程并打印堆栈                |
| [b]           | 找出当前阻塞其他线程的线程                    |
| [i `<value>`] | 指定 cpu 使用率统计的采样间隔，单位为毫秒，默认值为 200 |
| [--all]       | 显示所有匹配的线程                        |

##### cpu 使用率是如何统计出来的？

这里的 cpu 使用率与 linux 命令`top -H -p <pid>` 的线程`%CPU`类似，一段采样间隔时间内，当前 JVM 里各个线程的增量 cpu 时间与采样间隔时间的比例。

###### 工作原理说明：

- 首先第一次采样，获取所有线程的 CPU 时间(调用的是`java.lang.management.ThreadMXBean#getThreadCpuTime()`及`sun.management.HotspotThreadMBean.getInternalThreadCpuTimes()`接口)
- 然后睡眠等待一个间隔时间（默认为 200ms，可以通过`-i`指定间隔时间）
- 再次第二次采样，获取所有线程的 CPU 时间，对比两次采样数据，计算出每个线程的增量 CPU 时间
- 线程 CPU 使用率 = 线程增量 CPU 时间 / 采样间隔时间 * 100%

> 注意
> 注意： 这个统计也会产生一定的开销（JDK 这个接口本身开销比较大），因此会看到 as 的线程占用一定的百分比，为了降低统计自身的开销带来的影响，可以把采样间隔拉长一些，比如 5000 毫秒。

>提示
  另外一种查看 Java 进程的线程 cpu 使用率方法：可以使用[`show-busy-java-threads`](https://github.com/oldratlee/useful-scripts/blob/dev-2.x/docs/java.md#-show-busy-java-threads)这个脚本。

##### 举例

```sh
# 展示当前最忙的前 3 个线程并打印堆栈
thread -n 3
```

```sh
$ thread -n 3
"C1 CompilerThread0" [Internal] cpuUsage=1.63% deltaTime=3ms time=1170ms


"arthas-command-execute" Id=23 cpuUsage=0.11% deltaTime=0ms time=401ms RUNNABLE
    at java.management@11.0.7/sun.management.ThreadImpl.dumpThreads0(Native Method)
    at java.management@11.0.7/sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:466)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.processTopBusyThreads(ThreadCommand.java:199)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.process(ThreadCommand.java:122)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.process(AnnotatedCommandImpl.java:82)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.access$100(AnnotatedCommandImpl.java:18)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:111)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:108)
    at com.taobao.arthas.core.shell.system.impl.ProcessImpl$CommandProcessTask.run(ProcessImpl.java:385)
    at java.base@11.0.7/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
    at java.base@11.0.7/java.util.concurrent.FutureTask.run(FutureTask.java:264)
    at java.base@11.0.7/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
    at java.base@11.0.7/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
    at java.base@11.0.7/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
    at java.base@11.0.7/java.lang.Thread.run(Thread.java:834)


"VM Periodic Task Thread" [Internal] cpuUsage=0.07% deltaTime=0ms time=584ms

```

```sh
# 当没有参数时，显示所有线程的信息
thread
```

```sh
# 显示 1 号线程的运行堆栈
thread 1
```

```sh
$ thread 1
"main" Id=1 WAITING on java.util.concurrent.CountDownLatch$Sync@29fafb28
    at sun.misc.Unsafe.park(Native Method)
    -  waiting on java.util.concurrent.CountDownLatch$Sync@29fafb28
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:997)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1304)
    at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:231)

```

```sh
# 找出当前阻塞其他线程的线程，有时候我们发现应用卡住了， 通常是由于某个线程拿住了某个锁， 并且其他线程都在等待这把锁造成的。 为了排查这类问题， arthas提供了thread -b， 一键找出那个罪魁祸首。
thread -b
```

```sh
$ thread -b
"http-bio-8080-exec-4" Id=27 TIMED_WAITING
    at java.lang.Thread.sleep(Native Method)
    at test.arthas.TestThreadBlocking.doGet(TestThreadBlocking.java:22)
    -  locked java.lang.Object@725be470 <---- but blocks 4 other threads!
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:624)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:731)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:303)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:241)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
    at test.filter.TestDurexFilter.doFilter(TestDurexFilter.java:46)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:241)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:220)
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:122)
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:505)
    at com.taobao.tomcat.valves.ContextLoadFilterValve$FilterChainAdapter.doFilter(ContextLoadFilterValve.java:191)
    at com.taobao.eagleeye.EagleEyeFilter.doFilter(EagleEyeFilter.java:81)
    at com.taobao.tomcat.valves.ContextLoadFilterValve.invoke(ContextLoadFilterValve.java:150)
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:170)
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:103)
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:116)
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:429)
    at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1085)
    at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:625)
    at org.apache.tomcat.util.net.JIoEndpoint$SocketProcessor.run(JIoEndpoint.java:318)
    -  locked org.apache.tomcat.util.net.SocketWrapper@7127ee12
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
    at java.lang.Thread.run(Thread.java:745)

    Number of locked synchronizers = 1
    - java.util.concurrent.ThreadPoolExecutor$Worker@31a6493e

```

```sh
# 指定采样时间间隔，每过1000毫秒采样，显示最占时间的3个线程 
thread -i 1000 -n 3
```

```sh
$ thread -n 3 -i 1000
"as-command-execute-daemon" Id=4759 cpuUsage=23% RUNNABLE
    at sun.management.ThreadImpl.dumpThreads0(Native Method)
    at sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:440)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.processTopBusyThreads(ThreadCommand.java:133)
    at com.taobao.arthas.core.command.monitor200.ThreadCommand.process(ThreadCommand.java:79)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.process(AnnotatedCommandImpl.java:96)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl.access$100(AnnotatedCommandImpl.java:27)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:125)
    at com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl$ProcessHandler.handle(AnnotatedCommandImpl.java:122)
    at com.taobao.arthas.core.shell.system.impl.ProcessImpl$CommandProcessTask.run(ProcessImpl.java:332)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:756)

    Number of locked synchronizers = 1
    - java.util.concurrent.ThreadPoolExecutor$Worker@546aeec1
...

```

```sh
# 查看处于等待状态的线程 
thread --state WAITING
```

```sh
[arthas@28114]$ thread --state WAITING
Threads Total: 16, NEW: 0, RUNNABLE: 9, BLOCKED: 0, WAITING: 3, TIMED_WAITING: 4, TERMINATED: 0
ID   NAME                           GROUP           PRIORITY   STATE     %CPU      DELTA_TIME TIME      INTERRUPTE DAEMON
3    Finalizer                      system          8          WAITING   0.0       0.000      0:0.000   false      true
20   arthas-UserStat                system          9          WAITING   0.0       0.000      0:0.001   false      true
14   arthas-timer                   system          9          WAITING   0.0       0.000      0:0.000   false      true

```

#### jvm

> jvm 查看当前 JVM 的信息

```sh
$ jvm
RUNTIME
--------------------------------------------------------------------------------------------------------------
 MACHINE-NAME                   37@ff267334bb65
 JVM-START-TIME                 2020-07-23 07:50:36
 MANAGEMENT-SPEC-VERSION        1.2
 SPEC-NAME                      Java Virtual Machine Specification
 SPEC-VENDOR                    Oracle Corporation
 SPEC-VERSION                   1.8
 VM-NAME                        Java HotSpot(TM) 64-Bit Server VM
 VM-VENDOR                      Oracle Corporation
 VM-VERSION                     25.201-b09
 INPUT-ARGUMENTS                []
 CLASS-PATH                     demo-arthas-spring-boot.jar
 BOOT-CLASS-PATH                /usr/lib/jvm/java-8-oracle/jre/lib/resources.jar:/usr/lib/jvm/java-8-oracle/j
                                re/lib/rt.jar:/usr/lib/jvm/java-8-oracle/jre/lib/sunrsasign.jar:/usr/lib/jvm/
                                java-8-oracle/jre/lib/jsse.jar:/usr/lib/jvm/java-8-oracle/jre/lib/jce.jar:/us
                                r/lib/jvm/java-8-oracle/jre/lib/charsets.jar:/usr/lib/jvm/java-8-oracle/jre/l
                                ib/jfr.jar:/usr/lib/jvm/java-8-oracle/jre/classes
 LIBRARY-PATH                   /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib

--------------------------------------------------------------------------------------------------------------
 CLASS-LOADING
--------------------------------------------------------------------------------------------------------------
 LOADED-CLASS-COUNT             7529
 TOTAL-LOADED-CLASS-COUNT       7529
 UNLOADED-CLASS-COUNT           0
 IS-VERBOSE                     false

--------------------------------------------------------------------------------------------------------------
 COMPILATION
--------------------------------------------------------------------------------------------------------------
 NAME                           HotSpot 64-Bit Tiered Compilers
 TOTAL-COMPILE-TIME             14921(ms)

--------------------------------------------------------------------------------------------------------------
 GARBAGE-COLLECTORS
--------------------------------------------------------------------------------------------------------------
 PS Scavenge                            name : PS Scavenge
 [count/time (ms)]                      collectionCount : 7
                                        collectionTime : 68

 PS MarkSweep                           name : PS MarkSweep
 [count/time (ms)]                      collectionCount : 1
                                        collectionTime : 47

--------------------------------------------------------------------------------------------------------------
 MEMORY-MANAGERS
--------------------------------------------------------------------------------------------------------------
 CodeCacheManager               Code Cache

 Metaspace Manager              Metaspace
                                Compressed Class Space

 Copy                           Eden Space
                                Survivor Space

 MarkSweepCompact               Eden Space
                                Survivor Space
                                Tenured Gen


--------------------------------------------------------------------------------------------------------------
 MEMORY
--------------------------------------------------------------------------------------------------------------
 HEAP-MEMORY-USAGE                      init : 268435456(256.0 MiB)
 [memory in bytes]                      used : 18039504(17.2 MiB)
                                        committed : 181403648(173.0 MiB)
                                        max : 3817865216(3.6 GiB)

 NO-HEAP-MEMORY-USAGE                   init : 2555904(2.4 MiB)
 [memory in bytes]                      used : 33926216(32.4 MiB)
                                        committed : 35176448(33.5 MiB)
                                        max : -1(-1 B)

--------------------------------------------------------------------------------------------------------------
 OPERATING-SYSTEM
--------------------------------------------------------------------------------------------------------------
 OS                             Linux
 ARCH                           amd64
 PROCESSORS-COUNT               3
 LOAD-AVERAGE                   29.53
 VERSION                        4.15.0-52-generic

--------------------------------------------------------------------------------------------------------------
 THREAD
--------------------------------------------------------------------------------------------------------------
 COUNT                          30
 DAEMON-COUNT                   24
 PEAK-COUNT                     31
 STARTED-COUNT                  36
 DEADLOCK-COUNT                 0

--------------------------------------------------------------------------------------------------------------
 FILE-DESCRIPTOR
--------------------------------------------------------------------------------------------------------------
 MAX-FILE-DESCRIPTOR-COUNT      1048576
 OPEN-FILE-DESCRIPTOR-COUNT     100
Affect(row-cnt:0) cost in 88 ms.

```

##### THREAD 相关

- COUNT: JVM当前活跃的线程数
- DAEMON-COUNT: JVM当前活跃的守护线程数
- PEAK-COUNT: 从JVM启动开始曾经活着的最大线程数
- STARTED-COUNT: 从JVM启动开始总共启动过的线程次数
- DEADLOCK-COUNT: JVM当前死锁的线程数

##### 文件描述符相关

- MAX-FILE-DESCRIPTOR-COUNT：JVM进程最大可以打开的文件描述符数
- OPEN-FILE-DESCRIPTOR-COUNT：JVM当前打开的文件描述符数

#### sysprop

> sysprop 查看和修改JVM的系统属性

##### 举例

```sh
# 查看所有属性
sysprop
```

```sh
# 查看单个属性，支持通过tab补全
sysprop java.version
```

```sh
# 修改单个属性
sysprop user.country

sysprop user.country CN
```

```sh
$ sysprop user.country
user.country=US
$ sysprop user.country CN
Successfully changed the system property.
user.country=CN
```

#### sysenv

> sysenv 查看当前JVM的环境属性(System Environment Variables)

##### 举例

```sh
# 查看所有属性
sysenv

# 查看单个环境变量
sysenv USER
```

#### vmoption

> vmoption 查看，更新JVM诊断相关的参数

##### 举例

```sh
# 查看所有的选项
vmoption

# 查看指定的选项
vmoption PrintGCDetails

# 更新指定的选项
vmoption PrintGCDetails true
```

#### getstatic

> getstatic 通过getstatic命令可以方便的查看类的静态属性

##### 语法

```sh
getstatic 类名 属性名
```

##### 举例

```sh
# 显示demo.MathGame类中静态属性random
getstatic demo.MathGame random
```

```sh
$ getstatic demo.MathGame random
field: random
@Random[
    serialVersionUID=@Long[3905348978240129619],
    seed=@AtomicLong[120955813885284],
    multiplier=@Long[25214903917],
    addend=@Long[11],
    mask=@Long[281474976710655],
    DOUBLE_UNIT=@Double[1.1102230246251565E-16],
    BadBound=@String[bound must be positive],
    BadRange=@String[bound must be greater than origin],
    BadSize=@String[size must be non-negative],
    seedUniquifier=@AtomicLong[-3282039941672302964],
    nextNextGaussian=@Double[0.0],
    haveNextNextGaussian=@Boolean[false],
    serialPersistentFields=@ObjectStreamField[][isEmpty=false;size=3],
    unsafe=@Unsafe[sun.misc.Unsafe@2eaa1027],
    seedOffset=@Long[24],
]

```

#### ognl

> ognl 执行ognl表达式，这是从3.0.5版本新增的功能

##### 参数说明

|参数名称|参数说明|
|---|:-:|
|express|执行的表达式|
|[c:]|执行表达式的 ClassLoader 的 hashcode，默认值是SystemClassLoader|
|[x]|结果对象的展开层次，默认值1|

##### 举例

```
# 调用静态函数
ognl '@java.lang.System@out.println("hello")'

# 获取静态类的静态字段
ognl '@demo.MathGame@random'

# 执行多行表达式，赋值给临时变量，返回一个List
ognl '#value1=@System@getProperty("java.home"),
#value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
```

```sh
$ ognl '@java.lang.System@out.println("hello")'
null

$ ognl '@demo.MathGame@random'
@Random[
    serialVersionUID=@Long[3905348978240129619],
    seed=@AtomicLong[125451474443703],
    multiplier=@Long[25214903917],
    addend=@Long[11],
    mask=@Long[281474976710655],
    DOUBLE_UNIT=@Double[1.1102230246251565E-16],
    BadBound=@String[bound must be positive],
    BadRange=@String[bound must be greater than origin],
    BadSize=@String[size must be non-negative],
    seedUniquifier=@AtomicLong[-3282039941672302964],
    nextNextGaussian=@Double[0.0],
    haveNextNextGaussian=@Boolean[false],
    serialPersistentFields=@ObjectStreamField[][isEmpty=false;size=3],
    unsafe=@Unsafe[sun.misc.Unsafe@28ea5898],
    seedOffset=@Long[24],
]
```
#### heapdump

[`heapdump`](https://arthas.aliyun.com/doc/arthas-tutorials.html?language=cn&id=command-heapdump)

提示

dump java heap, 类似 jmap 命令的 heap dump 功能。

##### dump 到指定文件

```sh
[arthas@58205]$ heapdump arthas-output/dump.hprof
Dumping heap to arthas-output/dump.hprof ...
Heap dump file created
```

提示

生成文件在`arthas-output`目录，可以通过浏览器下载： http://localhost:8563/arthas-output/

##### 只 dump live 对象

```sh
[arthas@58205]$ heapdump --live /tmp/dump.hprof
Dumping heap to /tmp/dump.hprof ...
Heap dump file created
```

##### dump 到临时文件

```sh
[arthas@58205]$ heapdump
Dumping heap to /var/folders/my/wy7c9w9j5732xbkcyt1mb4g40000gp/T/heapdump2019-09-03-16-385121018449645518991.hprof...
Heap dump file created
```
### class/classloader相关

#### sc

> sc 查看JVM已加载的类信息，“Search-Class” 的简写，这个命令能搜索出所有已经加载到 JVM 中的Class 信息 sc 默认开启了子类匹配功能，也就是说所有当前类的子类也会被搜索出来，想要精确的匹配，请打开options disable-sub-class true 开关

##### 参数说明

|参数名称|参数说明|
|---|:-:|
|`class-pattern` |类名表达式匹配，支持全限定名，如com.taobao.test.AAA，也支持com/taobao/test/AAA这样的格式，这样，我们从异常堆栈里面把类名拷贝过来的时候，不需要在手动把/替换为.啦。|
|`method-pattern` |方法名表达式匹配|
|`[d]` |输出当前类的详细信息，包括这个类所加载的原始文件来源、类的声明、加载的 ClassLoader等详细信息。 如果一个类被多个ClassLoader所加载，则会出现多次|
|`[E]` |开启正则表达式匹配，默认为通配符匹配|
|`[f]` |输出当前类的成员变量信息（需要配合参数-d一起使用）|

##### 举例

```sh
# 模糊搜索，demo包下所有的类
sc demo.*

# 打印类的详细信息
sc -d demo.MathGame
```

```sh
$ sc demo.*
demo.MathGame
Affect(row-cnt:1) cost in 55 ms.

$ sc -d demo.MathGame
class-info        demo.MathGame
code-source       /private/tmp/math-game.jar
name              demo.MathGame
isInterface       false
isAnnotation      false
isEnum            false
isAnonymousClass  false
isArray           false
isLocalClass      false
isMemberClass     false
isPrimitive       false
isSynthetic       false
simple-name       MathGame
modifier          public
annotation
interfaces
super-class       +-java.lang.Object
class-loader      +-sun.misc.Launcher$AppClassLoader@3d4eac69
                    +-sun.misc.Launcher$ExtClassLoader@66350f69
classLoaderHash   3d4eac69

Affect(row-cnt:1) cost in 875 ms.

```

```sh
# 打印出类的Field信息
sc -df demo.MathGame
```

```sh
$ sc -d -f demo.MathGame
class-info        demo.MathGame
code-source       /private/tmp/math-game.jar
name              demo.MathGame
isInterface       false
isAnnotation      false
isEnum            false
isAnonymousClass  false
isArray           false
isLocalClass      false
isMemberClass     false
isPrimitive       false
isSynthetic       false
simple-name       MathGame
modifier          public
annotation
interfaces
super-class       +-java.lang.Object
class-loader      +-sun.misc.Launcher$AppClassLoader@3d4eac69
                    +-sun.misc.Launcher$ExtClassLoader@66350f69
classLoaderHash   3d4eac69
fields            modifierprivate,static
                  type    java.util.Random
                  name    random
                  value   java.util.Random@522b4
                          08a

                  modifierprivate
                  type    int
                  name    illegalArgumentCount


Affect(row-cnt:1) cost in 19 ms.
```

#### sm
> sm 查看已加载类的方法信息“Search-Method” 的简写，这个命令能搜索出所有已经加载了 Class 信息的方法信息。

sm 命令只能看到由当前类所声明 (declaring) 的方法，父类则无法看到。

##### 参数说明

|参数名称|参数说明|
|---|:-:|
|class-pattern|类名表达式匹配|
|method-pattern|方法名表达式匹配|
|[d]|展示每个方法的详细信息|
|[E]|开启正则表达式匹配，默认为通配符匹配|

##### 举例

```sh
# 显示String类加载的方法
sm java.lang.String
```

```sh
$ sm java.lang.String
java.lang.String-><init>
java.lang.String->equals
java.lang.String->toString
java.lang.String->hashCode
java.lang.String->compareTo
...
```

```sh
# 显示String中的toString方法详细信息
sm -d java.lang.String toString
```

```sh
$ sm -d java.lang.String toString
 declaring-class  java.lang.String
 method-name      toString
 modifier         public
 annotation
 parameters
 return           java.lang.String
 exceptions

Affect(row-cnt:1) cost in 3 ms.

```

#### jad

> jad 反编译指定已加载类的源码

> 反编绎时只显示源代码，默认情况下，反编译结果里会带有ClassLoader信息，通过--source-only选 项，可以只打印源代码。方便和mc/redefine命令结合使用。 jad --source-only demo.MathGame

##### 参数说明

|参数名称|参数说明|
|---|:-:|
|class-pattern|类名表达式匹配|
|[E]|开启正则表达式匹配，默认为通配符匹配|

##### 举例

```sh
# 编译java.lang.String
jad java.lang.String
```

> 反编绎时只显示源代码，默认情况下，反编译结果里会带有ClassLoader信息，通过--source-only选项，可以只打印源代码。方便和mc/redefine命令结合使用。jad --source-only demo.MathGame

```sh
# 反编译指定的函数
jad demo.MathGame main
```

```sh
# 反编译时不显示行号
jad demo.MathGame main --lineNumber false
```
#### mc

> mc Memory Compiler/内存编译器，编译.java文件生成.class

##### 举例

```
# 在内存中编译Hello.java为Hello.class
mc /root/Hello.java

# 可以通过-d命令指定输出目录
mc -d /root/bbb /root/Hello.java
```

> redefine 加载外部的.class文件，redefine到JVM里

- 注意， redefine后的原来的类不能恢复，redefine有可能失败（比如增加了新的field），参考jdk本身的文档。
- reset命令对redefine的类无效。如果想重置，需要redefine原始的字节码。
- redefine命令和jad/watch/trace/monitor/tt等命令会冲突。执行完redefine之后，如果 再执行上面提到的命令，则会把redefine的字节码重置。

##### redefine 的限制

- 不允许新增加field/method
- 正在跑的函数，没有退出不能生效，比如下面新增加的System.out.println，只有run()函数里的会生效

```java
public class MathGame {

public static void main(String[] args) throws InterruptedException {
    MathGame game = new MathGame();
    while (true) {
        game.run();
        TimeUnit.SECONDS.sleep( 1 );
        // 这个不生效，因为代码一直跑在 while里
        System.out.println("in loop");
    }
}

public void run() throws InterruptedException {
// 这个生效，因为run()函数每次都可以完整结束
System.out.println("call run()");
        try {
            int number = random.nextInt();
            List<Integer> primeFactors = primeFactors(number);
            print(number, primeFactors);
        } catch (Exception e) {
            System.out.println(String.format("illegalArgumentCount:%3d, ",
            illegalArgumentCount) + e.getMessage());
        }
    }
}
```

##### 案例：结合 jad/mc 命令使用

```sh
# 1. 使用jad反编译demo.MathGame输出到/root/MathGame.java
jad --source-only demo.MathGame > /root/MathGame.java
```

```sh
#2.按上面的代码编辑完毕以后，使用mc内存中对新的代码编译
mc /root/MathGame.java -d /root
```

```sh
# 3.使用redefine命令加载新的字节码
redefine /root/demo/MathGame.class
```

#### dump

> dump 将已加载类的字节码文件保存到特定目录：logs/arthas/classdump/

##### 参数

|参数名称|参数说明|
|---|:-:|
|class-pattern|类名表达式匹配|
|[c:]|类所属 ClassLoader 的 hashcode|
|[E]|开启正则表达式匹配，默认为通配符匹配|

##### 举例

```
# 把String类的字节码文件保存到~/logs/arthas/classdump/目录下
dump java.lang.String
```

```
# 把demo包下所有的类的字节码文件保存到~/logs/arthas/classdump/目录下
dump demo.*
```

```sh
$ dump java.lang.String
 HASHCODE  CLASSLOADER  LOCATION
 null                   /Users/admin/logs/arthas/classdump/java/lang/String.class
Affect(row-cnt:1) cost in 119 ms.
```

```sh
$ dump demo.*
 HASHCODE  CLASSLOADER                                    LOCATION
 3d4eac69  +-sun.misc.Launcher$AppClassLoader@3d4eac69    /Users/admin/logs/arthas/classdump/sun.misc.Launcher$AppClassLoader-3d4eac69/demo/MathGame.class
             +-sun.misc.Launcher$ExtClassLoader@66350f69
Affect(row-cnt:1) cost in 39 ms.
```

#### classloader

> classloader

- 1. classloader 命令将 JVM 中所有的classloader的信息统计出来，并可以展示继承树，urls等。
- 2. 可以让指定的classloader去getResources，打印出所有查找到的resources的url。对于ResourceNotFoundException异常比较有用。

##### 参数说明

|参数名称|参数说明|
|---|:-:|
|[l]|按类加载实例进行统计|
|[t]|打印所有ClassLoader的继承树|
|[a]|列出所有ClassLoader加载的类，请谨慎使用|
|[c:]|ClassLoader的hashcode|
|[c: r:]|用ClassLoader去查找resource|
|[c: load:]|用ClassLoader去加载指定的类|

##### 举例

```sh
# 按类加载类型查看统计信息
classloader
```

```sh
# 按类加载实例查看统计信息，可以看到类加载的hashCode
classloader -l
```

```sh
# 查看ClassLoader的继承树
classloader -t
```

```sh
# 通过类加载器的hash，查看此类加载器实际所在的位置
classloader -c 680f2737
```

```sh
# 使用ClassLoader去查找指定资源resource所在的位置
classloader -c 680f2737 -r META-INF/MANIFEST.MF
```

```sh
# 使用ClassLoader去查找类的class文件所在的位置
classloader -c 680f2737 -r java/lang/String.class
```

```sh
# 使用ClassLoader去加载类
classloader -c 70dea4e --load java.lang.String
```

##### 小结

###### classloader命令主要作用有哪些？

- 1. 显示所有类加载器的信息
- 2. 获取某个类加载器所在的jar包
- 3. 获取某个资源在哪个jar包中
- 4. 加载某个类

### monitor/watch/trace相关

#### monitor

> monitor

- 方法执行监控对匹配 class-pattern／method-pattern的类、方法的调用进行监控。
- monitor 命令是一个非实时返回命令，实时返回命令是输入之后立即返回，而非实时返回的命令，则是不断的等待目标 Java 进程返回信息，直到用户输入 Ctrl+C 为止。

##### 参数说明

方法拥有一个命名参数 [c:]，意思是统计周期（cycle of output），拥有一个整型的参数值

|参数名称|参数说明|
|---|:-:|
|class-pattern|类名表达式匹配|
|method-pattern|方法名表达式匹配|
|[E]|开启正则表达式匹配，默认为通配符匹配|
|[c:]|统计周期，默认值为120秒|

##### 举例

```
# 过 5 秒统计一次，统计类demo.MathGame中primeFactors方法
monitor -c 5 demo.MathGame primeFactors
```

##### 监控的维度说明

|监控项|说明|
|---|:-:|
|timestamp|时间戳|
|class|Java类|
|method|方法（构造方法、普通方法）|
|total|调用次数|
|success|成功次数|
|fail|失败次数|
|rt|平均耗时|
|fail-rate|失败率|

##### 小结

##### monitor命令的作用是什么？

用来监视一个时间段中指定方法的执行次数，成功次数，失败次数，耗时等这些信息

#### watch

> watch 方法执行数据观测，让你能方便的观察到指定方法的调用情况。 能观察到的范围为：返回值、抛出异常、入参，通过编写OGNL 表达式进行对应变量的查看。

##### 参数说明

###### watch 的参数比较多，主要是因为它能在 4 个不同的场景观察对象

###### 参数名称 参数说明

|参数名称|参数说明|
|---|:-:|
|class-pattern|类名表达式匹配|
|method-pattern|方法名表达式匹配|
|express|观察表达式|
|condition-express|条件表达式|
|[b]|在 方法调用之前 观察|
|[e]|在 方法异常之后 观察|
|[s]|在 方法返回之后 观察|
|[f]|在 方法结束之后 (正常返回和异常返回)观察|
|[E]|开启正则表达式匹配，默认为通配符匹配|
|[x:]|指定输出结果的属性遍历深度，默认为 1|

###### 这里重点要说明的是观察表达式，观察表达式的构成主要由ognl 表达式组成，所以你可以这样写"{params,returnObj}"，只要是一个合法的 ognl 表达式，都能被正常支持。

##### 特别说明

- watch 命令定义了4个观察事件点，即 -b 方法调用前，-e 方法异常后，-s 方法返回后，-f 方法结束后
- 4个观察事件点 -b、-e、-s 默认关闭，-f 默认打开，当指定观察点被打开后，在相应事件点会对观察表达式进行求值并输出
- 这里要注意方法入参和方法出参的区别，有可能在中间被修改导致前后不一致，除了 -b 事件点params 代表方法入参外，其余事件都代表方法出参
- 当使用 -b 时，由于观察事件点是在方法调用前，此时返回值或异常均不存在

##### 举例

```sh
# 观察demo.MathGame类中primeFactors方法出参和返回值，结果属性遍历深度为 2 。params表示所有参数数组，returnObject表示返回值

watch demo.MathGame primeFactors "{params,returnObj}" -x 2
```

```sh
# 观察方法入参，对比前一个例子，返回值为空（事件点为方法执行前，因此获取不到返回值）
watch demo.MathGame primeFactors "{params,returnObj}" -x 2 -b
```

```sh
# 同时观察方法调用前和方法返回后，参数里-n 2，表示只执行两次。这里输出结果中，第一次输出的是方法调用前的观察表达式的结果，第二次输出的是方法返回后的表达式的结果params表示参数，target表示执行方法的对象，returnObject表示返回值

watch demo.MathGame primeFactors "{params,target,returnObj}" -x 2 -b -s -n 2
```

```sh
# 观察当前对象中的属性，如果想查看方法运行前后，当前对象中的属性，可以使用target关键字，代表当前对象
watch demo.MathGame primeFactors 'target'
```

```sh
# 使用target.field_name访问当前对象的某个属性
watch demo.MathGame primeFactors 'target.illegalArgumentCount'
```

```sh
# 条件表达式的例子，输出第 1 参数小于的情况
watch demo.MathGame primeFactors "{params[0],target}" "params[0]<0"
```

#### trace

> trace

> 方法内部调用路径，并输出方法路径上的每个节点上耗时  
> trace 命令能主动搜索 class-pattern／method-pattern 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。  
> 观察表达式的构成主要由ognl 表达式组成，所以你可以这样写"{params,returnObj}"，只要是一个合法的 ognl 表达式，都能被正常支持。  
> 很多时候我们只想看到某个方法的rt大于某个时间之后的trace结果，现在Arthas可以按照方法执行的耗时来进行过滤了，例如trace *StringUtils isBlank '#cost>100'表示当执行时间超过100ms的时候，才会输出trace的结果。  
> watch/stack/trace这个三个命令都支持#cost  

##### 参数说明

| 参数名称              |        参数说明        |
| ----------------- | :----------------: |
| class-pattern     |      类名表达式匹配       |
| method-pattern    |      方法名表达式匹配      |
| condition-express |       条件表达式        |
| [E]               | 开启正则表达式匹配，默认为通配符匹配 |
| [n:]              |       命令执行次数       |
| #cost             |       方法执行耗时       |

##### 举例

```sh
# trace函数指定类的指定方法
trace demo.MathGame run
```

```sh
# 如果方法调用的次数很多，那么可以用-n参数指定捕捉结果的次数。比如下面的例子里，捕捉到一次调用就退出命令。
trace demo.MathGame run -n 1
```

```sh
# 默认情况下，trace不会包含jdk里的函数调用，如果希望trace jdk里的函数，需要显式设置--skipJDKMethod false。
trace --skipJDKMethod false demo.MathGame run
```

```sh
# 据调用耗时过滤，trace大于0.5ms的调用路径
trace demo.MathGame run '#cost > .5'
```

```sh
# 可以用正则表匹配路径上的多个类和函数，一定程度上达到多层trace的效果。
trace -E com.test.ClassA|org.test.ClassB method1|method2|method3
```

#### stack
> stack

> 输出当前方法被调用的调用路径  
> 很多时候我们都知道一个方法被执行，但这个方法被执行的路径非常多，或者你根本就不知道这个方法是从那里被执行了，此时你需要的是 stack 命令。

##### 参数说明

|参数名称|参数说明|
|---|:-:|
|class-pattern|类名表达式匹配|
|method-pattern|方法名表达式匹配|
|condition-express|条件表达式|
|[E]|开启正则表达式匹配，默认为通配符匹配|
|[n:]|执行次数限制|

##### 举例

```sh
# 获取primeFactors的调用路径
stack demo.MathGame primeFactors
```

```sh
# 条件表达式来过滤，第 0 个参数的值小于 0 ，-n表示获取 2 次
stack demo.MathGame primeFactors 'params[0]<0' -n 2
```

```sh
# 据执行时间来过滤，耗时大于 5 毫秒
stack demo.MathGame primeFactors '#cost>5'
```

#### tt

> tt time-tunnel 时间隧道,记录下指定方法每次调用的入参和返回信息，并能对这些不同时间下调用的信息进行观测

- watch 虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。
- 这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。
- 于是乎，TimeTunnel 命令就诞生了。

##### 参数解析

|tt的参数|说明|
|---|:-:|
|-t|记录某个方法在一个时间段中的调用|
|-l|显示所有已经记录的列表|
|-n|次数 只记录多少次|
|-s|表达式 搜索表达式|
|-i|索引号 查看指定索引号的详细调用信息|
|-p|重新调用指定的索引号时间碎片|命令参数解析

命令参数解析

- `-t`
    
    tt 命令有很多个主参数，`-t` 就是其中之一。这个参数的表明希望记录下类 `*Test` 的 `print` 方法的每次执行情况。
    
- `-n 3`
    
    当你执行一个调用量不高的方法时可能你还能有足够的时间用 `CTRL+C` 中断 tt 命令记录的过程，但如果遇到调用量非常大的方法，瞬间就能将你的 JVM 内存撑爆。
    
    此时你可以通过 `-n` 参数指定你需要记录的次数，当达到记录次数时 Arthas 会主动中断 tt 命令的记录过程，避免人工操作无法停止的情况。
    
- `-m 1`
    
    通过 `-m` 参数指定 Class 匹配的最大数量，防止匹配到的 Class 数量太多导致 JVM 挂起，默认值是 50。

##### 使用案例

```sh
# 最基本的使用来说，就是记录下当前方法的每次调用环境现场。
tt -t demo.MathGame primeFactors
```

##### 表格字段说明

|表格字段|字段解释|
|---|:-:|
|INDEX|时间片段记录编号，每一个编号代表着一次调用，后续tt还有很多命令都是基于此编号指定记录操作，非常重要。|
|TIMESTAMP|方法执行的本机时间，记录了这个时间片段所发生的本机时间|
|COST(ms)|方法执行的耗时|
|IS-RET|方法是否以正常返回的形式结束|
|IS-EXP|方法是否以抛异常的形式结束|
|OBJECT|执行对象的hashCode()，注意，曾经有人误认为是对象在JVM中的内存地址，但很遗憾他不是。但他能帮助你简单的标记当前执行方法的类实体|
|CLASS|执行的类名|
|METHOD|执行的方法名|

###### 条件表达式

不知道大家是否有在使用过程中遇到以下困惑

- Arthas 似乎很难区分出重载的方法
- 我只需要观察特定参数，但是 tt 却全部都给我记录了下来

条件表达式也是用 OGNL 来编写，核心的判断对象依然是 Advice 对象。除了 tt 命令之外，watch、trace、stack 命令也都支持条件表达式。

- 解决方法重载

```sh
tt -t *Test print params.length==1
```

- 通过制定参数个数的形式解决不同的方法签名，如果参数个数一样，你还可以这样写

```sh
tt -t *Test print 'params[1] instanceof Integer'
```

- 解决指定参数

```sh
tt -t *Test print params[0].mobile=="13989838402"
```

##### 检索调用记录

当你用 tt 记录了一大片的时间片段之后，你希望能从中筛选出自己需要的时间片段，这个时候你就需要对现有记录进行检索。

```sh
tt -l
```

##### 需要筛选出 primeFactors 方法的调用信息

```sh
tt -s 'method.name=="primeFactors"'
```

##### 查看调用信息

对于具体一个时间片的信息而言，你可以通过 -i 参数后边跟着对应的 INDEX 编号查看到他的详细信息。

```sh
tt -i 1002
```

##### 重做一次调用

当你稍稍做了一些调整之后，你可能需要前端系统重新触发一次你的调用，此时得求爷爷告奶奶的需要前端配合联调的同学再次发起一次调用。而有些场景下，这个调用不是这么好触发的。

tt 命令由于保存了当时调用的所有现场信息，所以我们可以自己主动对一个 INDEX 编号的时间片自主发起一次调用，从而解放你的沟通成本。此时你需要 -p 参数。通过 --replay-times 指定 调用次数，通过 --replay-interval 指定多次调用间隔(单位ms, 默认1000ms)

```sh
tt -i 1002 -p
```

#### options

> options 全局开关

|名称|默认值|描述|
|---|:-:|--:|
|内容|内容|内容|
|内容|内容|内容|
|unsafe|false|是否支持对系统级别的类进行增强，打开该开关可能导致把JVM搞挂，请慎重选择！|
|dump|false|是否支持被增强了的类dump到外部文件中，如果打开开关，class文件会被dump到/${application dir}/arthas-class-dump/目录下，具体位置详见控制台输出|
|batch-re-transform|true|是否支持批量对匹配到的类执行retransform操作|
|json-format|false|是否支持json化的输出|
|disable-sub-class|false|是否禁用子类匹配，默认在匹配目标类的时候会默认匹配到其子类，如果想精确匹配，可以关闭此开关|
|debug-for-asm|false|打印ASM相关的调试信息|
|save-result|false|是否打开执行结果存日志功能，打开之后所有命令的运行结果都将保存到~/logs/arthas-cache/result.log中|
|job-timeout|1d|异步后台任务的默认超时时间，超过这个时间，任务自动停止；比如设置1d, 2h, 3m, 25s，分别代表天、小时、分、秒|
|print-parent-fields|true|是否打印在parent class里的filed|

###### 案例

```sh
# 查看所有的 options
options
```

```sh
# 获取 option 的值
options json-format
```

```sh
# 设置指定的 option 例如，想打开执行结果存日志功能，输入如下命令即可：
options save-result true
```

##### 小结

options的作用是：查看或设置arthas全局环境变量

#### profiler

> profiler 火焰图火焰图

- profiler 命令支持生成应用热点的火焰图。本质上是通过不断的采样，然后把收集到的采样结果生成火焰图。
- 命令基本运行结构是 profiler 命令 [命令参数]

##### 案例

```
# 启动 profiler
profiler start
```

###### 默认情况下，生成的是cpu的火焰图，即event为cpu。可以用--event参数来指定。

```sh
# 显示支持的事件
profiler list
```

```sh
# 获取已采集的 sample 的数量
profiler getSamples
```

```sh
# 查看 profiler 状态
profiler status
```

###### 可以查看当前profiler在采样哪种event和采样时间。

```sh
# 停止 profiler 生成 svg 格式结果
profiler stop
```

```sh
# 默认情况下，生成的结果保存到应用的工作目录下的arthas-output目录。可以通过 --file参数来指定输出结果路径。比如：
profiler stop --file /tmp/output.svg
```

#### 生成 html 格式结果

```
# 默认情况下，结果文件是svg格式，如果想生成html格式，可以用--format参数指定：或者在--file参数里用文件名指名格式。比如--file /tmp/result.html 
profiler stop --format html
```

### 通过浏览器查看 arthas-output 下面的 profiler 结果

##### 默认情况下，arthas使用3658端口，则可以打开： 查看到arthas-output目录下面的profiler结果：

![](arthas-output.jpg)

##### 点击可以查看具体的结果：

![](arthas-output-svg.jpg)

### 火焰图的含义

- 火焰图是基于 perf 结果产生的SVG 图片，用来展示 CPU 的调用栈。
    - y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。
    - x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。注意，x 轴不代表时间，而是所有的调用栈合并后，按字母顺序排列的。

##### 火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"（plateaus），就表示该函数可能存在性能问题。

##### 颜色没有特殊含义，因为火焰图表示的是 CPU 的繁忙程度，所以一般选择暖色调。

###  小结

|profiler|命令作用|
|---|:-:|
|profiler start|启动profiler，默认情况下，生成cpu的火焰图|
|profiler list|显示所有支持的事件|
|profiler getSamples|获取已采集的sample的数量|
|profiler status|查看profiler的状态，运行的时间|
|profiler stop|停止profiler，生成火焰图的结果，指定输出目录和输出格式：svg或html|

