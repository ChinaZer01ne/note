## CompletableFuture

CompletableFuture‌是Java 8中引入的一个类，它实现了Future和CompletionStage接口，用于支持异步编程和非阻塞操作。CompletableFuture提供了丰富的API，使得并发任务的处理变得简单而高效，能够轻松创建、组合和链式调用异步操作，无需关心底层线程管理，从而提升了程序的响应速度和资源利用率‌。
> 它是 `Future` 的**超强进化版**，代表一个异步计算的结果，但提供了**异常灵活的组合、链式调用和异常处理**能力。
  其设计的目的是为了实现非阻塞的异步编程模型。
  - **CompletionStage 接口：** 知道 `CompletableFuture` 实现了 `CompletionStage` 接口，该接口定义了**大量组合异步操作的方法**（`thenApply`, `thenAccept`, `thenRun`, `thenCompose`, `thenCombine`, `handle`, `exceptionally` 等）。

作者：wo_am_i  
链接：https://juejin.cn/post/7433365373381853194  
来源：稀土掘金  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
## Callable
## FutureTask
