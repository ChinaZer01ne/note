1. 线程的生命周期
2. Thread Group
3. JVM运行时数据区，哪些是线程私有的
4. 如何优雅的停止线程，如何强制关掉线程
5. 为什么不加volatile就会一直卡死呢
6. join方法
7. 多线程死锁分析
8. 生产者消费者模型
9. 多生产者多消费者中 wait、notify的问题
10. 为什么生产消费模型中，条件判断用while不用if
11. sleep和wait的区别
12. Runtime钩子程序配合线程使用
13. 线程中如何捕获异常，不同的try-catch无法捕获
14. 自定义个boolean锁
15. 线程池线程管理 ： 自动扩容+拒绝策略 + 闲时回收
16. waitset
	1. 1、所有对象都有一个waitset，用来存放调用了该对象的wait()方法之后进入block状态的线程
	2.  2、线程进入block状态后，需要其它线程调用notify()方法进行唤醒。被唤醒后需要重新排队获取锁，因此不会立马执行
	3.  3、线程从waitset中被唤醒的顺序不一定是FIFO
	4.  4、线程被唤醒后，不会立马被执行，需要排队抢锁。线程wait后会进行代码地址记录，当线程被唤醒抢到锁后会从记录的地址继续执行。
	5. 为什么wait要在同步代码块里，因为，waitset存放等待的线程，waitset在对象上，所以要对对象加锁，才知道对哪个waitset里的线程进行阻塞或唤醒？
17. **volatile关键字在多线程三个特性角度：**  
	1. **一旦一个共享变量被volatile修饰，具备以下含义：**  
	2. **1.保证了不同线程间的可见性**  
	3. **2.禁止对其进行重排序，也就是保证了有序性**  
	4. **3.并未保证原子性
18. ****volatile关键字实质：**  
	1. **1.保证重排序的时候不会把后面的指令放到屏障的前面，也不会把前面的放到后面**  
	2. **2.强制对缓存的修改操作会立刻写入主存**  
	3. **3.如果是写操作，它会导致其他CPU中的缓存失效**
19. 一个线程如果对变量只有读操作，没有写操作，由于JMM的原因，那这个变量不会直接刷到主内存
20. JMM
21. 解决缓存不一致的问题
	1. 总线锁
	2. 缓存一致性协议
22. 原子性：8中原子性操作，注意a = b不是原子性，虽然是复制但是是两条指令read b; assign a;
23. 可见性：volatile， 锁
24. 有序性：happens-before原则
25. A a = new A(); 这行代码中的引用、对象、class对象都在内存中的什么位置?
	1. a在栈中
	2. A对象在堆中
	3. A.class对象在堆中，这个对象作为访问方法区数据的外部接口
	4. A的类元数据在方法区中
26. future设计
27. 单线程的指令重排序和多线程的指令重排序
	1. 单线程的指令重排序不会对结果造成影响
	2. 多线程下发生指令重排序可能会造成错误的结果
28. 线程上下文加载器打破双亲委派机制
29. 多线程设计模式
	1. 观察者
	2. 单线程设计模式
	3. 读写锁分离
	4. 不可变对象
	5. Future设计模式
	6. Guarded suspension
	7. Threadlocal
	8. 多线程上下文设计模式
	9. balking设计模式
	10. 生产消费模式
	11. count down
	12. thread-per-message
	13. two phase temination
	14. work thread
		1. Worker-Thread 和 Producer-Consumer设计模式区别不是很大，Worker-Thread中用Channel封装了queue和消费线程
	15. 多线程active objects
30. 原子类
31. cas存在的问题，aba
32. unsafe
	1. 底层的CPU指令
		1. cmpxchg1：compareAndSwapInt
		2. cmpxchg：compareAndSwapLong
		3. xchg1：putOrderedInt
		4. cmpxchgq：compareAndSwapObject
		5. lock1：volatile
		6. membar-acquire
33. exchanger：只适合2个线程