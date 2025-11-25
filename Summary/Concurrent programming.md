## CompletableFuture

CompletableFuture‌是Java 8中引入的一个类，它**实现了Future和CompletionStage接口，用于支持异步编程和非阻塞操作**。CompletableFuture提供了丰富的API，使得并发任务的处理变得简单而高效，能够轻松创建、组合和链式调用异步操作，无需关心底层线程管理，从而提升了程序的响应速度和资源利用率‌。
> * 它是 `Future` 的超强进化版，代表一个异步计算的结果，但提供了异常灵活的组合、链式调用和异常处理能力。
> * 其设计的目的是为了实现非阻塞的异步编程模型。
> * `CompletableFuture` 实现了 `CompletionStage` 接口，该接口定义了大量组合异步操作的方法（`thenApply`, `thenAccept`, `thenRun`, `thenCompose`, `thenCombine`, `handle`, `exceptionally` 等）。

### 创建 CompletableFuture
- `CompletableFuture.supplyAsync(Supplier<U> supplier)`: 异步执行一个有返回值的任务（常用）。
- `CompletableFuture.supplyAsync(Supplier<U> supplier, Executor executor)`: 指定自定义线程池执行。
- `CompletableFuture.runAsync(Runnable runnable)`: 异步执行一个无返回值的任务。
- `CompletableFuture.runAsync(Runnable runnable, Executor executor)`: 指定自定义线程池执行。
- `CompletableFuture.completedFuture(U value)`: 创建一个已经完成并包含给定结果的 `CompletableFuture`。
### 结果处理与链式调用 (最核心)
- `thenApply(Function<T, U> fn)`: 对**上一步结果**进行**同步**转换，返回新的 `CompletionStage`。
- `thenAccept(Consumer<T> action)`: 消费**上一步结果**（无返回值），返回 `CompletionStage<Void>`。
- `thenRun(Runnable action)`: 在上一步完成后执行一个动作（不关心上一步结果），返回 `CompletionStage<Void>`。
- 异步变体： `thenApplyAsync`, `thenAcceptAsync`, `thenRunAsync` - 在新线程中执行转换/消费/动作（默认使用 `ForkJoinPool` 或指定线程池）。理解 `...Async` 后缀的含义是关键！
### 组合多个 Future
- `thenCompose(Function<T, CompletionStage<U>> fn)`: **扁平化**。当前阶段完成后，使用其结果**异步触发**另一个 `CompletionStage`（避免嵌套 `CompletableFuture<CompletableFuture<U>>`）。
- `thenCombine(CompletionStage<U> other, BiFunction<T, U, V> fn)`: 等待**当前阶段**和 **`other` 阶段**都完成后，将两者的结果进行**同步**合并处理。
- `allOf(CompletableFuture<?>... cfs)`: 等待**所有**给定的 `CompletableFuture` 完成。返回 `CompletableFuture<Void>`。**注意：需要手动收集结果。**
- `anyOf(CompletableFuture<?>... cfs)`: 等待**任意一个**给定的 `CompletableFuture` 完成。返回 `CompletableFuture<Object>`（结果是第一个完成的那个的结果）。
### 异常处理 (极其重要)
- `exceptionally(Function<Throwable, T> fn)`: 像一个 `catch` 块。当链中**之前的阶段**抛出异常时，捕获异常并提供一个**恢复值**（或抛出新的异常）。返回新的 `CompletionStage`。
- `handle(BiFunction<T, Throwable, U> fn)`: 像一个 `finally` 块（但能返回结果）。无论之前的阶段正常完成还是异常结束，都会被调用。`fn` 的第一个参数是结果（正常时为值，异常时为 `null`），第二个参数是异常（正常时为 `null`）。**更通用，可以同时处理结果和异常。**
### 结果获取 (谨慎使用)
- `get()`: 阻塞当前线程直到结果可用或抛出异常（`ExecutionException`, `InterruptedException`）。**容易破坏异步性，只在必须同步获取结果的地方用（如 `main` 方法结尾）**。
- `get(long timeout, TimeUnit unit)`: 带超时的 `get`。
- `join()`: 类似 `get()`，但**不会抛出受检异常 `InterruptedException`**，只抛出 `CompletionException`（包装了原始异常）。在 `CompletableFuture` 链内部或明确知道不需要处理中断时更方便。**同样会阻塞！**
### 完成控制 (手动设置结果/异常)

- `complete(T value)`: 如果尚未完成，则手动设置结果并完成 Future。
- `completeExceptionally(Throwable ex)`: 如果尚未完成，则手动设置异常并完成 Future（异常结束）。
- `obtrudeValue(T value)` / `obtrudeException(Throwable ex)`: 强制设置结果或异常，**即使已经完成**。**慎用！**
### 线程池管理
- 理解默认使用 `ForkJoinPool.commonPool()`。
- **重要实践：** **为 I/O 密集型或需要隔离的任务显式指定自定义线程池 (`Executor`)**，避免阻塞公共池影响其他任务。这是生产环境中的常见优化点。
### 最佳实践

####  务必自定义线程池

