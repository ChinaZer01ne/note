#flashcards 
# JVM包括哪些部分？::
- 类加载子系统  
- 运行时数据区  
- 执行引擎（Execution Engine）  
- 本地方法接口（Native Method Interface）（JNI）  
- 本地方法库(Native Method Liberary
![](JVM结构图.png)
# 类加载机制

## 类加载过程
- [加载](#加载)  
- [连接](#连接)
	- [验证](#验证)
	- [解析](#解析)
	- [准备](#准备)
- [初始化](#初始化)  
### 加载

查找并加载类的二进制数据，在内存中生存类模板。

根据虚拟机规范，在加载阶段，虚拟机需要完成下面三件事：

**1）通过一个类的全限定名，来获取类的二进制流。**

**2）将这个字节流代表的静态存储结构转化为方法区的运行时数据结构（类模板）。**

**3）在堆中生成代表这个类的Class对象，作为方法区这个类各种数据访问的入口。**

> 因为没有规定Class类读取的来源，所以出现了从jar读取，从网络读取，有其他文件生成，数据库读取等等方式。
> 关于数组类，数组类本身不是由类加载器创建的，而是JVM在运行时根据需要直接创建，但数组元素类型（引用类型）仍需要类加载器加载。

### 连接

加载和连接阶段是交叉进行的，如一部分字节码文件格式的验证动作，加载还没完成，验证阶段或许已经开始了。

连接过程又分为三个阶段：**验证、准备、解析**。
#### 验证

确保被加载类的正确性并且不会危害虚拟机自身安全。

验证阶段大致会完成下面4个阶段的检验动作。**文件格式验证****、****元数据验证****、****字节码验证****、****符号引用验证**。

##### 文件格式验证（加载阶段执行）

- 是否以魔数0xCAFEBABE开头
- 主、次版本号是否在当前虚拟机处理范围内
- Class文件中各个部分及文件本身是否有被删除的或附加的其他信息
- ...

**第一阶段的主要目的是保证输入的字节流能正确的解析并存储于方法区之内，格式上符合描述一个Java类型信息的要求。该阶段的验证是基于二进制字节流进行的，只有通过了这个阶段验证后，字节流才会进入内存的方法区中进行存储**，也就是说，在加载的阶段连接阶段已经开始了，说明存在交叉。

##### 元数据验证
- 这个类是否有父类（除了Object，都存在父类）
- 这个类的父类是否继承了不允许被继承的类（final修饰的类）
- 如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
- ...

**第二阶段的主要目的是对类的元数据信息进行语义校验，保证不存在不符合Java语言规范的元数据信息。**

##### 字节码验证
- 保证任意时刻操作数栈的数据类型于指令代码序列都能配合工作，例如不会出现类似这样的情况：在操作栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中。
- 保证跳转指令不会跳转到方法体以外的字节码指令上。
- 保证方法体中的类型转换是有效的。
- ...
   
 **第三阶段验证的主要目的是通过对数据流和控制流分析，确定程序语义是合法的、符合逻辑的。**

##### 符号引用验证（在解析阶段执行）

第四阶段的校验发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三阶段——解析阶段中发生。符号引用验证可以看作是常量池中各种符号引用信息的匹配性校验。(**在解析阶段执行**)

- 符号引用中通过字符串描述的全限定类名是否能找到对应的类。
- 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段。
- 符号引用中的类、字段、方法的访问性（private、protected、public、default）是否可被当前类访问。

**符号引用验证的目的是确保解析动作能正常执行。对于虚拟机的类加载机制来说，验证阶段是一个非常重要的，但不是一定必要（因为对程序运行期没有影响）的简短。如果所运行的全部代码（包括自己编写的及第三方包中的代码）都已经被反复使用和验证过，那么在实施阶段就可以考虑使用`-Xverify:none`参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。**

#### 准备

为类的**静态变量分配内存**，并将其**初始化为默认值**的阶段。这些变量所使用的内存都将在方法区中进行分配。
> final 常量在编译阶段就指定了值，在准备阶段显式赋值。
> 如果是通过字面量来定义String，也是在准备阶段显示赋值。
> 如果是对象引用类型则不符合上面规则。
#### 解析

把类、接口、方法、字段的符号引用转换为直接引用。
* 符号引用：字节码中的字面量（比如方法名，字段）
* 直接引用：加载到内存，在内存中的地址（比如方法、字段的实际地址）
### 初始化

为类的静态变量赋予正确的初始值。

- 假如这个类还没有被加载和连接，那么先进行加载和连接
- 假如类存在直接父类，并且这个父类还没有被初始化，那么先初始化直接父类
- 假如类中存在初始化语句，那就依次执行这些初始化语句
#### 主动引用和被动引用

##### **主动引用**
**有六种情况必须立即对类进行“初始化”**

1. 遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有：
2. 使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需要先触发其初始化。
3. 当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
5. 当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。
6. 当一个接口中定义了JDK 8新加入的默认方法（被default关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

##### **被动使用**

被动使用不会引起类的初始化操作。

1. 当访问一个静态字段时，只有真正声明这个字段的类才会被初始化。当通过子类引用父类静态变量，不会导致子类的初始化（但子类会被加载）。也就是说一个类可能被加载但不被初始化
2. 通过数组定义类引用，不会触发此类的初始化（但会加载）
3. 引用常量不会触发类或接口的初始化。因为常量在链接准备阶段就被赋值了
4. 调用ClassLoader类的loadClass方法加载一个类，并不是对类的主动使用，不会导致类的初始化
## 类的卸载

### 什么情况下类会被卸载？::
* 该类所有对象被回收，Java堆中不存在该类及其任何派生子类的实例
* 加载该类的类加载器已经被回收了。这个条件除非是经过精心设计的可替换类加载器的场景，如OSGI、JSP的重加载等。否则类加载器通常不会被回收。
* 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法

### 类加载器的卸载
* 启动类加载器在JVM整个运行期间不可能被卸载（JVM规范和JLS规范）
* 被系统类加载器和扩展类加载器加载的类型在运行期间不太可能被卸载，因为系统类加载器实例或者扩展类加载器的实例基本上在整个运行期间总能直接或间接的访问到，其unreachable的可能性极小。
* 自定义类加载器只有在简单的上下文环境中才能被卸载，而且需借助强制调用虚拟机的垃圾收集功能才可以做到。

总之，类加载器很难被卸载。

## 类加载器有哪些？::、
* 启动类加载器（Bootstrap Class Loader）
    - 是虚拟机的一部分，负责加载Java核心库（如rt.jar）。
    - 它是由C++编写的，不是Java类加载器实现的。
- 扩展类加载器（Extension Class Loader）
    - 负责加载Java的扩展库（如jre\lib\ext目录下的jar包）。
    - 由sun.misc.Launcher$ExtClassLoader类实现。
- 应用程序类加载器（Application Class Loader）
    - 也称为系统类加载器（System Class Loader）。
    - 负责加载应用程序的类路径（Classpath）下的类和资源。
    - 通过调用ClassLoader.getSystemClassLoader()获取。
- 自定义类加载器：
    - 可以通过继承java.lang.ClassLoader类来自定义类加载器。
    - 自定义类加载器可以根据特定需求实现不同的加载逻辑，例如从数据库、网络、动态生成等地方加载类文件。
## 一个类只会被加载一次吗？::
### 类加载器的命名空间
- 每个类加载器都有自己的命名空间，命名空间由该加载器及所有父加载器所加载的类组成。
- 在同一个命名空间中，不会出现类的完整名字（包括类的包名）相同的两个类。意思就是不会出现两个相同的类。
- 在不同的命名空间中，有可能会出现类的完整名字（包括类的包名）相同的两个类。意思是可能出现多个相同的类。
- 如果一个class被加载了两次，加载到了不同命名空间，那么他们也不算是一个类，他们是不能进行相互赋值的。意思是不同命名空间的相同名字的类算作两个不同的类。在运行期，一个Java类是由该类的全限定类名（binary name）和用于加载该类的定义类加载器（definer loader）所共同决定的。如果同样的名字（全限定类名）的类是由两个不同的类加载器加载的，那么这些类就是不同的，即使.class文件的字节码一样，并且从相同位置加载，亦是如此。

**命名空间产生的一些问题：**

* **如果`A类`引用了`B`类，`A类`是由系统加载器加载的，`B类`是由自定义加载器加载的，那么在运行时就会报错，因为系统类加载器的命名空间中没有自定义加载器命名空间中的类（`B类`）；或者`A类`和`B类`是由两个不同的自定义加载器分别加载的，也会产生命名空间问题。**

* **同一个命名空间的类是相互可见的。子加载器所加载的类能够访问到父加载器加载的类，但父加载器加载的类无法访问到子加载器所加载的类。**
## 什么是双亲委派机制? ::
类加载器按照从上到下的顺序形成了一个层次结构，即双亲委派模型。该模型可以确保类的隔离性和安全性，并减少类的重复加载。

当一个类加载器接收到类加载请求时，它会按照以下步骤进行，具体实现可查看`ClassLoader#findClass`方法：
1. 当一个类加载器收到加载类的请求时，首先会检查自己是否已经加载了该类。如果已经加载了，则直接返回该类的Class对象，加载过程结束。
2. 如果该类还没有被加载过，那么该类加载器会将加载请求委派给它的父类加载器。
3. 父类加载器会按照相同的方式进行检查和委派，即先检查自己是否已经加载了该类，若加载了则返回Class对象，否则将加载请求继续委派给它的父加载器。
4. 这个委派过程会一直往上追溯，直到达到最顶层的启动类加载器（Bootstrap ClassLoader），如果找到了所需的类，则加载成功并返回Class对象。
5. 如果最顶层的启动类加载器仍然无法找到所需的类，那么会逐级向下，依次询问是否能够加载。一旦有一个类加载器能够加载该类，就会返回Class对象。
6. 如果所有的加载器都无法加载该类，那么最后由当前类加载器自己尝试加载。如果加载成功，则返回Class对象；如果加载失败，会抛出ClassNotFoundException异常。
![](jdk1.8前类加载器.png)
## 双亲委派机制有什么优劣？::
优点：
* 避免类的重复加载
* 避免加载恶意代码
缺点：
* 上层ClassLoader无法加载底层ClassLoader加载的类
## 如何打破双亲委派机制? ::
- 重写ClassLoader  
- SPI  
	- 线程上下文类加载器
- OSGI
## 自定义类加载器？::

### 为什么自定义类加载器？::
* 隔离加载类
* 修改类加载方式
* 扩展加载源
	* 从不同位置加载类
* 防止源码泄露
	* 加密字节码
### 如何自定义类加载器？::
* 继承`ClassLoader`
* 重写`loadClass`方法：双亲委派机制（不建议）
* 重写`findClass`方法：建议

# 运行时数据区
## 运行时数据区包括哪些部分？::
* 程序计数器（Program Counter Register）
* Java虚拟机栈（Java Virtual Machine Stack）
* 本地方法栈（Native Method Stack）
* 堆（Heap）
* 方法区（Method Area）

## 堆

### 对象

#### 有哪些对象引用类型
- 强引用（Strongly Reference）  
	>强引用是最常见的引用类型。只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。
- 软引用（Soft Reference）  
	>软引用允许在内存不足时回收对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，以避免OutOfMemoryError。
- 弱引用（Weak Reference）  
	>弱引用比软引用更加脆弱。如果一个对象只有弱引用指向，那么在下一次垃圾回收时就会被回收。弱引用主要用于实现一些特定场景下的缓存或数据结构。
- 虚引用（Phantom Reference）
	>虚引用是最弱的引用类型。它的作用并不是用来获取对象的引用，而是用来跟踪对象的回收状态。当虚引用指向的对象被垃圾回收器回收时，会被放入一个引用队列中，通过检查引用队列可以了解对象何时被回收。


# 执行引擎（Execution Engine）  
## 执行引擎包括哪些部分？::
- 解释器（Interpreter）
	>解释器是JVM执行引擎的核心组件之一。它逐行解析字节码并将其翻译成对应的本地机器指令，然后执行这些指令。解释器的优点是启动速度快，但由于每次都需要解释和执行字节码，因此执行效率相对较低。
- 即时编译器（Just-In-Time Compiler，JIT）  
	>即时编译器是一种在运行时将热点代码（经常被执行的代码）直接编译成本地机器码的技术。JIT编译器可以提高代码的执行速度，因为本地机器码的执行通常比解释器更快。JVM中有两种类型的即时编译器：客户端编译器（C1）和服务器编译器（C2），它们根据代码的特性和执行情况来选择合适的编译策略。
- 垃圾回收器（Garbage Collector）
	>垃圾回收器负责自动回收不再使用的对象内存空间，并进行内存的管理和整理。垃圾回收器的设计和实现对于应用程序的性能和响应时间至关重要。
## GC

### 如何确定垃圾？::
- 引用计数法  
- 可达性分析
### 有哪些常见的垃圾回收算法？
- 标记清除算法
- 标记整理法
- 复制回收算法  
- 三色标记
- 增量回收
### 有哪些常见的垃圾回收器？
* 新生代收集器
	* `Serial` 收集器(-XX:+UseSerialGC，单线程+复制回收+STW)  
	* `ParNew` 收集器(-XX: +UseParNewGC，多线程+复制回收+STW)  
		* Serial的多线程版本
	* `Parallel Scavenge` 收集器(-XX:+UserParallelGC，多线程+复制回收+STW)
		* **吞吐量优先**：不需要和用户交互的程序，大多是后台计算程序
		* JDK8默认
		* 常用参数
			* -XX:ParallelGCThreads 
			* -XX:MaxGCPauseMillis
			* -XX:GCTimeRatio
			* -XX:+UseAdaptiveSizePolicy
* 老年代收集器
	* `Serial Old` 收集器(-XX:+UseSerialGC，单线程+标记整理+STW)  
	* `Parallel Old` 收集器(-XX: +UseParallelOldGC，多线程+标记整理+STW)  
	* `CMS` 收集器（-XX:UseConcMarkSweepGC，，并发+标记清除算法+STW）  
		* 低延迟
* 整堆收集器
	* `G1`（Garbage First） 收集器
		* jdk9默认
#### 垃圾回收器组合
![](垃圾回收器组合.jpg)
* Serial Old可作为CMS回收失败的后备方案
* Serial + CMS 和 ParNew + Serial Old再jdk8中废弃，jdk9移除
* jdk14中弃用 Parallel Scavenge + Serial Old组合
* jdk9弃用CMS，jdk14中删除CMS垃圾回收器
### 评估GC的性能指标
* **吞吐量**
	* 运行用户代码的时间占总运行时间的比例
	* 注重吞吐量的应用，一般是暂停次数较少(CPU切换少)，每次暂停时间相对较长，对快速响应不敏感
* **暂停时间**
	* 执行垃圾收集时，程序工作线程被暂停的时间
	* 注重低延迟的应用，一般是暂停次数相对较多(CPU切换多)，每次暂停时间较短，需要快速响应
* **内存占用**
	* Java堆区所占的内存大小，内存越大回收时间会变长
* 收集频率
	* 收集操作发生的频率
* 快速
	* 一个对象从诞生到被回收所经历的时间

吞吐量、暂停时间是相互矛盾的，我们更**优先关注暂停时间，其次吞吐量**。**注重吞吐量的应用，一般是暂停次数较少，每次暂停时间相对较长，对快速响应不敏感**，**注重低延迟的应用，一般是暂停次数相对较多，每次暂停时间较短，需要快速响应**。每个垃圾回收会有自己的设计目标，要么是吞吐量，要么是暂停时间。现在的标准是：**在最大吞吐量优先的情况下，降低暂停时间；在可控的暂停时间内，提高吞吐量**。

### 根节点枚举？::
Hotspot中使用oopMap记录存活对象，完成根节点枚举。oopMap是在安全点的时候记录的。
### 什么是安全点和安全区域？::
当进行STW的时候用户线程要到附近的安全点停下来。Hotspot中当要垃圾收集的时候，会设置一个标志位，用户线程会轮询这个标志位，一旦发现标志位为真，用户线程就会到附近的安全点挂起。**安全点**的位置是在指令序列复用的地方，如方法调用、循环跳转、异常跳转等位置。如果用户线程是在没法轮询标志位的时候呢，比如用户线程在sleep状态，那就需要**安全区域**的概念了，这些位置引用不会发生变化，可以随时进行垃圾收集，可以立即安全区域是扩大的安全点。当用户线程运行到安全区域的时候会标识自己进入了安全区域，如果垃圾收集没有完成，用户线程会等待直到垃圾收集完成。

### 什么是三色标记？::
我们把遍历对象图时按照"是否标记"将对象分为三种颜色：
* 白色
	* 对象尚未被垃圾回收器访问过，在开始阶段所有对象都是白色，在分析结束阶段，还是白色的对象代表不可达。
* 灰色
	* 对象已经被垃圾回收器访问过，但该对象至少存在一个引用未被扫描。
* 黑色
	* 对象已经被垃圾回收器访问过，且这个对象的所有引用都被扫描过，他是存活对象，如果有其它对象指向了黑色对象，也不需要重新扫描。黑色对象不可能直接（不经过灰色对象）引用白色对象。

垃圾收集的过程，是对象图由黑到白推进的过程。
### 并发标记阶段存在的问题？::
由于垃圾回收线程和用户线程一起执行，由于引用会发生变化，那么就会出现两种情况
* 不能回收的对象被回收了，白色对象又被引用到了黑色对象上。
* 需要回收的没有被回收，这种情况问题不大，虽然有浮动垃圾，但可以放到下次GC

当且仅当同时满足下面两个条件的时候会发生“对象消失”的情况：
* 赋值器插入了一条或多条从黑色对象到白色对象的新引用。
* 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。

**如何解决引用变更的问题？**

* **增量标记（CMS）**： 破坏了第一个条件
	* 当黑色对象插入了新的到白色对象的引用关系的时候，先将这个引用记录下来，等并发扫描结束后，将记录的引用关系中黑色对象为根，重新扫描，可以理解为这些黑色对象变成了灰色，需要重新扫描。
* **原始快照（G1）**：破坏了第二个条件
	* 当灰色对象要删除指向白色对象的引用关系的时候，就将要删除的引用记录下来，等并发扫描结束后，将记录的引用关系中灰色对象为根，重新扫描一次。相当于无论引用是否删除，都会按照刚开始扫描那一刻的对象图快照来进行搜索。

**如何解决新创建对象导致引用变更的问题？**
在G1中，如果仅仅使用**原始快照**的方式，在并发标记阶段，由于新创建对象会导致黑色节点引用指向了白色节点，分析结束后会导致这些新对象会被清除，如何解决？G1为每个Region设计了两个名为“**TAMS（Top at Mark Start）**”的指针，把Region中的一部分空间划分出来用于并发回收过程中的新对象分配，并发回收时新分配的对象地址必须要在这两个指针位置以上。G1收集器默认在这个地址以上的对象是被隐式标记过的，即默认他们是存活对象，不纳入回收范围。

**新创建对象过快会有什么问题？**
与CMS中的“Concurrent Mode Failure”失败会导致Full GC类似，如果内存回收的速度赶不上内存分配的速度，G1收集器也要被迫冻结用户线程执行，导致Full GC而产生长时间STW。
### 说说CMS垃圾收集器的执行流程？::
![](CMS执行流程.jpg)
* 初始标记  
	* **仅仅标记出GC Roots能直接关联到的对象，速度很快。**
	* STW
	* jdk7是单线程，jdk8是多线程
* 并发标记  
	* **从GC Roots的直接关联对象开始遍历整个对象图的过程，耗时较长，但不需要STW**
	* 通过**增量更新**来解决对象消失的问题
* 重新标记
	* **修正并发标记期间因用户程序继续运行而导致标记变动的对象信息(针对垃圾变为非垃圾)**
	* STW
* 并发清除
	* **清理删除掉标记阶段判断的已经死亡的对象，释放内存空间**
### CMS有什么弊端？::
* 标记清除，内存碎片
* 由于和用户线程并行，可能会抢占用户线程资源，对CPU资源敏感
* 无法处理浮动垃圾
	* 并发标记阶段如果产生新的垃圾对象，CMS无法对这些对象进行标记，最终这些垃圾无法被及时回收，只能等到下次GC
### 什么是G1？特点与弊端？::
* 区域划分（Eden、S0、S1、Old、Humongous）
	* Eden、S0、S1
	* Old
	* Humongous（超过Eden区一半大小的对象被认为是大对象，直接放在Humongous区域）
* 区域价值回收（优先队列）
* 并行与并发
	* 并行时STW
	* 并发时与用户线程交替执行
* 分代收集
	* 兼顾年轻代和老年代
	* 区域利用
* 空间整合
	* Region之间时复制算法
	* 整体上标记压缩
* 可预测的时间模型
	* 根据暂停时间优先选择价值大的区域回收
* 可用用户线程协助进行垃圾回收

缺点：
* G1内存占用高，程序额外负载高于CMS
	* 比如RememberSet等等都会占内存
* 小内存应用CMS由于G1，大内存应用G1由于CMS，平衡点在6-8GB之间
### 说说G1垃圾收集器的执行流程？::
* 初始标记（Initial Marking） 
	* 仅仅是标记一下GC Roots能直接关联到的对象，并且修改TAMS指针的值，让下一阶段用户线程并发运行时，能正确的在可用的Region中分配新对象。这个过程需要STW，但耗时很短，而且是借用进行Minor GC的时候同步完成的，所以G1收集器在这个阶段实际并没有额外的停顿。
* 并发标记（Concurrent Marking）
	* 从GC Roots开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户线程并发执行，当对象图扫描完成后，还要重新处理SATB记录下的并发过程中引用变动的对象。
	* 通过“[**原始快照**](#并发标记阶段存在的问题？)”的方式（**SATB**）解决对象消失的问题。
	* 通过[**TAMS**](#并发标记阶段存在的问题？)来解决新分配对象被回收的问题。
* 最终标记（Final Marking）
	* 对用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留下来的少量的SATB记录
	* STW
* 筛选回收（Live Data Counting and Evacuation）
	* 负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来指定回收计划，可以自由选择任意多个Region构成会收集，然后把决定回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧的Region的全部空间。这里的操作涉及存活对象的移动，需要STW。
![](G1垃圾收集流程.jpg)
#### G1的三种回收类型
![](G1回收过程.jpg)
* 当年轻代的Eden区用尽时开始年轻代回收过程（S区做被动回收）
* 当堆内存使用达到一定值（默认45%）时，开始老年代并发标记过程
* 标记完成马上进行混合回收过程，年轻代和老年代Region一起回收的
#### 具体回收过程
* 年轻代GC（STW）
	* 扫描GC Roots和Rset
	* 更新Rset、处理Rset
	* 复制回收
	* 处理引用
* 年轻代并发标记
	* 初始标记（STW）
	* 根区域扫描：扫描标记S区引用的老年代对象，YGC之前完成
	* 并发标记
	* 再次标记
	* 独占清理（STW）:根据价值标记
	* 并发清理
* 混合回收（Mixed GC）
	* Mixed GC 不是 Full GC
* Full GC
	* STW，单线程回收
	* 堆内存过小会导致Full GC
	* 没有足够空间放晋升对象会导致Full GC
	* 并发处理时内存耗尽会导致Full GC
#### 三色标记
### 什么是记忆集和卡表？::
记忆集可以缩小GC Roots的扫描范围。
#### 记忆集（Remembered Set）
一个Region可能引用别的Region的对象，那么回收对象的时候要去扫描其他Region的对象？比如，YGC的时候去老年代查找有对象引用年轻代的对象？这样会降低回收效率。

解决方法：
无论时G1还是其他分代收集器，JVM都是使用Remembered Set来避免全局扫描，每个Region都有一个对应的Remembered Set，每次引用类型数据写操作的时候，都会产生一个Write Barrier写屏障暂时中断操作，然后检查将要写入的引用指向的对象是否和该引用类型数据在不同的Region（其他分代收集器检查是不是跨代），如果不同，则通过CardTable卡表把引用信息记录到Region的Remembered Set中，当垃圾收集时，会同时查看Remembered Set，就可以保证不会全局扫描，也不会遗漏。
![](记忆集与卡表.jpg)
如上图，由于Region1和Region3引用了Region2中的对象，所以Region2的Rset种记录了是Region1和Region3对自己的对象进行了引用。

### G1收集器的设置建议?::
* 年轻代大小
	* 避免使用-Xmn或-XX:NewRatio等相关选项显式设置年轻代大小
	* 年轻代大小设置不合理可能会达不到暂停目标，推荐让G1自己动态设置
* 暂停时间目标不要太过苛刻
* G1 GC的吞吐量目标是90%的应用程序时间和10%的垃圾收集时间
* 评估G1 GC的吞吐量时，暂停时间目标不要太严苛。目标太过严苛表示承受更多的垃圾回收开销，影响吞吐量
### GC日志分析
![](yonggc日志.jpg)
![](fullgc日志.jpg)
TODO
### Full GC和Young GC有什么区别？::
1. 范围：
>Young GC只针对年轻代进行垃圾回收，而Full GC则对整个堆（包括年轻代和老年代）进行垃圾回收。
2. 触发条件：
>Young GC通常由Eden区满、Survivor区不足等触发，表示年轻代垃圾回收。而Full GC通常由老年代空间不足、永久代/元空间不足等触发，表示整个堆的垃圾回收。
3. 执行时间：
>Young GC通常比Full GC执行时间更短，因为它只需要处理年轻代的对象，而年轻代的大小相对较小。Full GC需要扫描整个堆，包括大量的老年代对象，因此通常需要更长的执行时间。
4. 垃圾回收算法：
>Young GC使用的是快速且高效的复制算法（Copying Algorithm），通常是使用的是标记-复制（Mark-Copy）算法或者标记-整理（Mark-Sweep-Compact）算法。Full GC则可能使用更为复杂的算法，如标记-清除（Mark-Sweep）、标记-整理（Mark-Sweep-Compact）等。
5. 停顿时间：
>Young GC通常会引起较短的暂停，因为只涉及年轻代的对象，可以并行或并发进行垃圾回收。Full GC通常会引起较长的停顿，因为需要扫描整个堆，可能会导致较长的暂停时间。
6. 影响范围：
>Young GC主要影响应用程序的响应时间，因为它通常会在应用程序运行期间进行。Full GC则会导致更长的停顿时间，可能会明显影响应用程序的性能和响应性。
### 垃圾回收主要是针对哪些区域？::
1. 年轻代垃圾回收：
>年轻代使用的是基于复制（Copying）算法的垃圾回收。通常情况下，当Eden区满时，会触发Young GC，将存活的对象拷贝到Survivor区，并清空Eden区和From Survivor区。经过多次Young GC后，仍然存活的对象会被晋升到老年代。
2. 老年代垃圾回收：
>老年代中的垃圾回收算法主要有标记-清除（Mark-Sweep）、标记-整理（Mark-Sweep-Compact）等。这些算法会标记并清理不再被引用的对象，或者将存活的对象向一端移动，从而形成连续的内存空间。
3. 方法区/元空间垃圾回收：
>在方法区/元空间中，主要进行的是对无用的类和常量的回收。对于已经加载的类，当该类的所有实例都不再存活时，类的元数据信息可能会被回收。

### FULL GC频繁可能是JVM哪些区域发生了问题？如何排查？::
1. 老年代空间不足：
>如果老年代空间不足，即老年代中的对象占用的内存超过了老年代的容量，就会频繁触发Full GC。这可能是因为应用程序中存在大量的长生命周期对象，导致老年代的内存快速耗尽。

2. 永久代/元空间不足：
>在Java 8之前的版本中，永久代（Permanent Generation）用于存储类的元数据信息。而在Java 8及以后的版本中，元空间（Metaspace）取代了永久代。如果应用程序中使用了大量的类或者动态生成类，会导致永久代/元空间不足，从而频繁触发Full GC。

3. 内存泄漏：
>内存泄漏是指应用程序中的对象不再使用，但由于某些原因没有被垃圾回收，仍然占用着内存。如果存在大量的内存泄漏对象，堆中的空间会被迅速耗尽，导致频繁的Full GC。

4. 垃圾回收策略不合适：
> 垃圾回收策略的选择和调优对于应用程序的性能至关重要。如果选择的垃圾回收器或者垃圾回收策略不合适，可能导致频繁的Full GC。例如，如果选择了不适合应用程序特征的垃圾回收器，或者设置了不合理的垃圾回收参数，都可能导致Full GC频繁发生。

#### 如何排查？
* 分析GC日志：
>查看GC日志，了解Full GC的原因、频率和持续时间，以及发生GC的区域。
* 分析堆内存使用情况：
>查看堆内存的使用情况，特别是老年代的使用情况。观察是否有内存泄漏或者对象占用过多内存的情况。
* 优化内存分配和回收策略：
>根据应用程序的特点和需求，调整垃圾回收器的选择、垃圾回收参数的设置，以及内存分配策略，以减少Full GC的发生。
* 检查代码和资源管理：
>检查应用程序的代码，特别是对资源的使用和释放是否正确。确保及时释放不再使用的对象，避免内存泄漏。
* 调整堆大小：
>如果确定堆内存不足导致频繁的Full GC，可以考虑增加堆的大小，以提供更多的内存空间。
## 执行引擎主要做了什么？::
- Java字节码的执行过程  
- 编译期优化  
- 运行期优化
## 编译期优化主要有哪些内容？::
* **抽象语法树优化**：编译器会对源代码进行解析并构建抽象语法树（AST），然后进行优化，如常量折叠、公共子表达式消除、无用代码删除等。
* **静态类型检查**：编译器会进行静态类型检查，确保代码中的类型匹配和类型安全性。
* **控制流分析**：编译器通过控制流分析来理解程序的控制流程，以便进行后续的优化和转换。
* **内联函数**：编译器可以将一些小的方法调用直接替换为具体的方法体，减少方法调用带来的开销。
## 运行期优化主要有哪些内容？::
- **即时编译（JIT）**：JVM的执行引擎中的即时编译器会动态将热点代码（hotspot）转换成本地机器码，以提高执行速度。
- **逃逸分析**：通过逃逸分析技术，执行引擎可以确定对象是否逃逸出方法的作用域，从而进行一些针对性的优化，如栈上分配、标量替换等。
- **方法内联**：执行引擎会尝试将方法调用替换为直接的方法体，减少方法调用的开销。
# 本地方法接口
TODO
# 本地方法库
TODO

# JVM调优
## 如何进行性能优化？::
* [性能监控](#性能监控（发现问题）)
* [性能分析](#性能分析（排查问题）)
* [性能调优](#性能调优（解决问题）)
### 性能监控（发现问题）
- GC次数、时间监控
- cpu 负载监控
- 内存占用监控
- OOM、内存泄露
- 死锁
- 程序响应时间较长
### 性能分析（排查问题）
- 打印GC日志，通过GCviewer或者 [http://gceasy.io](http://gceasy.io) 来分析异常信息
- 灵活运用命令行工具、jstack、jmap、jinfo等
- dump出堆文件，使用内存分析工具分析文件
- 使用阿里Arthas、jconsole、JVisualVM来实时查看JVM状态
- jstack查看堆栈信息
### 性能调优（解决问题）
- 适当增加内存，根据业务背景选择垃圾回收器
- 优化代码，控制内存使用
- 增加机器，分散节点压力
- 合理设置线程池线程数量
- 使用中间件提高程序效率，比如缓存、消息队列等

## 性能评价/测试指标
* **[停顿时间](#停顿时间（或响应时间）)**
* **[吞吐量](#吞吐量 )**
* [并发数](#并发数)
* [内存占用](#内存占用)
### 停顿时间（或响应时间）
提交请求和返回该请求的响应之间使用的时间，一般比较关注平均响应时间。常用操作的响应时间列表：

|操作|响应时间|
|---|---|
|打开一个站点|几秒|
|数据库查询一条记录（有索引）|十几毫秒|
|机械磁盘一次寻址定位|4毫秒|
|从机械磁盘顺序读取1M数据|2毫秒|
|从SSD磁盘顺序读取1M数据|0.3毫秒|
|从远程分布式换成Redis 读取一个数据|0.5毫秒|
|从内存读取 1M数据|十几微妙|
|Java程序本地方法调用|几微妙|
|网络传输2Kb数据|1 微妙|


在垃圾回收环节中：
- 暂停时间：执行垃圾收集时，程序的工作线程被暂停的时间。
- -XX:MaxGCPauseMillis

### 吞吐量
- 对单位时间内完成的工作量（请求）的量度
- 在GC中：运行用户代码的事件占总运行时间的比例（总运行时间：程序的运行时间+内存回收的时间）
- 吞吐量为1-1/(1+n)，其中-XX::GCTimeRatio=n

### 并发数
- 同一时刻，对服务器有实际交互的请求数

### 内存占用
- Java堆区所占的内存大小
### 相互间的关系
以高速公路通行状况为例
- 吞吐量：每天通过高速公路收费站的车辆的数据
- 并发数：高速公路上正在行驶的车辆的数目
- 响应时间：车速
## 怎么选择垃圾回收器？
- 优先让JVM自适应，调整堆的大小
- 串行收集器：内存小于100M；单核、单机程序，并且没有停顿时间的要求
- 并行收集器：多CPU、高吞吐量、允许停顿时间超过1秒
- 并发收集器：多CPU、追求低停顿时间、快速响应（比如延迟不能超过1秒，如互联网应用）
- 官方推荐G1，性能高。现在互联网的项目，基本都是使用G1

特别说明：
- 没有最好的收集器，更没有万能的收集器
- 调优永远是针对特定场景、特定需求，不存在一劳永逸的收集器
## 有哪些常用的jvm调优参数？::

官网地址：[https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html)

* -Xms
* -Xmx
* -Xss
* -XX:ParallelGCThreads ：限制新生代垃圾回收线程数量，默认开启CPU小于8时和CPU核数相同，大于8时，`ParallelGCThreads = 3+[5 * 核数/8]`
* -XX:+PrintCommandLineFlags 程序运行时JVM默认设置或用户手动设置的XX选项
* -XX:+PrintFlagsInitial 打印所有XX选项的默认值
* -XX:+PrintFlagsFinal 打印所有XX选项的实际值
* -XX:+PrintVMOptions 打印JVM的参数

#### 堆、栈、方法区等内存大小设置
```shell
# 栈
-Xss128k <==> -XX:ThreadStackSize=128k 设置线程栈的大小为128K

# 堆
-Xms2048m <==> -XX:InitialHeapSize=2048m 设置JVM初始堆内存为2048M
-Xmx2048m <==> -XX:MaxHeapSize=2048m 设置JVM最大堆内存为2048M
-Xmn2g <==> -XX:NewSize=2g -XX:MaxNewSize=2g 设置年轻代大小为2G
-XX:SurvivorRatio=8 设置Eden区与Survivor区的比值，默认为8
-XX:NewRatio=2 设置老年代与年轻代的比例，默认为2
-XX:+UseAdaptiveSizePolicy 设置大小比例自适应，默认开启
-XX:PretenureSizeThreadshold=1024 设置让大于此阈值的对象直接分配在老年代，只对Serial、ParNew收集器有效
-XX:MaxTenuringThreshold=15 设置新生代晋升老年代的年龄限制，默认为15
-XX:TargetSurvivorRatio 设置MinorGC结束后Survivor区占用空间的期望比例

# 方法区
-XX:MetaspaceSize / -XX:PermSize=256m 设置元空间/永久代初始值为256M
-XX:MaxMetaspaceSize / -XX:MaxPermSize=256m 设置元空间/永久代最大值为256M
-XX:+UseCompressedOops 使用压缩对象
-XX:+UseCompressedClassPointers 使用压缩类指针
-XX:CompressedClassSpaceSize 设置Klass Metaspace的大小，默认1G

# 直接内存
-XX:MaxDirectMemorySize 指定DirectMemory容量，默认等于Java堆最大值
```
#### OutOfMemory相关的选项
```shell
-XX:+HeapDumpOnOutMemoryError 内存出现OOM时生成Heap转储文件，两者互斥
-XX:+HeapDumpBeforeFullGC 出现FullGC时生成Heap转储文件，两者互斥
-XX:HeapDumpPath=<path> 指定heap转储文件的存储路径，默认当前目录
-XX:OnOutOfMemoryError=<path> 指定可行性程序或脚本的路径，当发生OOM时执行脚本
```
#### 垃圾收集器相关选项
##### Serial回收器参数设置
```shell
# Serial回收器
-XX:+UseSerialGC  年轻代使用Serial GC， 老年代使用Serial Old GC
``` 
##### ParNew参数设置
```shell
# ParNew回收器
-XX:+UseParNewGC  年轻代使用ParNew GC
-XX:ParallelGCThreads  设置年轻代并行收集器的线程数。
	一般地，最好与CPU数量相等，以避免过多的线程数影响垃圾收集性能。
```
##### Parallel参数设置
```shell
# Parallel回收器
-XX:+UseParallelGC  年轻代使用 Parallel Scavenge GC，互相激活
-XX:+UseParallelOldGC  老年代使用 Parallel Old GC，互相激活
-XX:ParallelGCThreads
-XX:MaxGCPauseMillis  设置垃圾收集器最大停顿时间（即STW的时间），单位是毫秒。
	为了尽可能地把停顿时间控制在MaxGCPauseMills以内，收集器在工作时会调整Java堆大小或者其他一些参数。
	对于用户来讲，停顿时间越短体验越好；但是服务器端注重高并发，整体的吞吐量。
	所以服务器端适合Parallel，进行控制。该参数使用需谨慎。
-XX:GCTimeRatio  垃圾收集时间占总时间的比例（1 / (N＋1)），用于衡量吞吐量的大小
	取值范围（0,100），默认值99，也就是垃圾回收时间不超过1％。
	与前一个-XX：MaxGCPauseMillis参数有一定矛盾性。暂停时间越长，Radio参数就容易超过设定的比例。
-XX:+UseAdaptiveSizePolicy  设置Parallel Scavenge收集器具有自适应调节策略。
	在这种模式下，年轻代的大小、Eden和Survivor的比例、晋升老年代的对象年龄等参数会被自动调整，以达到在堆大小、吞吐量和停顿时间之间的平衡点。
	在手动调优比较困难的场合，可以直接使用这种自适应的方式，仅指定虚拟机的最大堆、目标的吞吐量（GCTimeRatio）和停顿时间（MaxGCPauseMills），让虚拟机自己完成调优工作。
```
##### CMS参数设置
```shell
# CMS回收器
-XX:+UseConcMarkSweepGC  年轻代使用CMS GC。
	开启该参数后会自动将-XX：＋UseParNewGC打开。即：ParNew（Young区）+ CMS（Old区）+ Serial Old的组合
-XX:CMSInitiatingOccupanyFraction  设置堆内存使用率的阈值，一旦达到该阈值，便开始进行回收。JDK5及以前版本的默认值为68，DK6及以上版本默认值为92％。
	如果内存增长缓慢，则可以设置一个稍大的值，大的阈值可以有效降低CMS的触发频率，减少老年代回收的次数可以较为明显地改善应用程序性能。
	反之，如果应用程序内存使用率增长很快，则应该降低这个阈值，以避免频繁触发老年代串行收集器。
	因此通过该选项便可以有效降低Fu1l GC的执行次数。
-XX:+UseCMSInitiatingOccupancyOnly  是否动态可调，使CMS一直按CMSInitiatingOccupancyFraction设定的值启动
-XX:+UseCMSCompactAtFullCollection  用于指定在执行完Full GC后对内存空间进行压缩整理
	以此避免内存碎片的产生。不过由于内存压缩整理过程无法并发执行，所带来的问题就是停顿时间变得更长了。
-XX:CMSFullGCsBeforeCompaction  设置在执行多少次Full GC后对内存空间进行压缩整理。
-XX:ParallelCMSThreads  设置CMS的线程数量。
	CMS 默认启动的线程数是(ParallelGCThreads＋3)/4，ParallelGCThreads 是年轻代并行收集器的线程数。
	当CPU 资源比较紧张时，受到CMS收集器线程的影响，应用程序的性能在垃圾回收阶段可能会非常糟糕。
-XX:ConcGCThreads  设置并发垃圾收集的线程数，默认该值是基于ParallelGCThreads计算出来的
-XX:+CMSScavengeBeforeRemark  强制hotspot在cms remark阶段之前做一次minor gc，用于提高remark阶段的速度
-XX:+CMSClassUnloadingEnable  如果有的话，启用回收Perm 区（JDK8之前）
-XX:+CMSParallelInitialEnabled  用于开启CMS initial-mark阶段采用多线程的方式进行标记
	用于提高标记速度，在Java8开始已经默认开启
-XX:+CMSParallelRemarkEnabled  用户开启CMS remark阶段采用多线程的方式进行重新标记，默认开启
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses
	这两个参数用户指定hotspot虚拟在执行System.gc()时使用CMS周期
-XX:+CMSPrecleaningEnabled  指定CMS是否需要进行Pre cleaning阶段
```
##### G1参数设置
```shell
# G1回收器
-XX:+UseG1GC 手动指定使用G1收集器执行内存回收任务。
-XX:G1HeapRegionSize 设置每个Region的大小。
	值是2的幂，范围是1MB到32MB之间，目标是根据最小的Java堆大小划分出约2048个区域。默认是堆内存的1/2000。
-XX:MaxGCPauseMillis  设置期望达到的最大GC停顿时间指标（JVM会尽力实现，但不保证达到）。默认值是200ms
-XX:ParallelGCThread  设置STW时GC线程数的值。最多设置为8
-XX:ConcGCThreads  设置并发标记的线程数。将n设置为并行垃圾回收线程数（ParallelGCThreads）的1/4左右。
-XX:InitiatingHeapOccupancyPercent 设置触发并发GC周期的Java堆占用率阈值。超过此值，就触发GC。默认值是45。
-XX:G1NewSizePercent  新生代占用整个堆内存的最小百分比（默认5％）
-XX:G1MaxNewSizePercent  新生代占用整个堆内存的最大百分比（默认60％）
-XX:G1ReservePercent=10  保留内存区域，防止 to space（Survivor中的to区）溢出
```

#### GC日志相关选项
```shell
-XX:+PrintGC <==> -verbose:gc  打印简要日志信息
-XX:+PrintGCDetails            打印详细日志信息
-XX:+PrintGCTimeStamps  打印程序启动到GC发生的时间，搭配-XX:+PrintGCDetails使用
-XX:+PrintGCDateStamps  打印GC发生时的时间戳，搭配-XX:+PrintGCDetails使用
-XX:+PrintHeapAtGC  打印GC前后的堆信息，如下图
-Xloggc:<file> 输出GC导指定路径下的文件中
```

#### 其他参数
```shell
-XX:+DisableExplicitGC  禁用hotspot执行System.gc()，默认禁用
-XX:ReservedCodeCacheSize=<n>[g|m|k]、-XX:InitialCodeCacheSize=<n>[g|m|k]  指定代码缓存的大小
-XX:+UseCodeCacheFlushing  放弃一些被编译的代码，避免代码缓存被占满时JVM切换到interpreted-only的情况
-XX:+DoEscapeAnalysis  开启逃逸分析
-XX:+UseBiasedLocking  开启偏向锁
-XX:+UseLargePages  开启使用大页面
-XX:+PrintTLAB  打印TLAB的使用情况
-XX:TLABSize  设置TLAB大小
```

####  通过Java代码获取JVM参数
Java提供了java.lang.management包用于监视和管理Java虚拟机和Java运行时中的其他组件，它允许本地或远程监控和管理运行的Java虚拟机。其中ManagementFactory类较为常用，另外Runtime类可获取内存、CPU核数等相关的数据。通过使用这些api，可以监控应用服务器的堆内存使用情况，设置一些阈值进行报警等处理。
```java
public class MemoryMonitor {
    public static void main(String[] args) {
        MemoryMXBean memorymbean = ManagementFactory.getMemoryMXBean();
        MemoryUsage usage = memorymbean.getHeapMemoryUsage();
        System.out.println("INIT HEAP: " + usage.getInit() / 1024 / 1024 + "m");
        System.out.println("MAX HEAP: " + usage.getMax() / 1024 / 1024 + "m");
        System.out.println("USE HEAP: " + usage.getUsed() / 1024 / 1024 + "m");
        System.out.println("\nFull Information:");
        System.out.println("Heap Memory Usage: " + memorymbean.getHeapMemoryUsage());
        System.out.println("Non-Heap Memory Usage: " + memorymbean.getNonHeapMemoryUsage());

        System.out.println("=======================通过java来获取相关系统状态============================ ");
        System.out.println("当前堆内存大小totalMemory " + (int) Runtime.getRuntime().totalMemory() / 1024 / 1024 + "m");// 当前堆内存大小
        System.out.println("空闲堆内存大小freeMemory " + (int) Runtime.getRuntime().freeMemory() / 1024 / 1024 + "m");// 空闲堆内存大小
        System.out.println("最大可用总堆内存maxMemory " + Runtime.getRuntime().maxMemory() / 1024 / 1024 + "m");// 最大可用总堆内存大小

    }
}
```

### G1相关参数
* -XX:+UseG1GC
* -XX:G1HeapRegionSize
	* 设置每个Region的大小，值是2的幂。范围是1MB到32MB之间，目标是根据最小的java堆大小划分出2048个区域。默认是堆内存的1/2000。最大管理64G堆内存。
* -XX:MaxGCPauseMillis
	* 设置期望达到的最大GC停顿时间指标（JVM会尽力实现，但不保证一定到达）默认200ms
	* **可以根据项目的95线设置**
* -XX:ParallelGCThreads
	* 设置STW时GC线程数，最多设置为8
* -XX:ConcGCThreads
	* 设置并发标记的线程数设置为ParallelGCThreads的1/4左右，超过ParallelGCThreads会报错
* -XX:InitiatingHeapOccupancyPercent
	* 设置触发并发GC周期的java堆占用率阈值。超过此值就会GC。默认45。
## 有哪些常用的jvm命令？::
* jps
* jstat（重要）
* jinfo
* jmap
* jstak（重要）
* jmap
* jcmd
* jstatd
### jps
jps(Java Process Status)：显示指定系统内所有的HotSpot虚拟机进程（查看虚拟机进程信息），可用于查询正在运行的虚拟机进程。

**options参数**
- -q：仅仅显示LVMID（local virtual machine id），即本地虚拟机唯一id。不显示主类的名称等
- -l：输出应用程序主类的全类名 或 如果进程执行的是jar包，则输出jar完整路径
- -m：输出虚拟机进程启动时传递给主类main()的参数
- -v：列出虚拟机进程启动时的JVM参数。比如：-Xms20m -Xmx50m是启动程序指定的jvm参数。

补充：如果某 Java 进程关闭了默认开启的UsePerfData参数（即使用参数-XX：-UsePerfData），那么jps命令（以及下面介绍的jstat）将无法探知该Java 进程。

### jstat（重要）
jstat（JVM Statistics Monitoring Tool）：用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。在没有GUI图形界面，只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的首选工具。常用于检测垃圾回收问题以及内存泄漏问题。

官方文档：[https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)

基本使用语法为：`jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]`
* **interval参数：** 用于指定输出统计数据的周期，单位为毫秒。即：查询间隔
* **count参数：** 用于指定查询的总次数
* **-t参数：** 可以在输出信息前加上一个Timestamp列，显示程序的运行时间。单位：秒
* **-h参数：** 可以在周期性数据输出时，输出多少行数据后输出一个表头信息

#### option参数

* 类装载相关的：
	- -class：显示ClassLoader的相关信息：类的装载、卸载数量、总空间、类装载所消耗的时间等
* 垃圾回收相关的：
	- -gc：显示与GC相关的堆信息。包括Eden区、两个Survivor区、老年代、永久代等的容量、已用空间、GC时间合计等信息。
	- -gccapacity：显示内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间。
	- -gcutil：显示内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比。
	- -gccause：与-gcutil功能一样，但是会额外输出导致最后一次或当前正在发生的GC产生的原因。
	- -gcnew：显示新生代GC状况
	- -gcnewcapacity：显示内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间
	- -geold：显示老年代GC状况
	- -gcoldcapacity：显示内容与-gcold基本相同，输出主要关注使用到的最大、最小空间
	- -gcpermcapacity：显示永久代使用到的最大、最小空间。
* JIT相关的：
	- -compiler：显示JIT编译器编译过的方法、耗时等信息
	- -printcompilation：输出已经被JIT编译的方法

####  jstat判断是否出现内存泄漏

* 第1步：在长时间运行的 Java 程序中，我们可以运行jstat命令连续获取多行性能数据，并取这几行数据中 OU 列（即已占用的老年代内存）的最小值。
* 第2步：然后，我们每隔一段较长的时间重复一次上述操作，来获得多组 OU 最小值。如果这些值呈上涨趋势，则说明该 Java 程序的老年代内存已使用量在不断上涨，这意味着无法回收的对象在不断增加，因此很有可能存在内存泄漏。
### jinfo
jinfo(Configuration Info for Java)：查看虚拟机配置参数信息，也可用于调整虚拟机的配置参数。在很多情况下，Java应用程序不会指定所有的Java虚拟机参数。而此时，开发人员可能不知道某一个具体的Java虚拟机参数的默认值。在这种情况下，可能需要通过查找文档获取某个参数的默认值。这个查找过程可能是非常艰难的。但有了jinfo工具，开发人员可以很方便地找到Java虚拟机参数的当前值。

基本使用语法为：`jinfo [options] pid`

* `jinfo -sysprops`
* `jinfo -flags pid`
* `jinfo -flag`

### jmap
jmap（JVM Memory Map）：作用一方面是获取dump文件（堆转储快照文件，二进制文件），它还可以获取目标Java进程的内存相关信息，包括Java堆各区域的使用情况、堆中对象的统计信息、类加载信息等。开发人员可以在控制台中输入命令“jmap -help”查阅jmap工具的具体使用方式和一些标准选项配置。

官方帮助文档：[https://docs.oracle.com/en/java/javase/11/tools/jmap.html](https://docs.oracle.com/en/java/javase/11/tools/jmap.html)

基本使用语法为：

- `jmap [option] <pid>`
- `jmap [option] <executable <core>`
- `jmap [option] [server_id@] <remote server IP or hostname>`

|选项|作用|
|---|---|
|**-dump**|生成dump文件（Java堆转储快照），`-dump:live`只保存堆中的存活对象|
|**-heap**|输出整个堆空间的详细信息，包括GC的使用、堆配置信息，以及内存的使用信息等|
|**-histo**|输出堆空间中对象的统计信息，包括类、实例数量和合计容量，`-histo:live`只统计堆中的存活对象|
|`-J <flag>`|传递参数给jmap启动的jvm|
|`-finalizerinfo`|显示在F-Queue中等待Finalizer线程执行finalize方法的对象，仅linux/solaris平台有效|
|`-permstat`|以ClassLoader为统计口径输出永久代的内存状态信息，仅linux/solaris平台有效|
|`-F`|当虚拟机进程对-dump选项没有任何响应时，强制执行生成dump文件，仅linux/solaris平台有效|

说明：这些参数和linux下输入显示的命令多少会有不同，包括也受jdk版本的影响。
```bash
> jmap -dump:format=b,file=<filename.hprof> <pid>
> jmap -dump:live,format=b,file=<filename.hprof> <pid>
```

由于jmap将访问堆中的所有对象，为了保证在此过程中不被应用线程干扰，jmap需要借助安全点机制，让所有线程停留在不改变堆中数据的状态。也就是说，由jmap导出的堆快照必定是安全点位置的。这可能导致基于该堆快照的分析结果存在偏差。

举个例子，假设在编译生成的机器码中，某些对象的生命周期在两个安全点之间，那么:live选项将无法探知到这些对象。

另外，如果某个线程长时间无法跑到安全点，jmap将一直等下去。与前面讲的jstat则不同，垃圾回收器会主动将jstat所需要的摘要数据保存至固定位置之中，而jstat只需直接读取即可。

### jstack（重要）

jstack（JVM Stack Trace）：用于生成虚拟机指定进程当前时刻的线程快照（虚拟机堆栈跟踪）。线程快照就是当前虚拟机内指定进程的每一条线程正在执行的方法堆栈的集合。

生成线程快照的作用：可用于定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等问题。这些都是导致线程长时间停顿的常见原因。当线程出现停顿时，就可以用jstack显示各个线程调用的堆栈情况。

官方帮助文档：[https://docs.oracle.com/en/java/javase/11/tools/jstack.html](https://docs.oracle.com/en/java/javase/11/tools/jstack.html)

在thread dump中，要留意下面几种状态
- **死锁，Deadlock（重点关注）**
- **等待资源，Waiting on condition（重点关注）**
- **等待获取监视器，Waiting on monitor entry（重点关注）**
- **阻塞，Blocked（重点关注）**
- 执行中，Runnable
- 暂停，Suspended
- 对象等待中，Object.wait() 或 TIMED＿WAITING
- 停止，Parked

|option参数|作用|
|---|---|
|-F|当正常输出的请求不被响应时，强制输出线程堆栈|
|-l|除堆栈外，显示关于锁的附加信息|
|-m|如果调用本地方法的话，可以显示C/C++的堆栈|

### jcmd
在JDK 1.7以后，新增了一个命令行工具jcmd。它是一个多功能的工具，可以用来实现前面除了jstat之外所有命令的功能。比如：用它来导出堆、内存使用、查看Java进程、导出线程信息、执行GC、JVM运行时间等。

官方帮助文档：[https://docs.oracle.com/en/java/javase/11/tools/jcmd.html](https://docs.oracle.com/en/java/javase/11/tools/jcmd.html)

jcmd拥有jmap的大部分功能，并且在Oracle的官方网站上也推荐使用jcmd命令代jmap命令。

* `jcmd -l`：列出所有的JVM进程
* `jcmd 进程号 help`：针对指定的进程，列出支持的所有具体命令
* `jcmd 进程号 具体命令`：显示指定进程的指令命令的数据
	- Thread.print 可以替换 jstack指令
	- GC.class_histogram 可以替换 jmap中的-histo操作
	- GC.heap_dump 可以替换 jmap中的-dump操作
	- GC.run 可以查看GC的执行情况
	- VM.uptime 可以查看程序的总执行时间，可以替换jstat指令中的-t操作
	- VM.system_properties 可以替换 jinfo -sysprops 进程id
	- VM.flags 可以获取JVM的配置参数信息

### jstatd
之前的指令只涉及到监控本机的Java应用程序，而在这些工具中，一些监控工具也支持对远程计算机的监控（如jps、jstat）。为了启用远程监控，则需要配合使用jstatd 工具。命令jstatd是一个RMI服务端程序，它的作用相当于代理服务器，建立本地计算机与远程监控工具的通信。jstatd服务器将本机的Java应用程序信息传递到远程计算机。

## 有哪些常用的调优工具？::
* JConsole
* Visual VM
* Eclipse MAT
* JProfiler
* Arthas
* Java Misssion Control
* 其他工具
	* Flame Graphs（火焰图）
	* Tprofiler
	* Btrace
	* YourKit
	* JProbe
	* Spring Insight
### JProfiler
官网地址：[https://www.ej-technologies.com/products/jprofiler/overview.html](https://www.ej-technologies.com/products/jprofiler/overview.html)

#### 数据采集方式：

JProfier数据采集方式分为两种：Sampling（样本采集）和Instrumentation（重构模式）

* ***Instrumentation**：这是JProfiler全功能模式。在class加载之前，JProfier把相关功能代码写入到需要分析的class的bytecode中，对正在运行的jvm有一定影响。
	- 优点：功能强大。在此设置中，调用堆栈信息是准确的。
	- 缺点：若要分析的class较多，则对应用的性能影响较大，CPU开销可能很高（取决于Filter的控制）。因此使用此模式一般配合Filter使用，只对特定的类或包进行分析
- **Sampling**：类似于样本统计，每隔一定时间（5ms）将每个线程栈中方法栈中的信息统计出来。
	- 优点：对CPU的开销非常低，对应用影响小（即使你不配置任何Filter）
	- 缺点：一些数据／特性不能提供（例如：方法的调用次数、执行时间）

注：JProfiler本身没有指出数据的采集类型，这里的采集类型是针对方法调用的采集类型。因为JProfiler的绝大多数核心功能都依赖方法调用采集的数据，所以可以直接认为是JProfiler的数据采集类型。
#### 内存视图 Live Memory

Live memory 内存剖析：class／class instance的相关信息。例如对象的个数，大小，对象创建的方法执行栈，对象创建的热点。

- **所有对象 All Objects**：显示所有加载的类的列表和在堆上分配的实例数。只有Java 1.5（JVMTI）才会显示此视图。
- **记录对象 Record Objects**：查看特定时间段对象的分配，并记录分配的调用堆栈。
- **分配访问树 Allocation Call Tree**：显示一棵请求树或者方法、类、包或对已选择类有带注释的分配信息的J2EE组件。
- **分配热点 Allocation Hot Spots**：显示一个列表，包括方法、类、包或分配已选类的J2EE组件。你可以标注当前值并且显示差异值。对于每个热点都可以显示它的跟踪记录树。
- **类追踪器 Class Tracker**：类跟踪视图可以包含任意数量的图表，显示选定的类和包的实例与时间。

#### 堆遍历 heap walker
#### cpu视图 cpu views
JProfiler 提供不同的方法来记录访问树以优化性能和细节。线程或者线程组以及线程状况可以被所有的视图选择。所有的视图都可以聚集到方法、类、包或J2EE组件等不同层上。
- **访问树 Call Tree**：显示一个积累的自顶向下的树，树中包含所有在JVM中已记录的访问队列。JDBC，JMS和JNDI服务请求都被注释在请求树中。请求树可以根据Servlet和JSP对URL的不同需要进行拆分。
- **热点 Hot Spots**：显示消耗时间最多的方法的列表。对每个热点都能够显示回溯树。该热点可以按照方法请求，JDBC，JMS和JNDI服务请求以及按照URL请求来进行计算。
- **访问图 Call Graph**：显示一个从已选方法、类、包或J2EE组件开始的访问队列的图。
- **方法统计 Method Statistis**：显示一段时间内记录的方法的调用时间细节。
#### 线程视图 threads
JProfiler通过对线程历史的监控判断其运行状态，并监控是否有线程阻塞产生，还能将一个线程所管理的方法以树状形式呈现。对线程剖析。

- **线程历史 Thread History**：显示一个与线程活动和线程状态在一起的活动时间表。
- **线程监控 Thread Monitor**：显示一个列表，包括所有的活动线程以及它们目前的活动状况。
- **线程转储 Thread Dumps**：显示所有线程的堆栈跟踪。

线程分析主要关心三个方面：
- 1．web容器的线程最大数。比如：Tomcat的线程容量应该略大于最大并发数。
- 2．线程阻塞
- 3．线程死锁
#### 监控和锁 Monitors ＆Locks

所有线程持有锁的情况以及锁的信息。观察JVM的内部线程并查看状态：

- **死锁探测图表 Current Locking Graph**：显示JVM中的当前死锁图表。
- **目前使用的监测器 Current Monitors**：显示目前使用的监测器并且包括它们的关联线程。
- **锁定历史图表 Locking History Graph**：显示记录在JVM中的锁定历史。
- **历史检测记录 Monitor History**：显示重大的等待事件和阻塞事件的历史记录。
- **监控器使用统计 Monitor Usage Statistics**：显示分组监测，线程和监测类的统计监测数据

### Arthas

上述工具都必须在服务端项目进程中配置相关的监控参数，然后工具通过远程连接到项目进程，获取相关的数据。这样就会带来一些不便，比如线上环境的网络是隔离的，本地的监控工具根本连不上线上环境。并且类似于Jprofiler这样的商业工具，是需要付费的。

  

那么有没有一款工具不需要远程连接，也不需要配置监控参数，同时也提供了丰富的性能监控数据呢？

  

阿里巴巴开源的性能分析神器Arthas应运而生。

  

Arthas是Alibaba开源的Java诊断工具，深受开发者喜爱。在线排查问题，无需重启；动态跟踪Java代码；实时监控JVM状态。Arthas 支持JDK 6＋，支持Linux／Mac／Windows，采用命令行交互模式，同时提供丰富的 Tab 自动补全功能，进一步方便进行问题的定位和诊断。当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：

  

- 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？

- 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？

- 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？

- 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！

- 是否有一个全局视角来查看系统的运行状况？

- 有什么办法可以监控到JVM的实时运行状态？

- 怎么快速定位应用的热点，生成火焰图？

  

官方地址：[https://arthas.aliyun.com/doc/quick-start.html](https://arthas.aliyun.com/doc/quick-start.html)

  

安装方式：如果速度较慢，可以尝试国内的码云Gitee下载。

```bash
wget https://io/arthas/arthas-boot.jar
wget https://arthas/gitee/io/arthas-boot.jar
```
Arthas只是一个java程序，所以可以直接用java -jar运行。  
  
除了在命令行查看外，Arthas目前还支持 Web Console。在成功启动连接进程之后就已经自动启动,可以直接访问 [http://127.0.0.1:8563/](http://127.0.0.1:8563/) 访问，页面上的操作模式和控制台完全一样。  
  
#### 基础指令
* quit/exit 
	>退出当前 Arthas客户端，其他 Arthas喜户端不受影响
* stop/shutdown 
	>关闭 Arthas服务端，所有 Arthas客户端全部退出
* help 
	>查看命令帮助信息
* cat 
	>打印文件内容，和linux里的cat命令类似
* echo 
	>打印参数，和linux里的echo命令类似
* grep 
	>匹配查找，和linux里的gep命令类似
* tee 
	>复制标隹输入到标准输出和指定的文件，和linux里的tee命令类似
* pwd 
	>返回当前的工作目录，和linux命令类似
* cls 
	>清空当前屏幕区域
* session 
	>查看当前会话的信息
* reset 
	>重置增强类，将被 Arthas增强过的类全部还原, Arthas服务端关闭时会重置所有增强过的类
* version 
	>输出当前目标Java进程所加载的 Arthas版本号
* history 
	>打印命令历史
* keymap 
	>Arthas快捷键列表及自定义快捷键
#### jvm相关
* dashboard 
	>当前系统的实时数据面板
* thread 
	>查看当前JVM的线程堆栈信息
* jvm 
	>查看当前JVM的信息
* sysprop 
	>查看和修改JVM的系统属性
* sysem 
	>查看JVM的环境变量
* vmoption 
	>查看和修改JVM里诊断相关的option
* perfcounter 
	>查看当前JVM的 Perf Counter信息
* logger 
	>查看和修改logger
* getstatic 
	>查看类的静态属性
* ognl 
	>执行ognl表达式
* mbean 
	>查看 Mbean的信息
* heapdump 
	>dump java heap，类似jmap命令的 heap dump功能
#### class/classloader相关
* sc 
	>查看JVM已加载的类信息
	* -d 输出当前类的详细信息，包括这个类所加载的原始文件来源、类的声明、加载的Classloader等详细信息。如果一个类被多个Classloader所加载，则会出现多次
	* -E 开启正则表达式匹配，默认为通配符匹配
	* -f 输出当前类的成员变量信息（需要配合参数-d一起使用）
	* -X 指定输出静态变量时属性的遍历深度，默认为0，即直接使用toString输出
* sm 
	>查看已加载类的方法信息
	* -d 展示每个方法的详细信息
	* -E 开启正则表达式匹配,默认为通配符匹配
* jad 
	>反编译指定已加载类的源码
* mc 
	>内存编译器，内存编译.java文件为.class文件
* retransform 
	>加载外部的.class文件, retransform到JVM里
* redefine 
	>加载外部的.class文件，redefine到JVM里
* dump 
	>dump已加载类的byte code到特定目录
* classloader 
	>查看classloader的继承树，urts，类加载信息，使用classloader
* getResource
	* -t 查看classloader的继承树
	* -l 按类加载实例查看统计信息
	* -c 用classloader对应的hashcode来查看对应的 Jar urls
#### monitor/watch/trace相关
* monitor 方法执行监控，调用次数、执行时间、失败率
	* -c 统计周期，默认值为120秒
* watch 方法执行观测，能观察到的范围为：返回值、抛出异常、入参，通过编写groovy表达式进行对应变量的查看
	* -b 在方法调用之前观察(默认关闭)
	* -e 在方法异常之后观察(默认关闭)
	* -s 在方法返回之后观察(默认关闭)
	* -f 在方法结束之后(正常返回和异常返回)观察(默认开启)
	* -x 指定输岀结果的属性遍历深度,默认为0
* trace 方法内部调用路径,并输出方法路径上的每个节点上耗时
	* -n 执行次数限制
* stack 输出当前方法被调用的调用路径
* tt 方法执行数据的时空隧道,记录下指定方法每次调用的入参和返回信息,并能对这些不同的时间下调用进行观测
#### 其他
* jobs 列出所有job
* kill 强制终止任务
* fg 将暂停的任务拉到前台执行
* bg 将暂停的任务放到后台执行
* grep 搜索满足条件的结果
* plaintext 将命令的结果去除ANSI颜色
* wc 按行统计输出结果
* options 查看或设置Arthas全局开关
* profiler 使用async-profiler对应用采样，生成火焰图
## 有哪些GC日志分析工具？::
### **GCEasy**
GCEasy是一款在线的GC日志分析器，可以通过GC日志分析进行内存泄露检测、GC暂停原因分析、JVM配置建议优化等功能，大多数功能是免费的。

官网地址：[https://gceasy.io/](https://gceasy.io/)

### **GCViewer**

GCViewer是一款离线的GC日志分析器，用于可视化Java VM选项 -verbose:gc 和 .NET生成的数据 `-Xloggc:<file>`。还可以计算与垃圾回收相关的性能指标（吞吐量、累积的暂停、最长的暂停等）。当通过更改世代大小或设置初始堆大小来调整特定应用程序的垃圾回收时，此功能非常有用。

源码下载：[https://github.com/chewiebug/GCViewer](https://github.com/chewiebug/GCViewer)
运行版本下载：[https://github.com/chewiebug/GCViewer/wiki/Changelog](https://github.com/chewiebug/GCViewer/wiki/Changelog)

### **HPjmeter**

- 工具很强大，但是只能打开由以下参数生成的GC log，-verbose:gc  -Xloggc:gc.log。添加其他参数生成的gc.log无法打开
- HPjmeter集成了以前的HPjtune功能，可以分析在HP机器上产生的垃圾回收日志文件
## 调优相关问题
### 生产环境发生了内存溢出该如何处理？
### 生产环境应该给服务器分配多少内存合适？
### 如何对垃圾回收器的性能进行调优？
### 生产环境CPU负载飙高该如何处理？
### 生产环境应该给应用分配多少线程合适？
### 不加log，如何确定请求是否执行了某一行代码？
### 不加log，如何实时查看某个方法的入参与返回值？
# 拓展
## 字节码文件中有哪些重要的内容？::
### 常量池
常量池主要存放编译期生成的各种**字面量**和**符号引用**，**这部分内容在类加载后进入方法区的运行时常量池**。字符串常量池放到了堆中。

## 命令相关
* `java -XX:+PrintFlagsInitial` 查看所有JVM参数启动的初始值
* `java -XX:+PrintFlagsFinal` 查看所有JVM参数的最终值
* `java -XX:+PrintCommandLineFlags` 查看哪些已经被用户或者JVM设置过的详细的XX参数的名称和值
* OOM自动dump：`-XX: +HeapDumpOnOutOfMemoryError`：在程序发生OOM时，导出应用程序的当前堆快照。`-XX: +HeapDumpPath`：可以指定堆快照的保存位置

## 浅堆深堆与内存泄露
### 浅堆（Shallow Heap）

浅堆是指一个对象所消耗的内存。在32位系统中，一个对象引用会占据4个字节，一个int类型会占据4个字节，long型变量会占据8个字节，每个对象头需要占用8个字节。根据堆快照格式不同，对象的大小可能会同8字节进行对齐。

以String为例：2个int值共占8字节，对象引用占用4字节，对象头8字节，合计20字节，向8字节对齐，故占24字节。（jdk7中）

|   |   |   |
|---|---|---|
|int|hash32|0|
|**int**|**hash**|**0**|
|**ref**|**value**|**C:\Users\Administrat**|

这24字节为String对象的浅堆大小。它与String的value实际取值无关，无论字符串长度如何，浅堆大小始终是24字节。

### 保留集（Retained Set）

对象A的保留集指当对象A被垃圾回收后，可以被释放的所有的对象集合（包括对象A本身），即对象A的保留集可以被认为是只能通过对象A被直接或间接访问到的所有对象的集合。通俗地说，就是指仅被对象A所持有的对象的集合。

### 深堆（Retained Heap）

深堆是指对象的保留集中所有的对象的浅堆大小之和。

注意：浅堆指对象本身占用的内存，不包括其内部引用对象的大小。一个对象的深堆指只能通过该对象访问到的（直接或间接）所有对象的浅堆之和，即对象被回收后，可以释放的真实空间。

### 对象的实际大小

这里，对象的实际大小定义为一个对象所能触及的所有对象的浅堆大小之和，也就是通常意义上我们说的对象大小。与深堆相比，似乎这个在日常开发中更为直观和被人接受，但实际上，这个概念和垃圾回收无关。

下图显示了一个简单的对象引用关系图，对象A引用了C和D，对象B引用了C和E。那么对象A的浅堆大小只是A本身，不含C和D，而A的实际大小为A、C、D三者之和。而A的深堆大小为A与D之和，由于对象C还可以通过对象B访问到，因此不在对象A的深堆范围内。

![](对象引用.png)

### 支配树（Dominator Tree）

支配树的概念源自图论。MAT提供了一个称为支配树（Dominator Tree）的对象图。支配树体现了对象实例间的支配关系。在对象引用图中，所有指向对象B的路径都经过对象A，则认为对象A支配对象B。如果对象A是离对象B最近的一个支配对象，则认为对象A为对象B的直接支配者。支配树是基于对象间的引用图所建立的，它有以下基本性质：

- 对象A的子树（所有被对象A支配的对象集合）表示对象A的保留集（retained set），即深堆。
- 如果对象A支配对象B，那么对象A的直接支配者也支配对象B。
- 支配树的边与对象引用图的边不直接对应。

如下图所示：左图表示对象引用图，右图表示左图所对应的支配树。对象A和B由根对象直接支配，由于在到对象C的路径中，可以经过A，也可以经过B，因此对象C的直接支配者也是根对象。对象F与对象D相互引用，因为到对象F的所有路径必然经过对象D，因此，对象D是对象F的直接支配者。而到对象D的所有路径中，必然经过对象C，即使是从对象F到对象D的引用，从根节点出发，也是经过对象C的，所以，对象D的直接支配者为对象C。同理，对象E支配对象G。到达对象H的可以通过对象D，也可以通过对象E，因此对象D和E都不能支配对象H，而经过对象C既可以到达D也可以到达E，因此对象C为对象H的直接支配者。

![](支配树.png)

## 内存泄漏（memory leak）

可达性分析算法来判断对象是否是不再使用的对象，本质都是判断一个对象是否还被引用。那么对于这种情况下，由于代码的实现不同就会出现很多种内存泄漏问题（让JVM误以为此对象还在引用中，无法回收，造成内存泄漏）。

＞ 是否还被使用？是
＞ 是否还被需要？否

![](内存泄露1.png)
严格来说，只有对象不会再被程序用到了，但是GC又不能回收他们的情况，才叫内存泄漏。但实际情况很多时候一些不太好的实践（或疏忽）会导致对象的生命周期变得很长甚至导致00M，也可以叫做宽泛意义上的“内存泄漏”。

如下图，当Y生命周期结束的时候，X依然引用着Y，这时候，垃圾回收期是不会回收对象Y的；如果对象X还引用着生命周期比较短的A、B、C，对象A又引用着对象 a、b、c，这样就可能造成大量无用的对象不能被回收，进而占据了内存资源，造成内存泄漏，直到内存溢出。

![](内存泄漏2.png)
申请了内存用完了不释放，比如一共有1024M的内存，分配了512M的内存一直不回收，那么可以用的内存只有512M了，仿佛泄露掉了一部分；通俗一点讲的话，内存泄漏就是【占着茅坑不拉shi】

### 内存溢出（out of memory）

申请内存时，没有足够的内存可以使用；通俗一点儿讲，一个厕所就三个坑，有两个站着茅坑不走的（内存泄漏），剩下最后一个坑，厕所表示接待压力很大，这时候一下子来了两个人，坑位（内存）就不够了，内存泄漏变成内存溢出了。可见，内存泄漏和内存溢出的关系：内存泄漏的增多，最终会导致内存溢出。

#### 泄漏的分类

- 经常发生：发生内存泄露的代码会被多次执行，每次执行，泄露一块内存；
- 偶然发生：在某些特定情况下才会发生
- 一次性：发生内存泄露的方法只会执行一次；
- 隐式泄漏：一直占着内存不释放，直到执行结束；严格的说这个不算内存泄漏，因为最终释放掉了，但是如果执行时间特别长，也可能会导致内存耗尽。

## Java中内存泄露的8种情况

### 静态集合类

静态集合类，如HashMap、LinkedList等等。如果这些容器为静态的，那么它们的生命周期与JVM程序一致，则容器中的对象在程序结束之前将不能被释放，从而造成内存泄漏。简单而言，长生命周期的对象持有短生命周期对象的引用，尽管短生命周期的对象不再使用，但是因为长生命周期对象持有它的引用而导致不能被回收。
```java
public class MemoryLeak {
    static List list = new ArrayList();
    public void oomTests(){
        Object obj＝new Object();//局部变量
        list.add(obj);
    }
}
```

### 单例模式

单例模式，和静态集合导致内存泄露的原因类似，因为单例的静态特性，它的生命周期和 JVM 的生命周期一样长，所以如果单例对象如果持有外部对象的引用，那么这个外部对象也不会被回收，那么就会造成内存泄漏。

### 内部类持有外部类

内部类持有外部类，如果一个外部类的实例对象的方法返回了一个内部类的实例对象。这个内部类对象被长期引用了，即使那个外部类实例对象不再被使用，但由于内部类持有外部类的实例对象，这个外部类对象将不会被垃圾回收，这也会造成内存泄漏。

### 各种连接，如数据库连接、网络连接和IO连接等

在对数据库进行操作的过程中，首先需要建立与数据库的连接，当不再使用时，需要调用close方法来释放与数据库的连接。只有连接被关闭后，垃圾回收器才会回收对应的对象。否则，如果在访问数据库的过程中，对Connection、Statement或ResultSet不显性地关闭，将会造成大量的对象无法被回收，从而引起内存泄漏。

```java
public static void main(String[] args) {
    try{
        Connection conn =null;
        Class.forName("com.mysql.jdbc.Driver");
        conn =DriverManager.getConnection("url","","");
        Statement stmt =conn.createStatement();
        ResultSet rs =stmt.executeQuery("....");
    } catch（Exception e）{//异常日志
    } finally {
        // 1．关闭结果集 Statement
        // 2．关闭声明的对象 ResultSet
        // 3．关闭连接 Connection
    }
}
```

### 变量不合理的作用域

变量不合理的作用域。一般而言，一个变量的定义的作用范围大于其使用范围，很有可能会造成内存泄漏。另一方面，如果没有及时地把对象设置为null，很有可能导致内存泄漏的发生。

```java
public class UsingRandom {
    private String msg;
    public void receiveMsg(){
        readFromNet();//从网络中接受数据保存到msg中
        saveDB();//把msg保存到数据库中
    }
}
```

如上面这个伪代码，通过readFromNet方法把接受的消息保存在变量msg中，然后调用saveDB方法把msg的内容保存到数据库中，此时msg已经就没用了，由于msg的生命周期与对象的生命周期相同，此时msg还不能回收，因此造成了内存泄漏。实际上这个msg变量可以放在receiveMsg方法内部，当方法使用完，那么msg的生命周期也就结束，此时就可以回收了。还有一种方法，在使用完msg后，把msg设置为null，这样垃圾回收器也会回收msg的内存空间。

### 改变哈希值

改变哈希值，当一个对象被存储进HashSet集合中以后，就不能修改这个对象中的那些参与计算哈希值的字段了。

否则，对象修改后的哈希值与最初存储进HashSet集合中时的哈希值就不同了，在这种情况下，即使在contains方法使用该对象的当前引用作为的参数去HashSet集合中检索对象，也将返回找不到对象的结果，这也会导致无法从HashSet集合中单独删除当前对象，造成内存泄漏。

这也是 String 为什么被设置成了不可变类型，我们可以放心地把 String 存入 HashSet，或者把String 当做 HashMap 的 key 值；

当我们想把自己定义的类保存到散列表的时候，需要保证对象的 hashCode 不可变。

```java
/**
 * 例1
 */
public class ChangeHashCode {
    public static void main(String[] args) {
        HashSet set = new HashSet();
        Person p1 = new Person(1001, "AA");
        Person p2 = new Person(1002, "BB");

        set.add(p1);
        set.add(p2);

        p1.name = "CC";//导致了内存的泄漏
        set.remove(p1); //删除失败

        System.out.println(set);

        set.add(new Person(1001, "CC"));
        System.out.println(set);

        set.add(new Person(1001, "AA"));
        System.out.println(set);

    }
}

class Person {
    int id;
    String name;

    public Person(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;

        Person person = (Person) o;

        if (id != person.id) return false;
        return name != null ? name.equals(person.name) : person.name == null;
    }

    @Override
    public int hashCode() {
        int result = id;
        result = 31 * result + (name != null ? name.hashCode() : 0);
        return result;
    }

    @Override
    public String toString() {
        return "Person{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```

```java
/**
 * 例2
 */
public class ChangeHashCode1 {
    public static void main(String[] args) {
        HashSet<Point> hs = new HashSet<Point>();
        Point cc = new Point();
        cc.setX(10);//hashCode = 41
        hs.add(cc);

        cc.setX(20);//hashCode = 51  此行为导致了内存的泄漏

        System.out.println("hs.remove = " + hs.remove(cc));//false
        hs.add(cc);
        System.out.println("hs.size = " + hs.size());//size = 2

        System.out.println(hs);
    }

}

class Point {
    int x;

    public int getX() {
        return x;
    }

    public void setX(int x) {
        this.x = x;
    }

    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + x;
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null) return false;
        if (getClass() != obj.getClass()) return false;
        Point other = (Point) obj;
        if (x != other.x) return false;
        return true;
    }

    @Override
    public String toString() {
        return "Point{" +
                "x=" + x +
                '}';
    }
}
```

### 缓存泄露

内存泄漏的另一个常见来源是缓存，一旦你把对象引用放入到缓存中，他就很容易遗忘。比如：之前项目在一次上线的时候，应用启动奇慢直到夯死，就是因为代码中会加载一个表中的数据到缓存（内存）中，测试环境只有几百条数据，但是生产环境有几百万的数据。

对于这个问题，可以使用WeakHashMap代表缓存，此种Map的特点是，当除了自身有对key的引用外，此key没有其他引用那么此map会自动丢弃此值。

```java
public class MapTest {
    static Map wMap = new WeakHashMap();
    static Map map = new HashMap();

    public static void main(String[] args) {
        init();
        testWeakHashMap();
        testHashMap();
    }

    public static void init() {
        String ref1 = new String("obejct1");
        String ref2 = new String("obejct2");
        String ref3 = new String("obejct3");
        String ref4 = new String("obejct4");
        wMap.put(ref1, "cacheObject1");
        wMap.put(ref2, "cacheObject2");
        map.put(ref3, "cacheObject3");
        map.put(ref4, "cacheObject4");
        System.out.println("String引用ref1，ref2，ref3，ref4 消失");

    }

    public static void testWeakHashMap() {
        System.out.println("WeakHashMap GC之前");
        for (Object o : wMap.entrySet()) {
            System.out.println(o);
        }
        try {
            System.gc();
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("WeakHashMap GC之后");
        for (Object o : wMap.entrySet()) {
            System.out.println(o);
        }
    }

    public static void testHashMap() {
        System.out.println("HashMap GC之前");
        for (Object o : map.entrySet()) {
            System.out.println(o);
        }
        try {
            System.gc();
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("HashMap GC之后");
        for (Object o : map.entrySet()) {
            System.out.println(o);
        }
    }

}
```

上面代码和图示主演演示WeakHashMap如何自动释放缓存对象，当init函数执行完成后，局部变量字符串引用weakd1，weakd2，d1，d2都会消失，此时只有静态map中保存中对字符串对象的引用，可以看到，调用gc之后，HashMap的没有被回收，而WeakHashMap里面的缓存被回收了。

### 监听器和其他回调

内存泄漏第三个常见来源是监听器和其他回调，如果客户端在你实现的API中注册回调，却没有显示的取消，那么就会积聚。

需要确保回调立即被当作垃圾回收的最佳方法是只保存它的弱引用，例如将他们保存成为WeakHashMap中的键。

# 内存泄露案例分析

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) { //入栈
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() { //出栈
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

上述程序并没有明显的错误，但是这段程序有一个内存泄漏，随着GC活动的增加，或者内存占用的不断增加，程序性能的降低就会表现出来，严重时可导致内存泄漏，但是这种失败情况相对较少。

代码的主要问题在pop函数，下面通过这张图示展现。假设这个栈一直增长，增长后如下图所示
![](内存泄漏分析1.png)

当进行大量的pop操作时，由于引用未进行置空，gc是不会释放的，如下图所示
![](内存泄漏分析2.png)
从上图中看以看出，如果栈先增长，再收缩，那么从栈中弹出的对象将不会被当作垃圾回收，即使程序不再使用栈中的这些队象，他们也不会回收，因为栈中仍然保存这对象的引用，俗称过期引用，这个内存泄露很隐蔽。

将代码中的pop()方法变成如下方法：
```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

一旦引用过期，清空这些引用，将引用置空。
![](内存泄漏分析3.png)

# 使用OQL语言查询对象信息

MAT支持一种类似于SQL的查询语言OQL（Object Query Language）。OQL使用类SQL语法，可以在堆中进行对象的查找和筛选。

## SELECT子句

在MAT中，Select子句的格式与SQL基本一致，用于指定要显示的列。Select子句中可以使用“＊”，查看结果对象的引用实例（相当于outgoing references）。

```sql
SELECT * FROM java.util.Vector v
```

使用“OBJECTS”关键字，可以将返回结果集中的项以对象的形式显示。
```sql
SELECT objects v.elementData FROM java.util.Vector v

SELECT OBJECTS s.value FROM java.lang.String s
```

在Select子句中，使用“AS RETAINED SET”关键字可以得到所得对象的保留集。

```sql
SELECT AS RETAINED SET *FROM com.atguigu.mat.Student
```

“DISTINCT”关键字用于在结果集中去除重复对象。

```sql
SELECT DISTINCT OBJECTS classof(s) FROM java.lang.String s
```

## FROM子句

From子句用于指定查询范围，它可以指定类名、正则表达式或者对象地址。

```sql
SELECT * FROM java.lang.String s
```
使用正则表达式，限定搜索范围，输出所有com.atguigu包下所有类的实例

```sql
SELECT * FROM "com\.atguigu\..*"
```
使用类的地址进行搜索。使用类的地址的好处是可以区分被不同ClassLoader加载的同一种类型。

```sql
select * from 0x37a0b4d
```

## WHERE子句

Where子句用于指定OQL的查询条件。OQL查询将只返回满足Where子句指定条件的对象。Where子句的格式与传统SQL极为相似。

返回长度大于10的char数组。

```java
SELECT *FROM Ichar[] s WHERE s.@length>10
```
返回包含“java”子字符串的所有字符串，使用“LIKE”操作符，“LIKE”操作符的操作参数为正则表达式。
```sql
SELECT * FROM java.lang.String s WHERE toString(s) LIKE ".*java.*"
```

返回所有value域不为null的字符串，使用“＝”操作符。
```java
SELECT * FROM java.lang.String s where s.value!=null
```

返回数组长度大于15，并且深堆大于1000字节的所有Vector对象。

```sql
SELECT * FROM java.util.Vector v WHERE v.elementData.@length>15 AND v.@retainedHeapSize>1000
```

## 内置对象与方法

OQL中可以访问堆内对象的属性，也可以访问堆内代理对象的属性。访问堆内对象的属性时，格式如下，其中alias为对象名称：

`[ <alias>. ] <field> . <field>. <field>`

访问java.io.File对象的path属性，并进一步访问path的value属性：

```sql
SELECT toString(f.path.value) FROM java.io.File f
```

显示String对象的内容、objectid和objectAddress。
```sql
SELECT s.toString(),s.@objectId, s.@objectAddress FROM java.lang.String s
```

显示java.util.Vector内部数组的长度。

```sql
SELECT v.elementData.@length FROM java.util.Vector v

```
显示所有的java.util.Vector对象及其子类型

```sql
select * from INSTANCEOF java.util.Vector
```





