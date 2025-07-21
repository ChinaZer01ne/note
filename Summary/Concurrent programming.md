## CompletableFuture

CompletableFuture‌是Java 8中引入的一个类，它实现了Future和CompletionStage接口，用于支持异步编程和非阻塞操作。CompletableFuture提供了丰富的API，使得并发任务的处理变得简单而高效，能够轻松创建、组合和链式调用异步操作，无需关心底层线程管理，从而提升了程序的响应速度和资源利用率‌。
> * 它是 `Future` 的**超强进化版**，代表一个异步计算的结果，但提供了**异常灵活的组合、链式调用和异常处理**能力。
> * 其设计的目的是为了实现非阻塞的异步编程模型。
> * `CompletableFuture` 实现了 `CompletionStage` 接口，该接口定义了**大量组合异步操作的方法**（`thenApply`, `thenAccept`, `thenRun`, `thenCompose`, `thenCombine`, `handle`, `exceptionally` 等）。

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
- **异步变体：** `thenApplyAsync`, `thenAcceptAsync`, `thenRunAsync` - 在**新线程**中执行转换/消费/动作（默认使用 `ForkJoinPool` 或指定线程池）。**理解 `...Async` 后缀的含义是关键！**
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
## FutureTask