- **根据任务类型配置线程池**：
    ```java
    // 示例：专用 I/O 线程池
    ExecutorService ioExecutor = Executors.newFixedThreadPool(50);
    CompletableFuture.supplyAsync(() -> httpCall(), ioExecutor);
    ```
- **拒绝策略必用 `AbortPolicy`**：线程池满时立即抛 `RejectedExecutionException`，避免隐式丢弃任务导致 `get()` 阻塞。

#### 用链式调用替代 `get()`/`join()`

- **优先使用非阻塞方法**：
	```java
    // 反例：阻塞调用
    String result = future.get(5, SECONDS); // 仍可能超时阻塞
    
    // 正例：链式处理
    future.thenApply(res -> process(res))
          .exceptionally(ex -> fallback());
    ```
    
- **必须超时控制**：
    - 对 I/O 任务使用 `orTimeout()`：
	```java
	future.orTimeout(2, SECONDS) // 超时自动取消任务
		  .exceptionally(ex -> "timeout");
	```

#### 线程池隔离与资源限制

- **关键操作独立线程池**：
    
	```java
    // 耗时任务与快任务隔离
    ExecutorService slowPool = Executors.newFixedThreadPool(10);
    ExecutorService fastPool = Executors.newFixedThreadPool(20);
    
    CompletableFuture.runAsync(() -> heavyTask(), slowPool);
    CompletableFuture.runAsync(() -> lightTask(), fastPool);
    ```
    
- **控制并发量**：通过 `Semaphore` 或 `CompletableFuture[]` 限制最大并行任务数，防止突发流量击溃线程池。

#### 异常处理与状态监控

- **链式捕获异常**：
	```java
    future.handle((res, ex) -> {
        if (ex != null) log.error("Failed", ex);
        return res != null ? res : defaultValue;
    });
    ```
    
- **监控线程池状态**：记录队列堆积、拒绝次数等指标，结合告警快速定位瓶颈。

#### 其他高级技巧

- **批量任务优化**：
    - 使用 `allOf()` 合并多个 Future，但避免在其后直接调用 `join()`（仍阻塞）。
    - 改用 `thenCompose()` 实现非阻塞依赖：
		```java
        future1.thenCompose(res1 -> future2(res1)) // future2 依赖 future1
               .thenAccept(finalRes -> ...);
        ```
- **单元测试避坑**：  
    用 Mockito 模拟线程池行为，避免测试阻塞：
	```java
    // 强制同步执行异步任务
    doAnswer(inv -> ((Runnable) inv.getArgument(0)).run())
       .when(executor).execute(any(Runnable.class));
    ```
       
### 面试深入 (理解原理、细节、陷阱、最佳实践)

1. **与 Future 的区别：**
    - `Future` 只能 `get()` 阻塞等待结果或轮询 `isDone()`，组合能力极弱。
    - `CompletableFuture` 提供强大的非阻塞回调、链式组合、异常处理。
2. **CompletionStage 接口的设计哲学：**
    - 理解其方法分类（转换、消费、运行、组合、异常处理）和返回类型。
    - 理解 `...Async` 方法的意义（在新线程中执行）。
3. **回调链的执行与线程模型：**
    - 理解链式调用中每个阶段（`thenXxx`）默认在哪个线程执行（通常是完成前一个阶段的线程，除非使用 `...Async`）。
    - **陷阱：** 如果在回调链中执行了**长时间阻塞的操作**（如同步 I/O），而没有使用 `...Async` 切换到新线程，会阻塞当前线程（可能是公共池线程），影响系统性能。**解决方案：** 将阻塞操作包装在 `supplyAsync/runAsync` 或使用 `thenXxxAsync` 中，并指定合适的线程池。
4. **异常传播：**
    - 理解异常如何在回调链中传播：如果某个阶段抛出异常，后续的 `thenApply`/`thenAccept` 等**不会执行**，而是直接跳到链中后续的 `exceptionally` 或 `handle` 方法。
    - `handle` 和 `exceptionally` 的区别和适用场景。
5. **组合操作 (`thenCompose` vs `thenCombine` vs `allOf`/`anyOf`)：**
    - 深入理解 `thenCompose` 解决嵌套 Future 问题的原理（类似 `flatMap`）。
    - 理解 `thenCombine` 用于并行执行两个独立任务并合并结果。
    - 掌握 `allOf` 后如何高效收集所有结果（例如，遍历原始数组，对每个 future 调用 `join()`）。
    - `anyOf` 的使用场景和结果处理。
6. **默认线程池 (`ForkJoinPool.commonPool()`) 的优缺点：**
    - 优点：方便，避免手动创建线程池。
    - 缺点：
        - 共享资源，一个任务阻塞可能影响其他任务（特别是公共池上的任务）。
        - 配置受限（并行度取决于处理器核心数）。
        - 不适合 I/O 密集型任务（线程会被阻塞）。
    - **最佳实践：** 为 I/O 操作或需要资源隔离的任务**总是使用自定义线程池**。
