## 概述

### Arthas（阿尔萨斯） 能为你做什么？

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220117180324.png "QQ截图20201229183512.png")

Arthas 是Alibaba开源的Java诊断工具，深受开发者喜爱。当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：
- 1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
- 2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
- 3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
- 4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
- 5. 是否有一个全局视角来查看系统的运行状况？
- 6. 有什么办法可以监控到JVM的实时运行状态？
- 7. 怎么快速定位应用的热点，生成火焰图？
## 安装与卸载
### 安装

- 下载arthas-boot.jar，然后用java -jar的方式启动：
	```
	curl -O https://alibaba.github.io/arthas/arthas-boot.jar
	java -jar arthas-boot.jar
	```

- 如果下载速度比较慢，可以使用aliyun的镜像：
    ```
    java -jar arthas-boot.jar --repo-mirror aliyun --use-http
    ```
### 卸载

```
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
            throw new IllegalArgumentException("number is: " + number + ", need >=
                    2");
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

### [2. 启动 Demo](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_2-%e5%90%af%e5%8a%a8-demo)

```
#下载已经打包好的arthas-demo.jar
curl -O https://alibaba.github.io/arthas/arthas-demo.jar

#在命令行下执行
java -jar arthas-demo.jar
```

#### [效果](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=%e6%95%88%e6%9e%9c)

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118123825.png "QQ截图20201229183512.png")

### [3. 启动 arthas](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_3-%e5%90%af%e5%8a%a8-arthas)

- 1. 因为arthas-demo.jar进程打开了一个窗口，所以另开一个命令窗口执行arthas-boot.jar
- 2. 选择要粘附的进程：arthas-demo.jar

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118124000.png "QQ截图20201229183512.png")

- 3. 如果粘附成功，在arthas-demo.jar那个窗口中会出现日志记录的信息，记录在c:\Users\Administrator\logs目录下

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118124028.png "QQ截图20201229183512.png")

- 4. 如果端口号被占用，也可以通过以下命令换成另一个端口号执行

```
java -jar arthas-boot.jar --telnet-port 9998 --http-port -1
```

### [4. 通过浏览器连接 arthas](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_4-%e9%80%9a%e8%bf%87%e6%b5%8f%e8%a7%88%e5%99%a8%e8%bf%9e%e6%8e%a5-arthas)

Arthas目前支持Web Console，用户在attach成功之后，可以直接访问：[http://127.0.0.1:3658/。可以填入IP，远程连接其它机器上的arthas。](http://127.0.0.1:3658/%E3%80%82%E5%8F%AF%E4%BB%A5%E5%A1%AB%E5%85%A5IP%EF%BC%8C%E8%BF%9C%E7%A8%8B%E8%BF%9E%E6%8E%A5%E5%85%B6%E5%AE%83%E6%9C%BA%E5%99%A8%E4%B8%8A%E7%9A%84arthas%E3%80%82)

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118133301.png "QQ截图20201229183512.png")

默认情况下，arthas只listen 127.0.0.1，所以如果想从远程连接，则可以使用 --target-ip参数指定listen的IP

### [5. dashboard 仪表板](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_5-dashboard-%e4%bb%aa%e8%a1%a8%e6%9d%bf)

输入dashboard(仪表板)，按回车/enter，会展示当前进程的信息，按ctrl+c可以中断执行。

注：输入前面部分字母，按tab可以自动补全命令

- 1. 第一部分是显示JVM中运行的所有线程：所在线程组，优先级，线程的状态，CPU的占用率，是否是后台进程等
- 2. 第二部分显示的JVM内存的使用情况
- 3. 第三部分是操作系统的一些信息和Java版本号

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118135300.png "QQ截图20201229183512.png")

### [6. 通过 thread 命令来获取到 arthas-demo 进程的 Main Class](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_6-%e9%80%9a%e8%bf%87-thread-%e5%91%bd%e4%bb%a4%e6%9d%a5%e8%8e%b7%e5%8f%96%e5%88%b0-arthas-demo-%e8%bf%9b%e7%a8%8b%e7%9a%84-main-class)

获取到arthas-demo进程的Main Class

thread 1会打印线程ID 1的栈，通常是main函数的线程。

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118135353.png "QQ截图20201229183512.png")

### [7. 通过 jad 来反编译 Main Class](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_7-%e9%80%9a%e8%bf%87-jad-%e6%9d%a5%e5%8f%8d%e7%bc%96%e8%af%91-main-class)

```
jad demo.MathGame
```

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118135415.png "QQ截图20201229183512.png")

### [8. watch](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_8-watch)

通过watch命令来查看demo.MathGame#primeFactors函数的返回值：

```
watch demo.MathGame primeFactors returnObj
```

### [9. 退出 arthas](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=_9-%e9%80%80%e5%87%ba-arthas)

- 如果只是退出当前的连接，可以用quit或者exit命令。Attach到目标进程上的arthas还会继续运行，端口会保持开放，下次连接时可以直接连接上。
- 如果想完全退出arthas，可以执行stop命令。

|命令|功能|
|---|---|
|dashboard|会展示当前进程的信息|
|thread|打印指定编号的线程调用栈|
|jad|反编译指定的类|
|watch|查看指定方法的返回值|

## [基础命令](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=%e5%9f%ba%e7%a1%80%e5%91%bd%e4%bb%a4)

> help 查看命令帮助信息

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118135825.png "QQ截图20201229183512.png")

> cat 打印文件内容，和linux里的cat命令类似 注：汉字有乱码的问题 如果没有写路径，则显示当前目录下的文件

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118135919.png "QQ截图20201229183512.png")

> grep 匹配查找，和linux里的grep命令类似，但它只能用于管道命令

|参数列表|作用|
|---|---|
|-n|显示行号|
|-i|忽略大小写查找|
|-m 行数|最大显示行数，要与查询字符串一起使用|
|-e "正则表达式"|使用正则表达式查找|

#### [举例](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=%e4%b8%be%e4%be%8b)

```
# 只显示包含java字符串的行系统属性
sysprop | grep java
```

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140240.png "QQ截图20201229183512.png")

```
# 显示包含java字符串的行和行号的系统属性
sysprop | grep java -n
```

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140252.png "QQ截图20201229183512.png")

```
# 显示包含system字符串的 10 行信息
thread | grep system -m 10
```

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140303.png "QQ截图20201229183512.png")

```
# 使用正则表达式，显示包含 2 个o字符的线程信息
thread | grep -e "o+"
```

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140356.png "QQ截图20201229183512.png")

> pwd 返回当前的工作目录，和linux命令类似

> cls 清空当前屏幕区域

> session 查看当前会话的信息

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140508.png "QQ截图20201229183512.png")

> reset 重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类

```
# 还原指定类 
reset Test 
# 还原所有以List结尾的类 
reset *List 
# 还原所有的类 
reset
```

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140625.png "QQ截图20201229183512.png")

> version 输出当前目标 Java 进程所加载的 Arthas 版本号

> history 打印命令历史

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140709.png "QQ截图20201229183512.png")

> quit 退出当前 Arthas 客户端，其他 Arthas 客户端不受影响

> stop 关闭 Arthas 服务端，所有 Arthas 客户端全部退出

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140754.png "QQ截图20201229183512.png")

> keymap Arthas快捷键列表及自定义快捷键

![输入图片说明](https://bright-boy.gitee.io/technical-notes/jvm/images/QQ%E6%88%AA%E5%9B%BE20220118140835.png "QQ截图20201229183512.png")

##### [Arthas 命令行快捷键](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=arthas-%e5%91%bd%e4%bb%a4%e8%a1%8c%e5%bf%ab%e6%8d%b7%e9%94%ae)

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

#### [后台异步命令相关快捷键](https://bright-boy.gitee.io/technical-notes/#/jvm/arthas?id=%e5%90%8e%e5%8f%b0%e5%bc%82%e6%ad%a5%e5%91%bd%e4%bb%a4%e7%9b%b8%e5%85%b3%e5%bf%ab%e6%8d%b7%e9%94%ae)

- ctrl + c: 终止当前命令
- ctrl + z: 挂起当前命令，后续可以 bg/fg 重新支持此命令，或 kill 掉
- ctrl + a: 回到行首
- ctrl + e: 回到行尾