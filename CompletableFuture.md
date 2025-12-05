

|API 类别|核心方法|功能描述|
|---|---|---|
|**任务创建**​|`supplyAsync(Supplier)`|创建有返回值的异步任务|
||`runAsync(Runnable)`|创建无返回值的异步任务|
||`completedFuture(U)`|创建已完成的 CompletableFuture|
|**结果转换**​|`thenApply(Function)`|对上个任务结果进行同步转换|
||`thenCompose(Function)`|将两个异步任务“扁平化”串联|
|**结果消费**​|`thenAccept(Consumer)`|同步消费结果，无返回值|
||`thenRun(Runnable)`|上个任务完成后执行动作，不关心结果|
|**任务组合**​|`thenCombine(CF, BiFunction)`|组合两个**独立**任务的结果|
||`allOf(CF...)`|等待**所有**给定任务完成|
||`anyOf(CF...)`|等待**任意**一个任务完成|
|**异常处理**​|`exceptionally(Function)`|捕获异常并提供降级结果|
||`handle(BiFunction)`|无论成功失败都执行，可获取结果或异常|

### 🔧 核心 API 详解

#### 1 创建异步任务

这是一切的起点。

- **`supplyAsync(Supplier)`**：用于执行有返回值的任务。
    
    ```
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        // 模拟耗时操作
        try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
        return "Hello, CompletableFuture!";
    });
    String result = future.join(); // 阻塞获取结果
    System.out.println(result); // 输出：Hello, CompletableFuture!
    ```
    
- **`runAsync(Runnable)`**：用于执行不需要返回结果的任务。
    
    ```
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        System.out.println("Task is running...");
    });
    future.join(); // 等待任务完成
    ```
    

**最佳实践**：这些方法通常有重载版本，允许你传入自定义的 `Executor`。默认使用 `ForkJoinPool.commonPool()`，但**对于I/O密集型或需要资源隔离的任务，强烈建议使用自定义线程池**​ 。

#### 2 处理单个任务结果

在任务完成后，你需要对它的结果进行处理。

- **`thenApply`- 转换结果**：接收上一个任务的结果，进行转换，并返回新值。
    
    ```
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
                                                      .thenApply(s -> s + " World")
                                                      .thenApply(String::toUpperCase);
    System.out.println(future.join()); // 输出：HELLO WORLD
    ```
    
- **`thenAccept`- 消费结果**：接收结果，但只消费不返回（类似 `forEach`）。
    
    ```
    CompletableFuture.supplyAsync(() -> "Hello")
                     .thenAccept(s -> System.out.println("Received: " + s));
    // 输出：Received: Hello
    ```
    
- **`thenRun`- 后续操作**：不关心上一个任务的结果，只是在它完成后执行一个操作。
    
    ```
    CompletableFuture.supplyAsync(() -> "Hello")
                     .thenRun(() -> System.out.println("Process completed."));
    // 输出：Process completed.
    ```
    

**注意**：以上方法都有对应的异步版本（如 `thenApplyAsync`），它们会在新的线程中执行，避免阻塞当前线程。

#### 3 组合多个任务

这是 CompletableFuture 最强大的能力之一，可以优雅地处理任务间的依赖关系。

- **`thenCompose`- 链式依赖**：用于**串行化**两个任务，当前一个任务完成后，将其结果作为后一个任务的输入。
    
    ```
    // 假设有方法 findUserById 和 getUserDetail
    CompletableFuture<User> userFuture = findUserById(1);
    // 传统方式会得到 CompletableFuture<CompletableFuture<UserDetail>>
    // 使用 thenCompose 进行“扁平化”处理
    CompletableFuture<UserDetail> resultFuture = userFuture.thenCompose(user -> getUserDetail(user.getId()));
    ```
    
- **`thenCombine`- 合并结果**：当两个**并行**执行的任务都完成后，将它们的结果进行合并。
    
    ```
    CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 10);
    CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 20);
    CompletableFuture<Integer> combinedFuture = future1.thenCombine(future2, (a, b) -> a + b);
    System.out.println(combinedFuture.join()); // 输出：30
    ```
    
- **`allOf`/ `anyOf`- 聚合等待**：
    
    - `allOf`等待所有任务完成，但不会聚合结果，需要手动处理。
        
    
    ```
    CompletableFuture<Void> allFutures = CompletableFuture.allOf(future1, future2, future3);
    allFutures.thenRun(() -> {
        // 所有任务都完成了，可以安全地调用 join() 获取结果
        String result1 = future1.join();
        String result2 = future2.join();
        // ... 处理所有结果
    });
    ```
    
    - `anyOf`在任意一个任务完成时触发。
        
    

#### 4 异常处理

在异步流水线中，异常处理至关重要。

- **`exceptionally`- 异常恢复**：类似于 `try-catch`，捕获异常并提供默认值。
    
    ```
    CompletableFuture<String> safeFuture = CompletableFuture.supplyAsync(() -> {
        if (true) throw new RuntimeException("Oops!");
        return "Success";
    }).exceptionally(ex -> "Fallback Value: " + ex.getMessage());
    System.out.println(safeFuture.join()); // 输出：Fallback Value: Oops!
    ```
    
- **`handle`- 最终处理**：无论成功与否都会执行。接收两个参数：结果和异常。你可以根据情况决定返回什么。
    
    ```
    CompletableFuture<Integer> handledFuture = CompletableFuture.supplyAsync(() -> 10 / 0) // 会抛出异常
        .handle((result, ex) -> ex != null ? 0 : result * 2); // 如果异常则返回0，否则对结果加倍
    System.out.println(handledFuture.join()); // 输出：0
    ```
    