7. **手动完成 (`complete`, `completeExceptionally`) 的应用场景：**
    - 超时控制（启动一个定时任务，超时则 `completeExceptionally(new TimeoutException())`）。
    - 响应外部事件。
    - 模拟结果进行测试。
8. **与响应式编程 (Reactive Streams, RxJava, Project Reactor) 的关系：**
    - `CompletableFuture` 可以看作是处理**单个异步值**的简化版响应式编程。
    - 响应式编程框架（如 Reactor 的 `Mono`）提供了更强大的**背压(Backpressure)**、**丰富的操作符**和**处理流**的能力。了解 `CompletableFuture` 是理解响应式的基础。
9. **常见陷阱：**
    - **阻塞回调链：** 在非 `...Async` 的回调中执行阻塞操作。
    - **忘记异常处理：** 链中没有 `exceptionally` 或 `handle`，导致异常被吞没，难以调试。
    - **滥用 `get()`/`join()`：** 在异步流程内部不必要地阻塞线程。
    - **线程池选择不当：** I/O 任务使用默认的 `ForkJoinPool.commonPool()`。
    - **循环引用或长链导致的资源泄漏：** 虽然 JVM 能处理，但设计时应避免不必要的长链或循环依赖。
10. **Java 8+ 的增强：**
    - `completeOnTimeout(T value, long timeout, TimeUnit unit)`: 超时后提供默认值。
    - `orTimeout(long timeout, TimeUnit unit)`: 超时后抛出 `TimeoutException`。
    - `defaultExecutor()`: 获取默认的异步执行器（通常是 `ForkJoinPool.commonPool()`）。
    - `copy()`: Java 12+，创建一个新的、独立的副本（结果相同）。
    - `minimalCompletionStage()`: Java 12+，返回一个只读的 `CompletionStage` 视图。
11. **底层原理 (加分项)：**
    - 了解其内部大致使用**栈**（Treiber Stack）来管理依赖（后续阶段）。
    - 理解 `Completion` 对象（如 `UniApply`, `UniAccept` 等）代表链中的每个阶段节点。
    - 知道它如何实现非阻塞的完成通知（通过 CAS 操作和栈的弹出执行）。
**总结关键点：**

- **日常：** 熟练创建 (`supplyAsync/runAsync`)、链式调用 (`thenApply/Accept/Run[Async]`)、组合 (`thenCompose/thenCombine/allOf/anyOf`)、异常处理 (`exceptionally/handle`)、**务必使用自定义线程池处理阻塞操作**、谨慎使用 `get/join`。
- **面试：** 深入理解其与 `Future` 的区别、线程模型（回调执行线程、默认线程池问题）、异常传播机制、组合操作原理 (`thenCompose` 解嵌套)、常见陷阱（阻塞回调、未处理异常）、手动完成的应用、以及与响应式编程的关系。能清晰解释为什么需要自定义线程池。

## Callable
`Callable` 是 Java 并发编程中另一个**极其重要**的基础接口，与 `Runnable` 类似，用于表示**一个需要执行的任务**。但它比 `Runnable` 更强大，主要体现在以下两点：
1. **返回值：** `Callable` 的 `call()` 方法可以**返回一个结果**（类型由泛型参数 `V` 指定）。
2. **抛出异常：** `call()` 方法**可以抛出受检异常 (`Exception`)**，调用方可以捕获并处理这些异常。

### 日常开发使用

1. **定义与核心方法：**
    
    - `interface Callable<V> { V call() throws Exception; }`
    - 你需要实现 `call()` 方法，在其中编写任务逻辑，并返回一个类型为 `V` 的值（如果不需要返回值，可以返回 `null`）。
    - 可以在 `call()` 方法中抛出任何 `Exception`（包括受检异常）。
        
2. **创建 Callable 任务：**
```java
    // 方式1：显式实现类
    Callable<Integer> myTask = new Callable<>() {
        @Override
        public Integer call() throws Exception {
            // 模拟耗时计算
            Thread.sleep(1000);
            return 42; // 返回计算结果
        }
    };
    
    // 方式2：Lambda 表达式 (最常用)
    Callable<Integer> myTask = () -> {
        Thread.sleep(1000);
        return 42;
    };
    
    // 方式3：方法引用 (如果逻辑在另一个方法里)
    public Integer computeAnswer() throws Exception { ... }
    Callable<Integer> myTask = this::computeAnswer;
```
    
    
3. **执行 Callable 任务：**
    
    - **核心机制：通过 `ExecutorService` 提交。**
        
    - `ExecutorService` 提供了提交 `Callable` 并获取 `Future` 的方法：
        - `Future<V> submit(Callable<V> task)`: 提交任务，返回一个代表其异步结果的 `Future<V>`。
        - `List<Future<V>> invokeAll(Collection<? extends Callable<V>> tasks) throws InterruptedException`: 提交一组任务，等待**所有**任务完成，返回包含它们 `Future` 的列表。
        - `List<Future<V>> invokeAll(Collection<? extends Callable<V>> tasks, long timeout, TimeUnit unit) throws InterruptedException`: 带超时的 `invokeAll`。
        - `V invokeAny(Collection<? extends Callable<V>> tasks) throws InterruptedException, ExecutionException`: 提交一组任务，返回**任意一个**成功完成的任务的结果（一旦有结果就返回，取消其他未完成的任务）。
        - `V invokeAny(Collection<? extends Callable<V>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException`: 带超时的 `invokeAny`。
    
