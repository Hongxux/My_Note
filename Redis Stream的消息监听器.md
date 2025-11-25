---
aliases:
  - 消息监听器
  - StreamListener
---
```
import org.springframework.data.redis.connection.stream.ObjectRecord;
import org.springframework.data.redis.connection.stream.RecordId;
import org.springframework.data.redis.stream.StreamListener;
import org.springframework.stereotype.Component;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
@RequiredArgsConstructor
public class MyStreamMessageListener implements StreamListener<String, ObjectRecord<String, String>> {

    private final RedisTemplate<String, String> redisTemplate;

    /**
     * **这是最关键的方法：消费者通过此方法自动获取信息**
     * Spring容器在后台自动轮询（对应图片的while(true)循环），
     * 当有新消息或pending消息时，会自动调用此方法并将消息推送给它。
     */
    @Override
    public void onMessage(ObjectRecord<String, String> message) {
        String stream = message.getStream();
        RecordId messageId = message.getId();
        String messageBody = message.getValue();

        log.info("接收到消息。Stream: [{}], ID: [{}], Body: [{}]", stream, messageId, messageBody);

        try {
            // 对应图片中的 handleMessage(msg)
            boolean success = handleMessage(messageBody);

            if (success) {
                // **手动ACK**，对应图片中的“完成后一定要ACK”
                redisTemplate.opsForStream().acknowledge(stream, CONSUMER_GROUP, messageId);
                log.info("消息处理成功，已ACK: {}", messageId);
            } else {
                log.warn("消息处理失败，等待重新投递: {}", messageId);
                // 不进行ACK，消息会保留在Pending List中，稍后由 pendingRequest 重新处理
            }

        } catch (Exception e) {
            log.error("处理消息时发生异常: {}", messageId, e);
            // 不进行ACK，消息会保留在Pending List中
            // 对应图片中 catch(Exception e) 后的逻辑，但这里通过框架自动重试，无需内层循环
        }
    }

    /**
     * 你的核心业务逻辑，对应图片中的 handleMessage
     */
    private boolean handleMessage(String message) {
        // 在这里实现你的业务逻辑，例如：
        // - 解析JSON
        // - 更新数据库
        // - 调用外部服务
        // 返回true表示成功，false表示需要重试
        return true; // 模拟成功
    }
}
```

这段代码是一个基于 Spring Data Redis 的 **Redis Stream 消息消费者实现**。它采用**事件驱动**模型，当有消息到达 Redis Stream 时，Spring 框架会自动调用 `onMessage`方法。这是一种比手动循环更现代和高效的消息处理方式。


|代码部分|核心职责|关键交互与影响|
|---|---|---|
|**`onMessage`方法**​|**消息处理的总入口和调度中心**。由Spring框架在收到新消息时自动调用。|协调整个处理流程：记录日志、调用业务逻辑、根据结果决定是否ACK。|
|**`handleMessage`方法**​|**执行业务逻辑**。这里是处理消息内容的核心。|返回的 `boolean`值直接决定 `onMessage`方法是否发送ACK。|
|**ACK确认机制**​|**向Redis服务器确认消息已成功处理**。|只有在业务逻辑返回 `true`时才执行，是保证消息不丢失的关键。|
|**异常处理块**​|**捕获并处理消息处理过程中抛出的异常**。|发生异常时，不会执行ACK，消息会保留在Pending List中以待重试。|

下面我们来详细剖析这段代码。

### 🔧 代码工作机制详解

#### 1. 消息如何被接收？

- 你的代码**不需要**自己写 `while(true)`循环来轮询消息。它通过实现 `StreamListener<String, ObjectRecord<String, String>>`接口，并将自己注册到 Spring 的 `StreamMessageListenerContainer`中。
    
- Spring 容器会在后台自动管理一个**常驻的监听任务**。当指定的 Redis Stream 中有新消息到达，或有待处理的 Pending 消息时，容器会**主动推送**这条消息，并调用你的 `onMessage`方法 。这是一种**监听-回调**模式。
    

#### 2. 核心处理流程与可靠性设计

`onMessage`方法内的 `try-catch`块和条件判断，共同构成了一套保证消息**至少被处理一次（At-Least-Once)**  的可靠机制。

```
try {
    // 1. 调用业务逻辑
    boolean success = handleMessage(messageBody);

    if (success) {
        // 2. 业务成功：发送ACK
        redisTemplate.opsForStream().acknowledge(stream, CONSUMER_GROUP, messageId);
        log.info("消息处理成功，已ACK: {}", messageId);
    } else {
        // 3. 业务逻辑自身返回失败：不发送ACK
        log.warn("消息处理失败，等待重新投递: {}", messageId);
    }
} catch (Exception e) {
    // 4. 处理过程发生异常：不发送ACK
    log.error("处理消息时发生异常: {}", messageId, e);
}
```

- **手动ACK机制**：代码使用**手动确认**模式。这意味着你必须显式调用 `acknowledge`方法，Redis 服务器才会将这条消息从该消费者组的“待处理列表”中移除 。这是可靠性的基石。
    
- **三种处理结果与ACK策略**：
    
    1. **业务成功（`success`为 `true`）**：发送 ACK，消息被标记为已成功处理。
        
    2. **业务逻辑失败（`success`为 `false`）**：不发送 ACK。消息会留在 Redis 的 Pending List 中，之后可以被**重新分派**给这个消费者或其他消费者进行重试 。
        
    3. **处理过程异常**：被 `catch`块捕获，不发送 ACK。效果同第2点，消息会被保留以供重试。
        
    

#### 3. 消费者组与负载均衡

代码中的 `CONSUMER_GROUP`常量（虽然未在片段中显示定义，但被ACK方法使用）表明它工作在**消费者组**模式下 。

- 在同一个消费者组内，**一条消息只会被一个消费者实例接收到**。如果你启动多个服务实例（同组不同消费者名），它们会自动进行负载均衡，共同消费一个 Stream 中的消息，提高处理能力 。
    
- 这也要求消费者组的配置（创建等）需要在应用启动前或启动时完成 。
    
