---
aliases:
  - ErrorHandler
  - 自定义错误处理器
---
这是一个用于 **Spring Data Redis Stream**​ 消费环境的**自定义全局异常处理器**。它的核心职责是捕获消息消费过程中抛出的异常，并根据异常类型决定最合适的处理策略，从而增强应用的健壮性。

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.RedisSystemException;
import java.util.function.Consumer;

@Slf4j
public class MyErrorHandler implements Consumer<Throwable> {

    @Override
    public void accept(Throwable throwable) {
        log.error("Stream监听发生异常: ", throwable);

        // 对应图片中的 catch(Exception e) 逻辑
        // 根据异常类型决定是否停止容器
        if (throwable instanceof RedisSystemException) {
            // 如果是Redis连接等系统级异常，可能需要停止容器并告警
            log.error("发生严重Redis异常，需要停止容器并检查连接。");
            // 返回true会停止容器，返回false会继续运行（让框架重试）
            // 这里可以根据业务需要调整，生产环境建议对网络异常返回false以允许重连
        } else {
            // 业务逻辑异常，不停止容器，只记录日志（对应图片中的continue）
            log.warn("业务处理异常，容器继续运行等待下一条消息。");
        }
    }
}
```

#### 1. 核心接口与类

- **`Consumer<Throwable>`接口**：这是Java内置的函数式接口。你的类实现它，意味着承诺提供一个 `accept`方法来“消费”（即处理）一个 `Throwable`异常对象。
    
- **`@Slf4j`注解**：这是Lombok提供的注解，它会自动在编译时生成一个名为 `log`的日志对象（通常基于SLF4J），让你能直接使用 `log.error()`等方法记录日志。
#### 2. 异常处理策略：分类治理

代码最核心的部分是 `if-else`逻辑，它根据异常类型实施“分类治理”


```
if (throwable instanceof RedisSystemException) {
    // ... 处理系统级异常
} else {
    // ... 处理业务级异常
}
```

- **`RedisSystemException`** ：这是Spring Data Redis定义的异常，通常包装了底层的Redis连接错误（如无法连接Redis服务器）、命令执行失败等**系统级、基础性故障**。这类异常通常意味着消息消费的基础环境出了问题，即使重试也可能失败，需要立即关注。
	- **引入重试与降级机制**：对于网络异常等可恢复故障，可以集成重试库（如Resilience4j）进行有限次数的重试。对于持续失败，可以触发降级逻辑，比如将消息转入死信队列
    
- **其他异常（`else`分支）**：通常被认为是**业务逻辑异常**，比如处理消息内容时发生的数据处理错误、格式转换异常等。这些异常通常只影响当前这一条消息，不应中断整个容器的运行。

- **对接监控系统**：在记录错误日志的同时，可以增加指标上报，便于监控系统（如Prometheus）告警，实现主动发现问题
    