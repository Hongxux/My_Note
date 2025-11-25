---
aliases:
  - StreamMessageListenerContainer
---
![[Redis Stream的构建信息消费服务 2025-11-20 21.34.51.excalidraw]]
## 完整代码

```
@Configuration  
@EnableAsync  
public class RedisStreamConfig {  
  
  
    /**  
     * 配置自定义线程池，用于执行消息监听任务  
     * 避免使用默认的SimpleAsyncTaskExecutor（会无限制创建线程）  
     */  
    @Bean  
    public ThreadPoolExecutor streamListenerExecutor(){  
        int corePoolSize = Runtime.getRuntime().availableProcessors();//获取服务器的可用处理器个数  
        return new ThreadPoolExecutor(  
                corePoolSize,  
                corePoolSize * 2,  
                60, TimeUnit.SECONDS,  
                new LinkedBlockingDeque<>(1000),  
                new ThreadFactory() {  
                    private final AtomicInteger threadIndex = new AtomicInteger(0);  
                    @Override  
                    public Thread newThread(Runnable r) {  
                        Thread t = new Thread(r);  
                        t.setName("redis-stream-comsumer" + "-" + threadIndex.getAndIncrement());  
                        t.setDaemon(true);  
                        return t;  
                    }  
                },  
                new ThreadPoolExecutor.CallerRunsPolicy()  
        );  
    }  
  
    /**  
     * 配置容器选项，对应图片中的 COUNT 和 BLOCK 参数以及执行策略  
     */  
  
    /**     * 创建消息监听容器  
     */  
    @Bean  
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>> streamMessageListenerContainer(RedisConnectionFactory  redisConnectionFactory, ThreadPoolExecutor streamListenerExecutor){  
        StreamMessageListenerContainer.StreamMessageListenerContainerOptions streamMessageListenerContainerOptions = StreamMessageListenerContainer.StreamMessageListenerContainerOptions  
                .builder()  
                .batchSize(10)  
                .pollTimeout(Duration.ofSeconds(10))  
                .executor(streamListenerExecutor)  
                .keySerializer(RedisSerializer.string()) // Stream Key 序列化器  
                .hashKeySerializer(RedisSerializer.string()) // 消息字段Key的序列化器  
                .hashValueSerializer(RedisSerializer.string()) // 消息字段Value的序列化器（重要！）  
                .build();  
        StreamMessageListenerContainer<String, MapRecord<String, String, String>> listenerContainer = StreamMessageListenerContainer.create(redisConnectionFactory, streamMessageListenerContainerOptions);  
        //StreamMessageListenerContainer必须调用 start()方法后才会开始监听消息。  
        //listenerContainer.start();  
        return listenerContainer;  
    }  
  
  
  
  
  
}
```
## 分块说明

### 自定义有界线程池
它没有使用 Spring 默认的 `SimpleAsyncTaskExecutor`（该执行器会为每个任务无限制地创建新线程，在生产环境下有资源耗尽的风险），而是配置了一个**有界线程池**6
```
@Bean(destroyMethod = "shutdown")
public ThreadPoolExecutor streamListenerExecutor() {
	int corePoolSize = Runtime.getRuntime().availableProcessors();
	return new ThreadPoolExecutor(
			corePoolSize,
			corePoolSize * 2,
			60L, TimeUnit.SECONDS,
			new LinkedBlockingQueue<>(1000),
			new ThreadFactory() {
				private final AtomicInteger threadIndex = new AtomicInteger(0);
				@Override
				public Thread newThread(Runnable r) {
					Thread thread = new Thread(r);
					thread.setName("redis-stream-consumer-" + threadIndex.incrementAndGet());
					thread.setDaemon(true);
					return thread;
				}
			},
			new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略：由调用线程直接执行
	);
}
```
![[Pasted image 20251120215056.png]]
- **作用**：这个线程池将负责运行所有**消息拉取**和**消息消费**任务。
    
- **关键参数**：
    
    - `corePoolSize`和 `maximumPoolSize`：根据CPU核数动态设置，合理利用多核性能。
        
    - `LinkedBlockingQueue(1000)`：设置了一个容量为1000的任务队列。当活动线程数达到核心数时，新任务会进入队列等待，而不是立即创建新线程，这有助于平滑处理突发流量。
	    - 这个数值需要进行压力测试得到。
        
    - [[线程池拒绝策略选择与配置]]
    - **[[守护线程]]（Daemon Thread）**： 不构成程序核心业务、旨在为其他线程提供服务，并且可以随时被终止的任务。

### 选择基于 `ObjectRecord`的 Redis Stream 消息队列
这种类型将整个消息体视为一个完整的对象，适合传输结构固定的复杂业务对象。

- **数据形式**：框架会尝试将消息中的所有字段序列化后存入一个大的值中，或在消费端将整个消息体反序列化成一个指定的Java对象。
    
- **序列化配置**：核心在于正确配置 `ObjectMapper`（例如 Jackson 的 `ObjectMapper`），并通常将 `targetType`设置为你的业务对象类（如 `Book.class`）。`hashValueSerializer`在此模式下通常用于序列化对象内部的字段值，需谨慎配置，有时使用 `RedisSerializer.string()`或 JSON 序列化器，若配置不当可能引发反序列化问题。
  
