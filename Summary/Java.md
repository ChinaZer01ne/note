# 集合
## 集合类继承关系
* Collection
	* List  
	    - ArrayList  
	    - LinkedList  
	    - CopyOnWriteArrayList  
	- Set  
	    - HashSet  
	    - TreeSet  
	- Queue  
	    - Deque
	    - ArrayDeque  
	    - PriorityQueue
* Map
	* HashMap  
	- ConcurrentHashMap  
	- TreeMap  
	- ConcurrentSkipListMap  
	- LinkedHashMap
## Collection
### ArrayList
* 懒加载的思想
* 初始化10
* 1.5倍扩容
### LinkedList
根据查找index的位置决定从前往后查找还是从后往前查找

> 编者注：需了解ArrayList和LinkedList的底层原理与区别。
### HashSet
HashSet底层是HashMap，map.put(e, new Object())，Key是Set的元素，Value是一个默认对象
### CopyOnWriteArrayList

其set方法很有意思，如果set的元素和原来元素相同，他会重新复制一下原数组的引用，这是为了保证volatile的语义，否则如果set不一样的元素会刷新内存，保存一样的元素不刷新内存，这会导致set方法之前的语句是否具有可见性表现出两种不同的现象，这样会导致代码很奇怪
- 写时复制  
- 缺点  
	读多写少场景  
	内存消耗大  
	数据一致性，可能读不到刚写的数据

#### 原理
* 每次添加元素的时候，操作者需要获取锁，也就意味着不能同时添加，添加的人会在原数组的基础上copy出新的数组，在新数组上写操作，最后变换引用。
* 读操作，无锁直接读，所以可能读不到刚添加的元素

#### 其他
* `CopyOnWriteArraySet`底层是`CopyOnWriteArrayList`，循环比较保证Set不重复

## Map
### HashMap
#### HashMap的数据结构是什么样的？::  
- **数组 + 链表 + 红黑树**（链表长度 ≥8 且数组长度 ≥64 时，链表转红黑树；红黑树节点 ≤6 时转回链表）
- 数组索引通过 `(n - 1) & hash` 计算（`n` 是数组长度）
> 为什么这么设计？1.7中仅仅是拉链法
> * 哈希碰撞性能退化： 如果大量键的哈希值冲突，导致某些桶的链表变得非常长。`get()`/`put()` 操作在该桶上退化为 O(n) 的时间复杂度，性能急剧下降（哈希碰撞攻击风险）。
> * 并发问题依旧：本身非线程安全，并发操作可能导致死循环（Java 7 扩容头插法导致）、数据丢失、size 不准。
#### 如何解决hash冲突以及应用场景? :: 
- 开放定址法  
	- ThreadLocal  
- 再哈希法  
	- 布隆过滤器  
- 链地址法  
	- HashMap  
- 建立公共溢出区  
#### 关键参数
- `DEFAULT_INITIAL_CAPACITY = 16`
- `DEFAULT_LOAD_FACTOR = 0.75f`
- 扩容阈值 = 容量 × 负载因子。
#### 讲一下HashMap的懒加载？::
- 懒加载  
#### 讲一下HashMap的扩容机制？::
* 首次调用put方法时，HashMap会发现table为空然后调用resize方法进行初始化。  
* 非首次调用put方法时，若HashMap发现size（元素个数）大于threshold（阈值）（数组长度乘以加载因子的值）并且添加位置出现了hash冲突（链表），则会调用resize方法进行扩容。  
* 链表长度大于8 且数组长度小于64 会进行扩容。
* 扩容时数组大小翻倍（`newCap = oldCap << 1`）。
- 重新计算元素位置：`newIndex = e.hash & (newCap - 1)`。
- **Java 8 优化**：扩容后元素位置要么在原索引，要么在 `原索引 + oldCap`。
#### HashMap有什么并发问题？::  
* 1.7：头插
	* 实际上头插法比尾插法效率高  
* 1.8：尾插
* **死循环**（Java 7 链表头插法导致，Java 8 改用尾插法已解决）。
- **数据丢失**：并发 put 时覆盖键值对。
- **size 不准确**：并发更新导致统计错误。
#### Key 的设计要求
- 重写 `hashCode()` 和 `equals()` 方法。
- **不可变性**：避免修改 Key 导致哈希值变化（如用 `String`、`Integer` 作 Key）。
#### HashMap其他内容
- 链表与红黑树的转化  
	8树化，6链化
- 二次散列  
- 扰动函数  
> 编者注：需了解：
> * HashMap的底层数据结构
> 	* 负载因子
> 	* hash冲突
> 	* 树化与链化
> * HashMap的put执行流程
> * HashMap的扩容机制
> * HashMap在多线程下存在什么问题
### ConcurrentHashMap
#### ConcurrentHashMap如何解决并发问题的？::
- 并发处理 
	- ava 7 分段锁（Segment
		- 结构：分多个 Segment（继承 `ReentrantLock`），每个 Segment 是一个小的 HashMap。
		- 并发度：锁粒度是 Segment 级别，不同 Segment 可并行操作。
	- Java 8 的优化（CAS + synchronized）
		* 锁粒度细化
		    - 锁单个数组桶（链表头节点/红黑树根节点），并发度 = 数组长度。
		    - 使用 `synchronized` 锁头节点（非整个表），减少锁竞争。
