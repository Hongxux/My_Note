
### 使用和配置自定义的线程池

- 需求背景：企业级应用首要的是**资源可控**。Spring Boot默认的`SimpleAsyncTaskExecutor`（每次调用创建新线程）在生产环境使用极易导致资源耗尽。因此，自定义线程池是第一步。

以下是两种主流的配置方式，其核心参数和适用场景可通过下表快速了解：

|**配置方式**​|**核心实现**​|**关键参数**​|**企业级考量**​|
|---|---|---|---|
|**`ThreadPoolTaskExecutor`**​|Spring 封装，与 `@Async`集成简便|`corePoolSize`, `maxPoolSize`, `queueCapacity`, `RejectedExecutionHandler`|参数需根据业务类型（CPU密集型/IO密集型）调整，例如IO密集型可适当调大线程数。|
|**`AsyncConfigurer`**​|实现接口，可统一配置异常处理|同上，通过 `getAsyncExecutor()`方法返回|实现了 `AsyncConfigurer`的配置类会覆盖默认的线程池配置。|

**1. 使用 `ThreadPoolTaskExecutor`（推荐）**

这是最常用且灵活的方式。

```
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // 核心线程数，长期维持的线程数
        executor.setCorePoolSize(10);
        // 最大线程数，线程池所能创建的最大线程数
        executor.setMaxPoolSize(50);
        // 队列容量，用于缓存等待执行任务的队列大小
        executor.setQueueCapacity(100);
        // 线程空闲后的存活时间（秒）
        executor.setKeepAliveSeconds(60);
        // 线程名前缀，便于日志追踪
        executor.setThreadNamePrefix("Async-Service-");
        
        // 拒绝策略：当线程池和队列都满时如何处理新任务
        // CallerRunsPolicy：由调用者线程（如HTTP请求线程）直接执行，是一种温和的降级策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // 等待所有任务结束后再关闭线程池，保证数据不丢失
        executor.setWaitForTasksToCompleteOnShutdown(true);
        // 等待任务结束的最大时间，超过则强制关闭
        executor.setAwaitTerminationSeconds(60);
        
        executor.initialize();
        return executor;
    }
}
```

**2. 通过 `AsyncConfigurer`全局配置**

这种方式可以同时指定默认的线程池和异常处理器。

```
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("Global-Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        // 全局捕获异步方法中未捕获的异常
        return (ex, method, params) -> {
            // 此处应接入企业级的日志和监控系统
            System.err.println("异步方法执行失败: " + method.getName());
            ex.printStackTrace();
        };
    }
}
```

配置好后，在服务层指定使用该线程池：

```
@Service
public class EmailService {
    @Async("taskExecutor") // 指定使用自定义的线程池
    public void sendWelcomeEmail(String email) {
        // 发送邮件的业务逻辑
    }
}
```

###  避开四大常见陷阱

1. **陷阱：同类调用导致异步失效**
    
    **问题**：由于Spring的异步基于AOP代理，在同一个类中，一个方法调用另一个`@Async`方法不会经过代理，导致异步失效。
    
    **解决**：将异步方法放到另一个Bean中，或通过注入自身代理调用。
    
    ```
    @Service
    public class OrderService {
        @Autowired
        private ApplicationContext applicationContext;
    
        public void createOrder() {
            // 错误：直接调用，异步不生效
            // this.asyncLog();
    
            // 正确：通过代理对象调用
            OrderService proxy = applicationContext.getBean(OrderService.class);
            proxy.asyncLog();
        }
    
        @Async
        public void asyncLog() { ... }
    }
    ```
    
2. **陷阱：异步方法异常被"静默吞没"**
    
    **问题**：无返回值的异步方法如果抛出异常，默认不会记录到调用方日志，导致问题难以排查。
    
    **解决**：
    
    - **对于无返回值方法**：务必配置全局的`AsyncUncaughtExceptionHandler`。
        
    - **对于有返回值方法**：返回`CompletableFuture`，并在调用端通过`future.get()`或`future.exceptionally()`处理异常。
        
    
3. **陷阱：事务与异步的上下文丢失**
    
    **问题**：`@Transactional`的事务信息绑定在`ThreadLocal`上。异步方法在新线程中运行，无法继承原线程的事务上下文。
    
    **解决**：将核心事务操作放在同步方法中完成，异步方法只负责非事务性的后续操作（如发邮件、记录日志）。如果异步方法自身需要事务，可使用`@Transactional(propagation = Propagation.REQUIRES_NEW)`开启一个新事务。
    
4. **陷阱：`ApplicationEvent`默认是同步的**
    
    **问题**：使用Spring的事件发布订阅模型（`ApplicationEventPublisher`）时，即使监听器方法加了`@Async`，事件发布默认也是同步的。
    
    **解决**：确保监听器方法上同时使用`@Async`和`@EventListener`（或`@TransactionalEventListener`）。
    
    ```
    @Component
    public class EmailEventListener {
        @Async("taskExecutor")
        @EventListener // 或 @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
        public void handleUserRegisteredEvent(UserRegisteredEvent event) {
            // 处理事件，如发送邮件
        }
    }
    ```
    