| 组件                             | 核心配置项              | 关键值/说明                                                           |
| ------------------------------ | ------------------ | ---------------------------------------------------------------- |
| **监听容器选项 (ContainerOptions)**​ | 泛型类型               | `ObjectRecord<String, Object>`                                   |
|                                | Hash Value 序列化器    | `RedisSerializer.json()`(如 `GenericJackson2JsonRedisSerializer`) |
|                                | 目标类型 (Target Type) | `Object.class`                                                   |
| **生产者 (Producer)**​            | 记录创建方式             | `ObjectRecord.create(streamKey, yourObject)`                     |
|                                | 发送模板               | `redisTemplate.opsForStream().add(objectRecord)`                 |
| **消费者 (Consumer)**​            | 监听器泛型              | `StreamListener<String, ObjectRecord<String, Object>>`           |
|                                | 消息提取               | `Object body = message.getValue()`                               |





### 选择基于 `MapRecord`的 Redis Stream 消息队列
这种类型将消息视为一个键值对集合，非常灵活，适合动态字段或不需要严格POJO映射的场景。

- **数据形式**：消息在Redis中存储为标准的字段-值对（field-value pairs）。消费端接收到的 `MapRecord`的 `value`是一个 `Map<HK, HV>`。
    
- **序列化配置**：需要分别为 `hashKeySerializer`（对应字段名）和 `hashValueSerializer`（对应字段值）配置序列化器。对于字符串类型的键值，通常使用 `StringRedisSerializer`。`targetType`通常设置为 `Map.class`（每次设置都出错，建议不要管）。


### 选择基于 `ByteBufferRecord`的 Redis Stream 消息队列
这是一种更底层的类型，直接操作字节数据，提供了最大的灵活性，但易用性较低。

- **数据形式**：消息体作为 `ByteBuffer`处理，框架不进行任何自动序列化/反序列化。
    
- **序列化配置**：通常需要配置字节序列化器，例如 `RedisSerializer.byteArray()`。业务代码需要自行处理 `ByteBuffer`的解析。
    
- **适用场景**：需要极致性能控制，或者有自定义、非标准序列化协议的场景。


### 2. 容器选项配置 (`containerOptions`)
这部分定义了 `StreamMessageListenerContainer` 的工作行为。
```java
StreamMessageListenerContainer.StreamMessageListenerContainerOptions streamMessageListenerContainerOptions = StreamMessageListenerContainer.StreamMessageListenerContainerOptions  
        .builder()  
        .batchSize(10)  
        .pollTimeout(Duration.ofSeconds(10))  //配置读取的阻塞事件
        .executor(streamListenerExecutor)  //配置自定义的线程池
        .keySerializer(RedisSerializer.string()) // Stream Key 序列化器  
        .hashKeySerializer(RedisSerializer.string()) // 消息字段Key的序列化器  
        .hashValueSerializer(RedisSerializer.string()) // 消息字段Value的序列化器（重要！）  
        .build();
```


| **`batchSize`**​           | 单次轮询从Stream中获取的**最大消息条数**。      | 建议 `10`。根据消息处理速度和实时性要求调整，高吞吐场景可适当增大。                                    |
| -------------------------- | ------------------------------- | ----------------------------------------------------------------------- |
| **`pollTimeout`**​         | **轮询超时时间**。监听器等待新消息的阻塞时间。       | 建议 `Duration.ofSeconds(2)`。**必须小于**Redis服务器超时设置。                        |
| **`executor`**​            | **执行轮询任务的线程池**。负责运行消息监听任务。      | 使用 `Executors.newCachedThreadPool()`或配置合理的自定义线程池，避免监听器饥饿<br><br>。       |
| **`keySerializer`**​       | 用于 **Stream Key**​ 的序列化器。       | `RedisSerializer.string()`<br><br>。                                     |
| **`hashKeySerializer`**​   | 用于消息体（Map）中**字段名（Field）**的序列化器。 | `RedisSerializer.string()`<br><br>。                                     |
| **`hashValueSerializer`**​ | 用于消息体（Map）中**字段值（Value）**的序列化器。 | `RedisSerializer.string()`（字符串值）或JSON序列化器（复杂对象）<br><br>。                |
| **`targetType`**​          | 指定消息体转换的**目标Java类型**。           | 使用 `MapRecord`时设为 `Map.class`；使用 `ObjectRecord`时设为自定义类（如 `Book.class`）。 |
| **`errorHandler`**​        | **错误处理器**。处理消息获取或消费过程中的异常。      | 实现 `ErrorHandler`接口，记录日志并决定是否恢复<br><br>。                                |
| **`objectMapper`**​        | 将消息字段和值转换为Map的**映射器**。          | 使用 `ObjectRecord`且消息体为对象时，可配置 `new ObjectHashMapper()`<br><br>。         |

### 3.监听器容器的泛型选择

1. **保持一致性是关键**：**生产端写入消息的格式必须与消费端 `StreamMessageListenerContainer`配置期望的泛型类型和序列化方式完全匹配**。例如，若生产端使用 `MapRecord`写入字段值对，消费端也应使用 `MapRecord`并配置相应的 `hashKeySerializer`和 `hashValueSerializer`来接收。
    
2. **优先考虑 `MapRecord<String, String, String>`**：对于大多数基于字段-值对的消息（尤其是像您从Lua脚本生产的数据），`MapRecord<String, String, String>`通常是简单直接的选择。配置清晰，易于调试。
    
3. **谨慎处理 `ObjectRecord`**：虽然它更面向对象，但在序列化器配置上可能需要更多注意，以避免类型转换异常。
    
4. **明确配置序列化器**：不要依赖默认序列化配置。明确配置 `keySerializer`, `hashKeySerializer`, `hashValueSerializer`和 `targetType`可以有效避免许多难以排查的问题。

- **`ObjectRecord<K, V>`**：适合**结构固定**的复杂业务对象。
    
- **`MapRecord<K, HK, HV>`**：适合**动态字段**或简单的键值对消息，灵活性高。
    
- **`ByteBufferRecord`**：用于需要**直接处理字节**的高级场景。
    