- sizeCtrl的含义  
	- -1  
		初始化的时候会尝试CAS将sizeCtrl设置为-1
	- >0  
		扩容阈值，threshold
	- -n  
		表示扩容线程 + 1
- hash函数的特殊处理  
	- hash>=0,是Node节点；  
	- hash=-2是TreeBin,表示该table元素是红黑树的节点；  
	- hash=-1是ForwardingNode；容器正在扩容  
	- hash = -3是ReservationNode，在调用computeIfAbsent方法时可能会使用的占位对象  
#### ConcurrentHashMap是如何扩容的？::
* 可能的触发时机
	* 链表长度到8，数组长度没有到64
	* 超过负载因子阈值（0.75）
	* putAll方法
- 扩容  
	- 辅助扩容  
		- 计算步长
		- 创建新数组
		- 领取任务
		- 迁移数据
		- 核查
	- 数据迁移  
		- lastRun  
	- 扩容期间读写问题  
    - 计数器的实现  
        - baseCount + counterCell数组  
    - ForwardingNode  
        迁移完毕  
        读跳转
- 红黑树并发读写问题  
#### ConcurrentHashMap在红黑树上为什么又双向链表指针？

保证ConcurrentHashMap在写的时候，红黑树发生宣战，不影响对ConcurrentHashMap的读操作。在扩容的时候是通过双向链表来迁移数据的。

#### 关键操作

- **`putVal()`**：
	- 桶为空时：用 `CAS` 写入头节点（无锁）。
	- 桶非空时：`synchronized` 锁头节点后插入。
		
- **`get()`**：无锁（通过 `volatile` 保证可见性）。
- **扩容（transfer）**：
	- 支持多线程协同扩容（线程可帮助迁移数据）。
	- 通过 `ForwardingNode` 标记迁移中的桶。
		
- **计数（size()）**：
	- 使用 `CounterCell[]`（类似 LongAdder）分段统计，避免竞争。
            
#### **线程安全设计**：

- `volatile Node<K,V>[] table`：保证数组引用可见性。
- `volatile Node next`：保证链表遍历可见性。
- `UNSAFE.compareAndSwap***`：实现无锁更新。

> 编者注：需了解：
> * ConcurrentHashMap如何解决并发问题
> * ConcurrentHashMap是如何扩容的
> * ConcurrentHashMap中sizeCtrl的含义

#### HashMap vs ConcurrentHashMap

| **特性**       | **HashMap** | **ConcurrentHashMap**       |
| ------------ | ----------- | --------------------------- |
| **线程安全**     | ❌ 不安全       | ✅ 安全                        |
| **锁机制**      | 无锁          | Java 7：分段锁；Java 8：桶级锁 + CAS |
| **Null 键/值** | ✅ 允许        | ❌ 不允许（会抛 NPE）               |
| **迭代器**      | `Fail-Fast` | `Weakly Consistent`（弱一致性）   |
| **性能开销**     | 低（单线程）      | 中等（锁细化降低竞争）                 |
#### 加分项
> * 解释 **哈希扰动函数** `(h = key.hashCode()) ^ (h >>> 16)`（高位参与运算，减少冲突）。
> * 了解 **CHM 中的 `ForwardingNode`**（扩容时转发查询）。
> * 理解 **`sizeCtl` 的作用**（控制初始化、扩容状态）。
### ConcurrentSkipListMap
跳表  
### LinkedHashMap
LRU
# 并发