```java
	ExecutorService executor = Executors.newFixedThreadPool(2);
    
    // 提交单个任务
    Future<Integer> futureResult = executor.submit(myTask);
    
    // 提交一组任务
    List<Callable<Integer>> tasks = ...;
    List<Future<Integer>> allFutures = executor.invokeAll(tasks);
    
    // 获取任意一个结果
    Integer firstResult = executor.invokeAny(tasks);
```
    
4. **获取结果与处理异常 (通过 Future)：**
    
    - 提交任务返回的 `Future<V>` 对象是你与异步任务交互的**句柄**。
    - 关键方法：
        - `V get() throws InterruptedException, ExecutionException`: **阻塞**当前线程直到任务完成，然后返回结果。如果任务抛出异常，`ExecutionException` 会包装该异常。
        - `V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException`: 带超时的 `get()`。
        - `boolean isDone()`: 查询任务是否已完成（正常完成、被取消或异常结束）。
        - `boolean cancel(boolean mayInterruptIfRunning)`: 尝试取消任务。
        - `boolean isCancelled()`: 查询任务是否在正常完成前被取消。
    
	```java
		try {
	        Integer result = futureResult.get(); // 阻塞等待结果
	        System.out.println("Result: " + result);
	    } catch (InterruptedException e) {
	        // 当前线程在等待时被中断
	        e.printStackTrace();
	    } catch (ExecutionException e) {
	        // 任务本身执行时抛出了异常，通过 e.getCause() 获取原始异常
	        Throwable cause = e.getCause();
	        if (cause instanceof MySpecificException) {
	            // 处理特定业务异常
	        } else {
	            // 处理其他异常
	        }
	    }
	```
    
5. **与 Runnable 的区别 (日常必须清楚)：**
    
    |特性|`Callable<V>`|`Runnable`|
    |---|---|---|
    |**返回值**|**有** (`V`)|**无** (`void`)|
    |**抛出异常**|**可以**抛出 `Exception` (受检异常)|**不能**抛出受检异常 (只能运行时异常)|
    |**执行方法**|`call()`|`run()`|
    |**ExecutorService**|`submit()` 返回 `Future<V>`|`submit()` 返回 `Future<?>` (可调用 `get()`, 但总是 `null`)|
    |**Lambda**|`() -> { ... return value; }`|`() -> { ... }`|
    

###  面试深入

1. **为什么需要 Callable？**
    
    - 解决 `Runnable` 的两大局限：无法返回计算结果、无法方便地传播受检异常。`Callable` + `Future` 提供了**获取异步任务结果和异常**的标准机制。
        
2. **Future 接口的作用与局限性：**
    
    - **作用：** `Future` 是 `Callable`/`Runnable` 任务提交后返回的**异步计算结果的句柄**。提供查询状态、取消任务、**阻塞获取结果**的方法。
        
    - **局限性 (催生 CompletableFuture)：**
        
        - **阻塞性：** `get()` 方法是阻塞的，容易导致线程挂起，降低并发效率。
        - **组合能力弱：** 难以优雅地组合多个异步任务（如任务A完成后触发任务B，任务A和B都完成后合并结果）。需要手动轮询或回调嵌套，代码复杂。
        - **异常处理不便：** 只能通过 `ExecutionException` 包装获取原始异常，处理逻辑可能冗长。
        - **手动完成困难：** 标准 `Future` 接口没有提供手动设置结果/异常的方法（`CompletableFuture` 提供了 `complete`/`completeExceptionally`）。
            
3. **Callable 与 Future 的关系：**
    
    - `ExecutorService.submit(Callable task)` **将 `Callable` 包装成一个异步执行的任务单元**，并**立即返回一个 `Future` 对象**。
    - 这个 `Future` 对象**代表 `Callable` 任务最终的执行结果（或异常）**。
    - 你可以通过 `Future` 的方法**在未来的某个时间点**获取这个结果或检查状态。`Future` 是 `Callable` 执行结果的“凭证”或“占位符”。
        
4. **invokeAll 和 invokeAny 的工作原理与使用场景：**
    - **invokeAll:**
        - **原理：** 内部提交所有任务，使用 `CountDownLatch` 或类似机制等待所有任务完成（或超时）。
        - **使用场景：** 需要并行执行一批独立任务，并**等待所有任务都完成**后再继续处理（如并行查询多个数据源并聚合结果）。
        - **结果获取：** 返回的 `List<Future<V>>` 顺序与提交的 `Callable` 集合顺序一致。遍历列表，对每个 `Future` 调用 `get()` 获取结果（注意处理异常）。
    - **invokeAny:**
        - **原理：** 提交所有任务，使用类似 `CompletionService` 的机制（通常基于阻塞队列），**只要有一个任务成功完成，就立即返回其结果并尝试取消其他所有未完成的任务**。如果所有任务都失败，抛出 `ExecutionException`（包装最后一个任务的异常）。
        - **使用场景：** 需要并行执行多个等效任务（如访问多个冗余服务），**只要其中一个成功返回结果即可**（如获取最快的服务响应）。
        - **结果：** 直接返回第一个成功任务的结果值 `V`。**注意：** 被取消的任务可能还在运行中被中断（取决于 `cancel(true)` 的效果）。
5. **异常处理最佳实践：**
    - 在 `Callable.call()` 方法中，应抛出具体的业务异常或标准异常，不要仅仅打印日志或吞掉。
    - 在调用 `Future.get()` 时，**必须捕获 `InterruptedException` 和 `ExecutionException`**。
    - **关键：** 通过 `ExecutionException.getCause()` **获取 `call()` 方法中抛出的原始异常**，进行具体的业务处理（重试、降级、告警等）。
    - 考虑在 `Callable` 内部进行必要的 `try-catch`，将非受检异常转换为有意义的受检异常或包装后再抛出，提供更清晰的错误信息。
6. **线程池的选择：**
    - 与 `CompletableFuture` 类似，**为 `Callable` 任务选择合适的 `ExecutorService` 至关重要**。
    - CPU 密集型：`Executors.newFixedThreadPool(nThreads)` (nThreads ≈ CPU核心数)。
    - I/O 密集型：`Executors.newCachedThreadPool()` 或 `newFixedThreadPool` (设置更大的线程数，或使用专为 I/O 优化的线程池如 `ThreadPoolExecutor` 配合合适的 `BlockingQueue`)。
    - **避免无界队列：** 防止任务堆积导致 OOM。
    - **重要：** 避免在 `Callable` 任务内部执行**长时间阻塞操作**而不释放线程池线程（如长时间同步 I/O）。如果必须阻塞，确保线程池足够大或使用异步 I/O/NIO。
        
7. **与 CompletableFuture 的结合：**
    
    - `CompletableFuture` 提供了更强大的异步编程能力，但它底层仍然离不开 `ExecutorService` 和任务单元 (`Runnable`/`Callable`)。
    - 可以将传统的 `Callable` 任务包装成 `CompletableFuture`：
      
	```java
		CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> {
			try {
				return myCallable.call(); // 调用 Callable 的 call 方法
			} catch (Exception e) {
				throw new CompletionException(e); // 将受检异常包装为非受检异常
			}
		}, executor);
	```
- 这样就能利用 `CompletableFuture` 的链式调用、组合、非阻塞回调等高级特性来处理原本由 `Callable` 定义的任务逻辑和结果。

### 总结关键点

- **日常：**
    
    - 掌握 `Callable` 的定义（`call()` 方法返回 `V` 且可抛 `Exception`）。
    - 熟练使用 Lambda 创建 `Callable`。
    - **核心：** 通过 `ExecutorService.submit(Callable)` 提交任务并获取 `Future<V>`。
    - **必须：** 使用 `Future.get()` (带异常处理) 或 `isDone()` 查询结果。
    - 理解 `invokeAll` (等所有) 和 `invokeAny` (等任意一个) 的使用场景。
    - 清晰区分 `Callable` (有返回/抛异常) 与 `Runnable` (无返回/不抛受检异常)。
    - **务必**为任务选择合适的线程池 (`ExecutorService`)。
- **面试：**
    
    - 深刻理解 `Callable` 解决的 `Runnable` 的痛点（返回值、异常）。
    - 理解 `Future` 的作用（异步结果句柄）及其核心方法 (`get`, `isDone`, `cancel`)。
    - 清晰阐述 `Future` 的局限性（阻塞、组合弱），引出 `CompletableFuture` 的必要性。
    - 掌握 `invokeAll` 和 `invokeAny` 的工作原理、使用场景和区别。
    - **重点：** `Callable` 异常处理流程（`call()` 抛出 -> `Future.get()` 抛出 `ExecutionException` -> `getCause()` 获取原始异常）。
    - 理解线程池选择策略及其对 `Callable` 任务执行的影响。
    - 了解如何将 `Callable` 与 `CompletableFuture` 结合使用。

**简单来说：** `Callable` 定义了**做什么任务**（有结果，可能出错），`ExecutorService` 负责**调度和执行**这个任务，`Future` 提供了**监控和获取这个任务最终结果（或错误）** 的手段。它们是 Java 并发编程中执行异步计算任务的基础设施，而 `CompletableFuture` 是在此基础上构建的更高级的异步编程抽象。掌握 `Callable` 是理解 Java 并发模型和后续学习 `CompletableFuture` 及响应式编程的基石。

## FutureTask
`FutureTask` 是 Java 并发包 (`java.util.concurrent`) 中一个**非常重要且实用**的类，它充当了 `Runnable`, `Future`, 和 `Callable` 之间的**桥梁**。理解 `FutureTask` 对于深入掌握 Java 并发模型、任务执行机制以及应对相关面试至关重要。

###  核心概念与角色