## Java中有哪些实现线程安全的方式？::
1. **使用同步机制**：可以使用关键字[`synchronized`](#synchronized) 或 `ReentrantLock` 类来实现互斥访问，确保同一时间只有一个线程可以访问共享资源。
2. **使用原子类**：Java提供了一系列原子类，如 `AtomicInteger`、`AtomicLong`、`AtomicReference` 等，它们提供了原子操作，保证了操作的原子性，避免了多线程竞争的问题。
3. **使用线程安全的集合类**：Java提供了线程安全的集合类，如 `ConcurrentHashMap`、`CopyOnWriteArrayList` 等，它们在内部实现上使用了同步机制，可以安全地在多线程环境下使用。
4. **使用volatile关键字**：`volatile` 关键字可以确保共享变量的可见性，即对一个 `volatile` 变量的写操作对于其他线程是立即可见的，从而避免了多线程之间的数据不一致问题。
5. **使用ThreadLocal类**：`ThreadLocal` 类提供了线程局部变量的功能，每个线程都有自己独立的变量副本，避免了线程间的数据共享和竞争。
6. **使用并发工具类**：Java提供了一系列并发工具类，如 `CountDownLatch`、`CyclicBarrier`、`Semaphore` 等，它们可以实现线程间的协调和同步，确保多线程操作的正确性。
7. **使用不可变对象**：不可变对象是指创建后状态不可变的对象，比如用[final]()修饰的变量。它们不需要额外的同步机制就可以在多线程环境下安全地共享。
## Java的内存模型
### 什么是JMM？::
JMM（Java Memory Model）是Java内存模型的缩写，它定义了Java程序中多线程并发访问共享内存时的行为规范。JMM规定了线程如何与主内存和线程本地内存交互，以及如何保证可见性、有序性和原子性。
### 为什么要有JMM？::
1. **跨平台性**：
>JMM的设计目标之一是保证Java程序在不同平台上的一致性行为。通过在不同平台上定义共享内存的访问规则，JMM可以确保Java程序在各种操作系统和硬件架构上的一致性。
2. **提供并发编程的规范**：
>多线程编程中存在一些常见的问题，如竞态条件（Race Condition）、死锁（Deadlock）和内存可见性等。JMM提供了一套规范，定义了如何正确地编写并发程序，以避免这些问题。
3. **优化程序执行**：
>JMM还定义了一些允许编译器和处理器进行的优化规则。这些规则允许编译器和处理器对指令进行重排序，以提高程序的执行效率。同时，JMM也提供了一些特殊的指令和内存屏障，用于保证多线程环境下的可见性和有序性
![](/Summary/image/Java/JMM抽象结构图.png)
### JMM规范了哪些内容？::
Java内存模型实际上就是规范了JVM如何提供按需禁用缓存和重排序优化的方法。其核心就包括[volatile](#volatile)、synchronized和final三个关键字，以及几项Happens-Before规则。

1. 所有共享变量都存储于主内存。这里说的变量指的是实例变量和类变量，不包含局部变量，因为局部变量是线程私有的，不存在竞争问题
2. 每个线程还存在自己的工作内存，线程的工作内存，保留了被线程使用的变量的工作副本，线程对变量的所有操作，都必须在工作内存中完成，而不能直接读写主内存中转来完成
3. 不同线程之间也不能直接访问对方工作内存中的变量，线程间变量值的传递需要通过主内存中转来完成
4. 内存屏障
5. Happens-Before原则
#### JMM规范了8中原子性操作
**lock（锁定）**：作用于主内存，它把一个变量标记为一条线程独占状态；
**read（读取）**：作用于主内存，它把变量值从主内存传送到线程的工作内存中，以便随后的load动作使用；
**load（载入）**：作用于工作内存，它把read操作的值放入工作内存中的变量副本中；
**use（使用）**：作用于工作内存，它把工作内存中的值传递给执行引擎，每当虚拟机遇到一个需要使用这个变量的指令时候，将会执行这个动作；
**assign（赋值）**：作用于工作内存，它把从执行引擎获取的值赋值给工作内存中的变量，每当虚拟机遇到一个给变量赋值的指令时候，执行该操作；
**store（存储）**：作用于工作内存，它把工作内存中的一个变量传送给主内存中，以备随后的write操作使用；
**write（写入）**：作用于主内存，它把store传送值放到主内存中的变量中。
**unlock（解锁）**：作用于主内存，它将一个处于锁定状态的变量释放出来，释放后的变量才能够被其他线程锁定；

>a = b 并不是原子性操作， read a; assign b;

Java内存模型还规定了执行上述8种基本操作时必须满足如下规则:
1、不允许read和load、store和write操作之一单独出现（即不允许一个变量从主存读取了但是工作内存不接受，或者从工作内存发起写操作但是主存不接受的情况），以上两个操作必须按顺序执行，但没有保证必须连续执行，也就是说，read与load之间、store与write之间是可插入其他指令的。
2、不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。
3、不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。
4、一个新的变量只能从主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。
5、一个变量在同一个时刻只允许一条线程对其执行lock操作，但lock操作可以被同一个条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁（重入）。
6、如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
7、如果一个变量实现没有被lock操作锁定，则不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。
8、对一个变量执行unlock操作之前，必须先把此变量同步回主内存（执行store和write操作）。
![](JMM原子性操作.jpg)
#### 什么是happens-before原则?
1. **程序次序规则**：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写后面的操作。（这里说的是具有依赖性的代码，如果代码之间不存在依赖性，那么还是会出现指令重排序的情况，这条规则是用来保证单线程的执行的有序性）
2. **锁定规则**：一个`unlock`操作先行发生于后面对同一个锁的`lock`操作。
3. **volatile变量原则**：对一个变量的写操作先行发生于后面对这个变量的读操作。
4. **传递规则**：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C。
5. **线程启动规则**：Thread对象的`start()`方法先行发生于此线程的每一个动作。
6. **线程中断规则**：对线程`interrupt()`方法的调用先行发生于被中断线程的代码检测到中断事件的发生。
7. **线程终结规则**：线程中所有的操作都先行发生于线程的终止检测，我们可以通过`Thread.join()`方法结束、`Thread.isAlive()`的返回值手段检测到线程已经终止执行。
8. **对象终结规则**：一个对象的初始化完成先行发生于他的`finalize()`方法的开始。

### JMM和CPU多级缓存
JMM屏蔽了不同硬件细节，定义了不同机器下，Java程序中多线程并发访问共享内存时的行为规范，其中存在线程本地内存不一致的问题。在CPU多级缓存下，同样存在缓存不一致的问题。无论是JMM中线程工作内存和主存的不一致，还是CPU多级缓存和主存的不一致，java中提供了一系列关键字以及锁来影响这些行为。

### 一个可见性的例子
```java
public class ThreadTest {  
  
    public static int a = 0;  
  
    public static void main(String[] args) {  
        new Thread(() -> {  
            int tmp = a;  
            while (tmp < 50) {  
                synchronized (ThreadTest.class) {  
                    if (tmp != a) {  
                        try {  
                            Thread.sleep(100);  
                        } catch (InterruptedException e) {  
                            throw new RuntimeException(e);  
                        }  
                        tmp = a;  
                    }  
  
                }  
            }  
  
            System.out.println("read thread :" + a);  
        }, "read").start();  
  
        new Thread(() -> {  
            while (a < 50) {  
                    try {  
                        Thread.sleep(100);  
                    } catch (InterruptedException e) {  
                        throw new RuntimeException(e);  
                    }  
                    a++;  
            }  
            System.out.println("write thread :" + a);  
        }, "write").start();  
  
    }  
}
```
#### 解决方式
##### 方式一: volatile
```java
public static volatile int a = 0;  
```
##### 方式二：synchronized
```java
// 读线程增加synchronized
new Thread(() -> {  
    int tmp = a;  
    while (tmp < 50) {  
        synchronized (ThreadTest.class) {  
            if (tmp != a) {  
                try {  
                    Thread.sleep(100);  
                } catch (InterruptedException e) {  
                    throw new RuntimeException(e);  
                }  
                tmp = a;  
            }  
  
        }  
    }  
  
    System.out.println("read thread :" + a);  
}, "read").start();
```
##### 为什么这么加不可以？

```java
synchronized (ThreadTest.class) {  
	while (tmp < 50) {  
		if (tmp != a) {  
			try {  
				Thread.sleep(100);  
			} catch (InterruptedException e) {  
				throw new RuntimeException(e);  
			}  
			tmp = a;  
		}  
	}  
}  
```
这么加，a变量在循环中只会读取一次，因为synchronized只会执行一次；如果synchronized加在循环里面，由于synchronized执行多次，那么变量a就会多次从主存读取。
## 阻塞队列
* BlockingQueue
	* **ArrayBlockingQueue**
		* 数组有界队列
	* **LinkedBlockingQueue**
		* 链表有界队列，最大值Integer.Max
	* **PriorityBlockingQueue**
		* 支持优先级无界队列
	* **DelayQueue**
		* 优先级的延迟无界队列
	* **SynchronousQueue**
		* 不存储元素的阻塞队列，就是单个元素的队列
	* ...
## AQS
### AQS是如何实现的？::
#### AQS的数据结构
* volatile修饰的state
* Node组成的双向链表
* Condition单链表
#### 主要方法
acquire方法：当线程尝试获取同步状态时，会调用AQS的acquire方法。该方法首先通过CAS操作尝试获取同步状态，如果成功则返回；如果失败，则线程会被加入到等待队列中，并进入阻塞状态。
release方法：当线程释放同步状态时，会调用AQS的release方法。该方法会释放同步状态，并尝试唤醒等待队列中的线程。
### AQS有哪些应用？::
* ReentrantLock
* ReentrantReadWriteLock
* CountDownLatch
	* 所有线程countDown，变成0，最后线程才执行
	* 做减法
* CyclicBarrier
	* 所有线程await，变成某值，最后await的线程一起执行
	* 做加法
* Semaphore
	* 多个共享资源互斥
### AQS主要的面试点有哪些？::
TODO 

## 锁

### synchronized
#### 什么是自旋锁？::
许多情况下，共享数据的锁定状态持续时间较短，挂起线程不值得，通过让线程执行忙循环等待锁的释放，不让出CPU。缺点：若锁被其他线程长时间占用，会带来许多性能上的开销（消耗cpu不做事情，cpu的使用率就下降。）
> 实现锁的主要难点在于锁的acquire接口，在acquire里面有一个死循环，循环中判断锁对象的locked字段是否为0，如果为0那表明当前锁没有持有者，当前对于acquire的调用可以获取锁。之后我们通过设置锁对象的locked字段为1来获取锁。最后返回。如果锁的locked字段不为0，那么当前对于acquire的调用就不能获取锁，程序会一直spin。也就是说，程序在循环中不停的重复执行，直到锁的持有者调用了release并将锁对象的locked设置为0。 为了防止两个进程可能同时读到锁的locked字段为0，CPU提供了特殊的指令就是amoswap（atomic memory swap）来保证。
#### JVM做了哪些锁优化操作
* 自适应自自旋
	> 自旋的时间不再固定。由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定。如果对于某个锁，自旋很少成功获得过锁，那以后要获取这个锁时将有可能直接省略掉自旋过程，避免浪费处理器资源。
* 锁消除
	> `JIT`编译时，对运行上下文进行扫描，去除不可能存在竞争的锁。比如说：`StringBuffer`是线程安全的，因为是`synchronized`修饰的，如果`StringBuffer`是一个局部变量，`JVM`就会自动消除`StringBuffer`内部的锁来提升性能。
* 锁粗化
	> 通过扩大加锁范围避免反复加锁和解锁。比如在循环中执行，`StringBuffer`的`append`操作时，锁范围会扩大
* 轻量级锁
* 偏向锁
- ...
#### 说一说锁升级过程

锁升级方向：**无锁 ---> 偏向锁---> 轻量级锁---> 重量级锁**

锁升级流程：

如果一个线程获得了锁，那么锁就进入偏向模式，此时`Mark Word`的结构也变成为**偏向锁**结构，当该线程再次请求锁时，无需再做任何同步操作，即获取锁的过程只需要检查`Mark Word`的锁标记位为偏向锁以及当前线程Id等于`Mark Word`的`ThreadID`即可，如果`ThreadID`并未指向当前线程，则通过`CAS`操作竞争锁。如果竞争成功，则将`Mark Word`中`ThreadID`设置为当前线程`ID`，如果`CAS`获取偏向锁失败，则表示有竞争。当到达全局安全点（`safepoint`）时获得偏向锁的线程被挂起，偏向锁升级为**轻量级锁**，然后被阻塞在安全点的线程继续往下执行同步代码。这样就省去了大量有关申请锁的操作。
![](/Summary/image/Java/synchronized原理.jpg)
#### 什么是偏向锁？::

偏向锁是Java虚拟机为了提高程序性能而设计的一种锁机制，它的核心思想是在没有竞争的情况下，将对象的标记设置为偏向，并将线程ID记录在对象头中。偏向锁的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。

#### 偏向锁是如何实现的？::
TODO
Mark Word
#### 什么是轻量级锁？::

轻量级锁是一种优化的锁实现方式，旨在减少在无实际竞争情况下使用重量级锁产生的性能消耗。它通过避免系统调用引起的内核态与用户态切换以及线程阻塞造成的线程切换等方式来提高程序的执行效率。轻量级锁的实现基于CAS（Compare And Swap）操作，**当线程需要获取对象的锁时，如果对象未被锁定，该线程将把对象头部的Mark Word拷贝到线程栈的锁记录（Lock Record）中，并使用CAS将对象头部的Mark Word替换为指向锁记录的指针**。如果CAS操作成功，该线程就获得了该对象的锁，可以执行同步操作。如果CAS操作过程中发现随想头部的Mark Word已经被其他线程修改过了，那么说明该对象已经被其他线程锁定了，当前线程需要尝试其他的锁实现方式，比如重量级锁。

https://blog.csdn.net/Weixiaohuai/article/details/126498242
#### 轻量级锁是如何实现的？::
TODO
线程栈帧
#### 什么是重量级锁？::

当锁升级成轻量级锁的时候，线程通过CAS将Mark Word指向栈帧记录失败的时候，会自旋重试，如果还是失败，锁就会升级成重量级锁。
每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的Mark Word 中就被设置指向 Monitor 对象的指针。不加 synchronized 的对象不会关联Monitor。

![](重量级锁monitor.webp)

当有线程获取到锁时，锁对象头中的Mark Word变成了指向Monitor的指针。（原本mark word当中的内容会存储到Monitor当中，释放时会取出这些内容再次放到mark word。）
    
thread3 来竞争这把锁，此时只有它自己，那么thread3将会被设置为Monitor的Owner，有且只能有一个Owner。

如果thread3持有锁的过程中，如果thread4和thread5也来竞争锁，就会添加到EntryList当中，此时线程将被阻塞（BLOCKED）。
    
当thread执行完同步代码块当中的内容，会唤醒EntryList当中的线程来竞争锁，此竞争是非公平的。
    
另外，在WaitSet当中的thread1和thread2，其状态是WAITING，表示他们之前获得过锁，执行了等待方法。

https://baijiahao.baidu.com/s?id=1717781876275288385&wfr=spider&for=pc
#### 重量级锁是如何实现的？::

JVM每个对象都会有一个监视器，监视器和对象一起创建、销毁。
monitor
#### 偏向锁、轻量级锁和重量级锁有什么区别？::

偏向锁，轻量级锁都是乐观锁，重量级锁是悲观锁

| 特点    | 偏向锁                                     | 轻量级锁                                                  | 重量级锁                          |
| ----- | --------------------------------------- | ----------------------------------------------------- | ----------------------------- |
| 竞争状态  | 无                                       | 短暂竞争状态                                                | 激烈竞争状态                        |
| 加锁过程  | 获取锁时，将对象头中的Mark Word设置为指向当前线程的Thread ID | 获取锁时，尝试使用CAS操作将Mark Word修改为指向锁记录（Lock Record）的指针（在栈中） | 获取锁时，涉及到系统调用，例如操作系统提供的互斥量或信号量 |
| 解锁过程  | 解锁时，检查对象头中的Mark Word是否指向当前线程的Thread ID  | 解锁时，使用原子操作将Mark Word恢复为指向对象的原始HashCode                | 解锁时，涉及到系统调用，例如操作系统提供的互斥量或信号量  |
| 锁膨胀过程 | 当另一个线程尝试获取偏向锁时，偏向锁会自动升级为轻量级锁            | 当另一个线程尝试获取轻量级锁时，轻量级锁会自动升级为重量级锁                        | 无                             |
| 性能影响  | 对于只有一个线程访问的场景，性能最佳                      | 性能较好                                                  | 性能较差                          |
| 适用场景  | 适用于只有一个线程频繁访问临界区的场景                     | 适用于短暂竞争状态下的临界区访问                                      | 适用于竞争激烈或长时间占有锁的场景             |

#### 偏向锁，轻量级锁，重量级锁都是怎么实现的？::
TODO
### ReentrantLock
TODO

#### ReentrantLock和synchronized有什么区别
TODO
### 无锁机制的实现方式

#### final
##### final语义中的内存屏障
对于final域，编译器和CPU会遵循两个排序规则：
1. 新建对象过程中，构造体中对final域的初始化写入和这个对象赋值给其他引用变量，这两个操作不能重排序；（废话嘛）
2. 初次读包含final域的对象引用和读取这个final域，这两个操作不能重排序；（晦涩，意思就是先赋值引用，再调用final值）

总之上面规则的意思可以这样理解，必需保证一个对象的所有final域被写入完毕后才能引用和读取。这也是内存屏障的起的作用：
- 写final域：在编译器写final域完毕，构造体结束之前，会插入一个StoreStore屏障，保证前面的对final写入对其他线程/CPU可见，并阻止重排序。
- 读final域：在上述规则2中，两步操作不能重排序的机理就是在读final域前插入了LoadLoad屏障。

X86处理器中，由于CPU不会对写-写操作进行重排序，所以StoreStore屏障会被省略；而X86也不会对逻辑上有先后依赖关系的操作进行重排序，所以LoadLoad也会变省略。


#### volatile
volatile是Java中的一个关键字。它主要是解决对共享变量访问时候的可见性问题和内存重排序问题。
在并发编程中，多个线程可以同时访问和修改共享的变量。然而，由于线程之间的执行顺序和优化机制的存在，有时可能会出现意外的结果或错误。这些问题包括可见性问题和内存重排序问题。
##### 实现原理
使用 "volatile" 关键字修饰的变量，底层使用**内存屏障**实现可见性和禁止指令重排序。
1. 可见性：  
    当一个线程修改了一个 volatile 变量的值时，这个新值会被立即写入主内存。而其他线程读取该变量时，会从主内存中获取最新的值而不是缓存中的旧值。这样可以确保所有线程对该变量的访问都能看到最新的值。
2. 禁止重排序：  
    使用 volatile 关键字修饰的变量会禁止编译器和处理器对其进行重排序，从而保证了操作的顺序性。
需要注意的是，volatile 并不能完全解决所有并发编程的问题。它主要用于确保可见性和禁止重排序，但并不能保证原子性。
##### volatile如何实现的内存可见性和禁止指令重排序呢？
内存屏障（memory barrier）：内存屏障是一种硬件或指令级别的机制，用于确保对内存的操作顺序和可见性。当一个线程写入一个 volatile 变量时，会在写操作之后插入一个内存屏障，将该写操作刷新到主内存中。这样可以保证其他线程在读取该变量时，从主内存中获取最新的值，而不是从本地缓存中获取。

另外，内存屏障还可以防止指令重排序。编译器和处理器在优化代码执行时，可能会对指令进行重排序，以提高性能。然而，这种重排序可能在多线程环境下引发问题。通过在 volatile 变量的写操作之后插入内存屏障，可以防止编译器和处理器对其进行重排序，确保操作的顺序性。

具体实现方式会因不同的体系结构和编译器而有所不同。例如，在x86架构上，编译器会使用`lock`指令来实现内存屏障，而在ARM架构上，会使用`dmb`（data memory barrier）指令。
##### 有了synchronized为什么还会有volatile关键字？::
volatile更加轻量级，他是无锁的一种实现。

##### 为什么有了MESI协议还要又volatile关键字? ::

https://blog.csdn.net/pengxurui/article/details/127932108

#### CAS

通过`compareAndSwap`，也就是`CAS`，来保证了对数据操作的原子性。先把当前值和底层的值（线程工作空间的值）进行比较，如果相等，也就是说该值未被其他线程改变，则执行更新的操作，否则那么就不停的循环判断。这是靠一条底层指令`cmpxchg`来实现的。

适用场景：并发不高，不需要阻塞，可以不上锁。

特点：不断比较更新，直到成功。

缺点
* 高并发的场景会导致一直自旋，cpu压力大；
* ABA问题

##### 底层原理
* 自旋 + UnSafe
* cmpxchg

##### cas存在什么问题

* ABA

CAS机制生效的前提是，取出内存中某时刻的数据，而在下时刻比较并替换。

如果在比较之前，数据发生了变化，例如：A->B->A，即A变为B然后又变化A，那么这个数据还是发生了变化，但是CAS还是会成功。

Java中CAS机制使用版本号进行对比，避免ABA问题，具体可以看`AtomicStampedReference`。

#### <a id = "无锁ThreadLocal"></a>ThreadLocal
[ThreadLocal](##ThreadLocal)

### 日常使用锁的最佳实践？::
* 减小锁的持有时间：
	>减小锁的持有时间是为了降低锁的冲突的可能性，提高体系的并发能力。
	- 只在必要时进行同步加锁操作
	- 只在必须加锁的代码段加锁
* 锁粒度的优化
	- 锁细化
	>比如ConcurrentHashMap的分段锁，提升吞吐量。	但是减小锁的粒度也带来了新的问题，当锁粒度过于小的时候，获取全局锁消耗的资源也相应增加，以 ConcurrentHashMap 为例，如果它需要获取当前的 size 就需要对每一个段都加锁。
	
	- 锁粗化
	 >在一般情况下，为了保证多线程之间的高效并发，会要求线程持有锁的时间尽量短，但是过度的细化会产生大量的申请和释放锁的操作，这对性能的影响也是非常大的。比如循环内的加锁操作。

*  锁分离
	- 读写分离锁替代独占锁
	>ReadWriteLock 使用读写分离锁来替代独占锁，它也是减小锁的粒度的一种方式，上面讲的是对数据结构层面的减小锁持有时间的，这里是根据业务来划分锁的持有，在读多写少的场景使用读写分离锁会大大提高系统的并发性能。
	- 重入锁和内部锁
		>重入锁的使用相较于内部锁更加复杂，重入锁必须手动显示释放锁，内部锁则可以自动释放，重入锁提供了一套提高性能的功能和 Condition 机制，重入锁可以设置锁的等待时间 boolean tryLock(long time)，锁中断 lockInterruptibly() 和快速锁轮询 tryLock() 等可以有效的避免死锁的产生。内部锁则是通过 wait() 和 notfiy() 实现锁的控制。
	- 自旋锁
	>自旋锁是 JVM 为了解决对多线程并发时频繁的挂起和恢复线程的操作问题的锁，当访问共享资源的时候，锁的等待时间可能很短，可能会比线程的挂起和恢复时间还要短，因此在这段时间里做线程的切换时不值得的。自旋锁可以使线程没有取得锁时不被挂起，而去执行一个空的循环，当线程获取了锁就会继续执行代码。
	>但是自旋锁只适用于线程竞争相对小、锁占用时间短的代码，对于锁竞争激烈的系统中不仅浪费了 CPU 资源，也免不了被挂起。JVM 可以设置自旋锁的开启和等待次数，防止一直执行空循环。
* 无锁
	* ThreadLocal
	* 原子类

## ForkJoin

## 线程
### 线程的生命周期

![](线程的生命周期.jpg)
![](线程生命周期流转.jpg)

> sleep和wait区别？
> 1、sleep是线程中的方法，但是wait是Object中的方法。
> 2、sleep方法不会释放lock，但是wait会释放，而且会加入到等待队列中。
> 3、sleep方法不依赖于同步器synchronized，但是wait需要依赖synchronized。
> 4、sleep不需要被唤醒，但是wait需要（不指定时间需要被别人中断）。
### Java一个线程占多大内存？::

默认1M，可以通过Thread的构造方法指定。

### Java线程占的内存是属于JVM还是操作系统内存？::
都有
### 主线程如何捕获子线程的异常？
1. 通过在主线程中设置`thread`对象的`UncaughtExceptionHandler`方法可以实现
2. 通过Future类也可以实现，`future.get()`会抛出异常

### 线程之间有哪些通信方式

- **volatile和synchronized关键字（锁）**
- **等待/通知机制（wait/notify）**
- **管道输入/输出流**
- **使用Thread.join()**
- **使用ThreadLocal**
## 线程池  
### 为什么要使用线程池？::
* 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。  
* 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。  
* 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。
	> 为什么要有线程池，是需要对线程管理，java中默认一个线程栈是1M，那么1024个请求就是1G内存，所以不对线程管理，很容易造成资源耗尽，程序崩溃
### 有哪些常见的线程池？::  
- newSingleThreadExecutor（单线程的线程池）  
- newFixedThreadPool（固定大小的线程池）  
- newCachedThreadPool（来一个任务做一个任务，会一直复用或创建线程）  
- newSingleThreadScheduledExecutor  

### 线程池有哪些核心参数以及其含义？::
- corePoolSize  
- maximumPoolSize  
- keepAliverTime：当活跃线程数大于核心线程数时，空闲的多余线程最大存活时间  
- unit：存活时间的单位  
- workQueue：存放任务的队列  
- handler：拒绝策略  
	- AbortPolicy：直接丢弃任务，抛出异常，这是默认策略  
	- CallerRunsPolicy：只用调用者所在的线程来处理任务  
	- DiscardOldestPolicy：丢弃等待队列中最旧的任务，并执行当前任务  
	- DiscardPolicy：直接丢弃任务，也不抛出异常    
> 编者注：线程池的各个参数与线程池的执行流程。
### 线程池的核心参数应该怎么设置？::
TODO

### 线程池如何保证核心线程不被销毁？::
核心线程会循环从队列中取任务，如果取不到任务则会在队列上阻塞等待而不会被销毁，而非核心线程拿不到任务的时候会结束循环，那意味着线程结束了。
> 编者注：
> * 如果是非核心线程会调用阻塞队列的`poll`方法，超时返回，销毁线程；如果是核心线程会调用阻塞队列的`take`方法，让worker线程挂起在阻塞队列上。
> * 挂起的操作底层是调用的Unsafe的park和unpark方法。


### 其他问题
线程池的状态有什么，如何记录的？  
线程池为什么添加空任务的非核心线程  
在没任务时，线程池中的工作线程在干嘛？  
工作线程出现异常会导致什么问题？  
工作线程继承AQS的目的是什么？


## ThreadLocal

### ThreadLocal的应用场景
* 线程隔离：
>当某个对象不是线程安全的，但又需要在多线程环境下使用时，可以将该对象存储在ThreadLocal中，使每个线程拥有独立的对象副本，避免了线程安全问题。
* 事务管理：
>在一些需要事务管理的应用中，可以使用ThreadLocal来存储数据库连接、事务对象等。这样可以确保每个线程都使用自己独立的数据库连接和事务，避免了线程间的干扰和数据一致性问题。
* 线程上下文传递：
>在Web应用中，可以使用ThreadLocal来存储当前用户的上下文信息，如用户ID、用户名等。这样可以在整个请求处理过程中方便地获取和使用用户相关的信息，而无需在方法参数中传递或使用全局变量。
* 线程池任务处理：
>在使用线程池执行任务时，可以使用ThreadLocal来存储任务相关的上下文信息。例如，可以使用ThreadLocal来存储任务的标识符、请求参数等，以便在任务执行过程中使用。
* 性能优化：
>有些计算密集型或资源消耗较大的操作，可以使用ThreadLocal来缓存中间结果，避免重复计算或资源的频繁获取和释放。这样可以提升性能和效率。

### ThreadLocal的实现原理::
`ThreadLocal`的实现原理可以简单概括为以下几点：
* 每个`Thread`对象中都有一个`ThreadLocalMap`对象，用于存储线程局部变量。`ThreadLocalMap`是`ThreadLocal`的内部类，它以`ThreadLocal`对象作为键，线程局部变量作为值存储在`HashMap`中。
* 当通过`ThreadLocal`的get()方法获取线程局部变量时，会先获取当前线程的`ThreadLocalMap`对象。然后，根据`ThreadLocal`对象作为键，在`ThreadLocalMap`中查找对应的值。
* 当通过`ThreadLocal`的set()方法设置线程局部变量时，会先获取当前线程的`ThreadLocalMap`对象。然后，使用`ThreadLocal`对象作为键，将线程局部变量存储在`ThreadLocalMap`中。
* 当线程结束时，`ThreadLocalMap`中的所有以`ThreadLocal`对象为键的条目会被自动清除，防止内存泄漏（弱引用）。
> ThreadLocal其实是操作Thread类中ThreadLocalMap变量的一个工具类，每个ThreadLocal相当于ThreadLocalMap中的一个entry。

![](threadlocal结构.png)

### ThreadLocal其他
* 解决hash冲突
	* 开放定址法，当前位置有了，就看下一个元素（类似环形数组），一定能找到位置，因为每次set都会去尝试扩容
* 扩容
	* 每次set后会检查是否到了扩容阈值，到了就扩容
	* 默认大小16，扩容是2倍的扩

> 编者注：关于ThreadLocal需要掌握的：
> * ThreadLocal应用场景，可以解决什么问题？
> * ThreadLocal底层实现原理
> * ThreadLocal解决hash冲突的方式


## 新特性
* Java 8
	* Lambda 表达式
	* Stream API
	* 新日期时间 API（java.time）
	* 接口默认方法与静态方法
	* 并发工具增强
		* CompletableFuture：链式异步任务编排
		* StampedLock：乐观读锁提升并发性能
	* 其他关键更新
		* Optional 类：优雅处理空值，避免NullPointerException
		* HashMap 优化
		* 安全性：默认启用TLS 1.2，支持ECC加密算法
* Java 11
	* 局部变量类型推断（var）
		* 仅限局部变量：`var list = new ArrayList<String>();`
	* HTTP/2 客户端
		* 原生支持异步HTTP/2请求，替代HttpURLConnection。
	* 字符串 API 增强
		* `" ".isBlank() → true`，`"a\nb".lines().count() → 2`
	* ZGC 垃圾收集器
		* 亚毫秒级暂停，适合大内存应用（如数据分析系统）
* Java 17
	* 文本块（Text Blocks）
	  
		```java
		String json = """
		    { 
		      "name": "Java", 
		      "version": 17 
		    }
		""";
		```
	* 密封类（Sealed Classes）
	  ```java
	  // 限制子类范围
	  public sealed class Shape permits Circle, Square { ... }  
		```
	* 模式匹配 `instanceof`
	  ```java
	  if (obj instanceof String s) {
			System.out.println(s.length());  // 自动类型转换
	  }
		```
* Java 21
	* 虚拟线程（Virtual Threads）
	    - 轻量级线程（协程），支持百万级并发，替代线程池
    - 结构化并发
	    - 多任务原子化管理，避免线程泄漏
	- 记录模式（Record Patterns）
	  ```java
		record Point(int x, int y) {}
		if (p instanceof Point(int x, int y)) {
		    System.out.println(x + y);  // 解构Record类
		}
		```
	* 字符串模板
	  ```java
		String name = "Alice";
		String message = STR."Hello, \{name}!";  // 替代String.format()
		```
* Java 24-25
	* 简单源文件与实例主方法
  ```java
	  void main() {  // 无public static修饰
	    println("Hello, World!");
	  }
	```
	* Scoped Values
		* 替代`ThreadLocal`，安全共享不可变数据（虚拟线程友好）
	* 模块导入声明
		* 简化包导入：`import java.xml.*;` → 单行模块声明
	
	