1. **是什么？**
    - `FutureTask` 是一个**可取消的异步计算任务**。
    - 它实现了 `Future<V>` 接口（提供查询状态、取消任务、获取结果的能力）。
    - 它实现了 `Runnable` 接口（意味着它可以被提交给 `Executor` 或 `Thread` 直接执行）。
    - 它**包装**了一个 `Callable<V>` 或 `Runnable`（内部会将 `Runnable` 适配成 `Callable`）。
    - 它在内部管理任务的执行状态（未启动、运行中、已完成）和结果（或异常）。
2. **核心作用：**
    
    - **将 `Callable` 或 `Runnable` 任务封装成一个 `RunnableFuture`（同时是 `Runnable` 和 `Future`）：** 这使得任务既能被线程/线程池执行（`Runnable`），又能提供对任务执行结果和状态的控制（`Future`）。
    - **提供任务状态管理和结果存储：** 它维护任务的生命周期状态（`NEW`, `COMPLETING`, `NORMAL`, `EXCEPTIONAL`, `CANCELLED`, `INTERRUPTING`, `INTERRUPTED`），并在任务完成后安全地存储结果或异常。
    - **实现可取消性 (`cancel`):** 允许在任务完成前尝试取消它。
    - **保证结果发布的可见性：** 使用内部状态机和可能的同步机制（如 `LockSupport`, `volatile` 变量）确保一个任务的结果或异常一旦设置，对所有后续调用 `get()` 的线程都是可见的。

###  日常开发使用

1. **创建 FutureTask：**
    
```java
// 1. 用 Callable 创建 (最直接)
    Callable<Integer> callable = () -> { ... return 42; };
    FutureTask<Integer> futureTask1 = new FutureTask<>(callable);
    
    // 2. 用 Runnable 和结果值创建 (将 Runnable 适配成 Callable)
    Runnable runnable = () -> { ... }; // 无返回值操作
    Integer resultValue = null; // 或预设的默认值
    FutureTask<Integer> futureTask2 = new FutureTask<>(runnable, resultValue);
```
    
2. **执行 FutureTask：**
    - **方式 1：提交给线程池 (`ExecutorService`)** (推荐)
	```java
	ExecutorService executor = Executors.newSingleThreadExecutor();
	executor.execute(futureTask); // 利用其 Runnable 特性
	// 或者，虽然 submit(Runnable) 返回 Future<?>，但用 FutureTask 本身更直接
	// Future<?> future = executor.submit(futureTask);
	```
	- **方式 2：直接启动新线程执行**
	```java
	Thread thread = new Thread(futureTask); // 利用其 Runnable 特性
	        thread.start();
	```
3. **获取结果与状态控制：** (通过 `Future` 接口)
    ```java
    // 1. 阻塞获取结果 (和 Future 一样)
    try {
        Integer result = futureTask.get(); // 阻塞直到完成或异常
        // 或带超时
        // Integer result = futureTask.get(5, TimeUnit.SECONDS);
    } catch (InterruptedException | ExecutionException e) {
        // 处理中断或任务执行异常 (e.getCause() 获取原始异常)
    } // catch (TimeoutException e) { ... } // 如果用了带超时的 get
    
    // 2. 查询状态
    boolean isDone = futureTask.isDone(); // 任务是否完成 (正常、取消、异常都算完成)
    boolean isCancelled = futureTask.isCancelled(); // 任务是否在完成前被取消
    
    // 3. 尝试取消任务
    boolean mayInterruptIfRunning = true; // true: 尝试中断正在执行任务的线程; false: 仅标记取消
    boolean cancelSuccess = futureTask.cancel(mayInterruptIfRunning);
    ```
4. **手动运行（通常不推荐，了解即可）：**
    - 可以直接调用 `futureTask.run()`。这会在**当前线程**中同步执行任务（而不是异步）。通常只在特定场景（如自定义调度）或测试中使用。

### 面试深入 (原理、设计、场景、陷阱)

1. **与 `Future` / `Callable` / `Runnable` 的关系：**
    - `FutureTask` 是 `RunnableFuture<V>` 的标准实现，而 `RunnableFuture<V>` 扩展了 `Runnable` 和 `Future<V>`。
    - 它**适配器模式**的体现：将 `Callable` 或 `Runnable + 结果` 包装成一个统一的、可执行且可获取结果/状态的对象。
    - **为什么需要它？** `ExecutorService.submit(Callable)` 内部其实就是创建了一个 `FutureTask` 来包装你的 `Callable`，然后执行它并返回这个 `FutureTask`（作为 `Future`）。它提供了任务执行、状态管理和结果获取的基础设施。
2. **内部状态机：**
    - `FutureTask` 使用一个 `volatile int state` 变量来表示任务的状态，状态变迁是单向的：
        - `NEW` (0)：新建，任务尚未执行。
        - `COMPLETING` (1)：任务执行即将完成（结果或异常正在被设置）。**瞬时状态**。
        - `NORMAL` (2)：任务**正常完成**，结果已设置。
        - `EXCEPTIONAL` (3)：任务因**抛出异常**而完成。
        - `CANCELLED` (4)：任务在**未启动时被取消** (`cancel(false)` 或 `cancel(true)` 但任务未启动)。
        - `INTERRUPTING` (5)：任务**正在被中断** (`cancel(true)` 且任务正在运行)。**瞬时状态**。
        - `INTERRUPTED` (6)：任务**已被中断** (`cancel(true)` 成功中断了运行中的任务)。
    - 状态变迁路径 (`NEW -> ...`):
        - `NEW -> COMPLETING -> NORMAL` (正常完成)
        - `NEW -> COMPLETING -> EXCEPTIONAL` (异常完成)
        - `NEW -> CANCELLED` (未启动被取消)
        - `NEW -> INTERRUPTING -> INTERRUPTED` (运行中被中断取消)
    - **关键点：** 状态机保证了操作的原子性和结果的可见性。`get()` 方法会根据状态决定是阻塞等待、立即返回结果还是抛出异常。
3. **`run()` 方法的核心逻辑：**
    1. 检查状态必须是 `NEW`，并通过 CAS 确保只有一个线程能执行任务。
    2. 调用封装的 `Callable.call()`（或适配后的 `Runnable.run()`）。
    3. 如果调用成功，设置结果 (`set(result)`)，状态从 `NEW` 经 `COMPLETING` 变为 `NORMAL`。
    4. 如果调用抛出异常，设置异常 (`setException(ex)`)，状态从 `NEW` 经 `COMPLETING` 变为 `EXCEPTIONAL`。
    5. **无论如何完成，最终都会调用 `finishCompletion()`：** 唤醒所有在 `get()` 上阻塞的线程，并执行注册的后续操作（`CompletableFuture` 用这个机制实现回调链）。
4. **`cancel(boolean mayInterruptIfRunning)` 的工作原理：**
    - 如果任务状态不是 `NEW`，直接返回 `false` (无法取消)。
    - 尝试通过 CAS 将状态从 `NEW` 改为：
        - 如果 `mayInterruptIfRunning == false` -> 直接改为 `CANCELLED`。
        - 如果 `mayInterruptIfRunning == true` -> 先改为 `INTERRUPTING` (瞬时状态)。
    - 如果成功改为 `INTERRUPTING`：
        - 尝试中断执行任务的线程 (`Thread.interrupt()`)。
        - 将状态最终改为 `INTERRUPTED`。
    - 调用 `finishCompletion()` 唤醒等待线程。
    - 返回 `true` 表示取消请求已成功发出（注意：**成功发出请求不代表任务一定立即停止**，取决于任务是否响应中断）。
5. **`get()` 方法的工作原理：**
    - 如果状态是 `NORMAL`，直接返回结果。
    - 如果状态是 `EXCEPTIONAL`，抛出 `ExecutionException` (包装原始异常)。
    - 如果状态是 `CANCELLED` / `INTERRUPTING` / `INTERRUPTED`，抛出 `CancellationException`。
    - 如果状态是 `NEW` 或 `COMPLETING`：
        - 当前线程会通过 `LockSupport.park()` 或 `wait()` (旧版) 挂起（阻塞）。
        - 当任务状态变为最终状态 (`NORMAL`, `EXCEPTIONAL`, `CANCELLED`, `INTERRUPTED`) 时，`finishCompletion()` 会唤醒所有阻塞在 `get()` 上的线程。
6. **适用场景：**
    - **需要手动控制任务执行时机：** 当不想直接使用 `ExecutorService.submit()`，而是想自己决定何时何地（在哪个线程）执行任务时。
    - **实现自定义的任务调度器/执行器：** 作为任务队列中的基本任务单元。
    - **需要重复执行的任务？** **注意：** `FutureTask` 设计为**一次性**的。一旦运行完成（无论成功、失败、取消），就不能再运行。如果需要重复执行，需要创建新的 `FutureTask` 实例。
    - **需要 `runAndReset()` 的场景 (高级)：** `FutureTask` 提供了一个 `protected` 方法 `runAndReset()`。它执行任务但不设置结果/状态（保持 `NEW` 状态），允许任务被再次执行。**这是 `FutureTask` 特有的高级功能，但使用场景相对狭窄且需要小心同步问题**（例如，实现自定义的周期性任务，但通常 `ScheduledThreadPoolExecutor` 是更好的选择）。
7. **与 `CompletableFuture` 的区别 (关键面试点)：**
    |特性|`FutureTask`|`CompletableFuture`|
    |---|---|---|
    |**核心目标**|**基础任务执行单元**：封装任务、管理状态、提供结果|**高级异步编程模型**：提供强大的**组合、链式调用、异常处理**|
    |**实现接口**|`Runnable`, `Future<V>`|`Future<V>`, `CompletionStage<V>`|
    |**执行方式**|需要显式提交给线程或调用 `run()`|通常通过 `supplyAsync`/`runAsync` 自动提交执行|
    |**组合能力**|**无**。只能阻塞获取单个结果 (`get()`)。|**极强** (`thenApply`, `thenCompose`, `thenCombine`, `allOf`, `anyOf`...)|
    |**回调/非阻塞获取**|**无**。只能阻塞 `get()`。|**核心特性**：通过 `thenAccept`, `whenComplete` 等注册回调，非阻塞。|
    |**异常处理**|通过 `ExecutionException` 包装，需在 `get()` 捕获|提供链式异常处理 (`exceptionally`, `handle`)|
    |**手动完成**|只能通过 `run()` 执行任务来设置结果/异常。|提供 `complete(value)`, `completeExceptionally(ex)` 手动完成。|
    |**可重用性**|**一次性** (除非使用 `runAndReset()`，但复杂)|**一次性**|
    |**线程池集成**|需要显式提交 (`execute()`/`submit()`)|集成在创建方法中 (`supplyAsync(..., executor)`)|
    |**复杂性**|相对较低 (基础状态机)|相对较高 (复杂的依赖链管理)|
    |**适用场景**|基础任务封装、自定义执行器底层|现代异步编程、非阻塞IO集成、复杂任务流编排|
    
    **总结：** `FutureTask` 是**构建块**，提供了任务执行和结果管理的基础能力。`CompletableFuture` 是建立在类似基础（内部可能也用了类似 `FutureTask` 的机制）之上的**高级抽象**，专注于简化复杂的异步编程。
1. **常见陷阱与注意事项：**
    - **一次性：** 最大的陷阱是忘记 `FutureTask` 是一次性的。尝试重新执行一个已经完成（无论方式）的 `FutureTask` 的 `run()` 方法**不会做任何事**。需要重新创建实例。
    - **阻塞 `get()`：** 在主线程或关键线程中调用 `get()` 会导致阻塞，破坏响应性/并发性。尽量结合 `isDone()` 轮询或使用 `CompletableFuture` 的非阻塞回调。
    - **正确取消：** `cancel(true)` 只是发送中断请求，任务代码**必须响应中断** (`Thread.interrupted()`) 才能真正停止。否则任务可能继续运行直到结束。
    - **内存可见性：** `FutureTask` 内部通过状态机 (`volatile state`) 和 CAS 操作保证结果的可见性，这是安全的。但任务内部使用的共享数据仍需开发者自己保证线程安全（通过同步、`volatile`、原子类等）。
    - **`runAndReset()` 的复杂性：** 使用这个高级功能需要非常小心同步和状态重置逻辑，容易出错。优先考虑标准库的调度器 (`ScheduledExecutorService`)。
    - **结果类型：** 用 `Runnable` 构造时，结果是预设的固定值 (`resultValue`)。如果任务实际需要动态计算结果，必须使用 `Callable` 构造。
2. **为什么 `FutureTask` 仍然重要？**
    - **基础性：** 它是理解 `ExecutorService` 提交任务后返回 `Future` 内部运作的基础。
    - **灵活性：** 在需要完全控制任务执行线程或实现高度定制化执行逻辑时，它提供了底层控制能力。
    - **学习价值：** 其状态机的实现是并发编程中状态管理和线程间通信（结果传递、唤醒）的经典案例。

### 总结关键点
- **核心身份：** `FutureTask` 是 `Runnable` + `Future` 的实现 (`RunnableFuture`)。它将 `Callable` 或 `Runnable` 任务包装成一个可执行、可取消、可获取结果/状态的对象。
- **状态机：** 理解其内部状态 (`NEW`, `COMPLETING`, `NORMAL`, `EXCEPTIONAL`, `CANCELLED`, `INTERRUPTING`, `INTERRUPTED`) 及其变迁是深入掌握其原理的关键。
- **执行：** 通过 `Thread.start()` 或 `Executor.execute/submit()` 执行其 `run()` 方法。
- **结果获取：** 通过 `Future.get()` (阻塞) 获取结果或异常。
- **取消：** 通过 `cancel(mayInterruptIfRunning)` 尝试取消任务，效果取决于任务是否响应中断。
- **一次性：** **设计为一次性执行！** 完成后无法重新运行（除非用 `runAndReset()`，但慎用）。
- **与 `CompletableFuture` 区别：** `FutureTask` 是**基础任务执行单元**，提供基本执行和结果管理；`CompletableFuture` 是**高级异步组合框架**，提供链式调用、组合、非阻塞回调。`CompletableFuture` 在功能和易用性上远超 `FutureTask`。
- **适用场景：** 手动控制任务执行、自定义执行器/调度器底层实现、需要 `runAndReset()` 的特定高级场景（罕见）。
- **面试重点：** 状态机原理、`run()`/`get()`/`cancel()` 的工作机制、与 `CompletableFuture` 的核心区别、一次性特性、取消的语义。

**简单来说：** 当你想把一个耗时的计算（`Callable`）或操作（`Runnable`）打包成一个既能扔给线程池/线程去跑（`Runnable`），又能在跑完后拿到结果或知道为什么没跑完（`Future`）的对象时，`FutureTask` 就是这个打包好的“任务包裹”。它是 JDK 并发工具链中承上启下的重要一环。但在日常异步编程中，`CompletableFuture` 通常是更现代、更强大的选择。