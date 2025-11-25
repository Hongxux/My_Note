**消费者组机制**实现了负载均衡和可靠消费
**消息持久化**保证了数据不丢失
**内存存储**则带来了高性能。


## Redis Stream的实现底层
Redis Stream 是 Redis 5.0 版本专为消息队列场景设计的**日志型数据结构**。它巧妙融合了多种底层技术，实现了高性能、高可靠的消息传递。下面这个表格汇总了它的核心特点，并关联了其背后的实现原理，让你能快速建立整体认知。

|核心特点|具体表现|背后的实现原理|
|---|---|---|
|**消息持久化与回溯**​|消息被追加到流中并持久化到磁盘，支持按消息ID或时间范围查询历史消息。|采用**只追加日志**的存储模式，并结合Redis本身的RDB/AOF持久化机制。底层使用**Rax树（基数树）索引消息ID**，支持高效的范围查询。|
|**消费者组机制**​|支持多个消费者组独立消费同一流；同一组内多个消费者竞争消费，实现负载均衡。|为每个消费者组维护一个**最后送达消息ID（last_delivered_id）**​ 和一个**待处理条目列表（Pending Entries List, PEL）**，从而跟踪消费进度和处理中的消息。|
|**可靠的消息传递**​|提供**至少一次**的投递语义。消费者需显式确认消息，未确认的消息会一直留在PEL中，可供重新投递。|消息被消费者读取后进入**PEL**，直到收到**XACK**命令才会被移除。通过**XPENDING**和**XCLAIM**命令，可以监控和转移处理超时的消息，防止消息因消费者故障而丢失。|
|**高效的内存与存储**​|在保证功能的同时，能有效节省内存。|使用紧凑的**listpack**结构存储多条消息内容；使用**Rax树**存储消息ID，该结构能高效压缩存储具有相同前缀的键，从而减少内存占用。|
|**阻塞式读取**​|消费者可以阻塞等待新消息到达，避免无效的轮询，减少CPU开销。|客户端使用**XREAD**或**XREADGROUP**命令时，可设置**BLOCK**参数，服务器会在有新消息时唤醒阻塞的客户端连接。|

---

这些强大特性的背后，是Redis Stream精妙的底层设计，主要依赖于**Rax树**和**listpack**两种核心数据结构。

### 1. 混合存储结构：Rax树 + listpack

Redis Stream并没有为每条消息单独存储，而是采用了一种批量存储的策略。

- **listpack（列表集合）**：这是一个**紧凑的、连续内存块**，用于**实际存储多条消息的内容**（键值对）。一个listpack内会存放多条消息，这种方式减少了内存碎片和管理开销。
    
- **Rax树（基数树）**：这是一个用于**高效索引消息ID**的数据结构。Rax树的每个节点对应一个消息ID的前缀，其**值是一个指针，指向一个包含多条消息的listpack**。这种设计特别适合存储像消息ID（`<时间戳>-<序列号>`）这样具有递增和相同前缀的数据，能极大地节省内存。
    

**工作流程**：当使用`XADD`添加一条消息时，Redis会将其追加到合适的listpack中，并在Rax树中创建或更新对应的索引节点。当通过ID查询消息时，首先在Rax树中进行快速查找（时间复杂度接近O(logN)），定位到对应的listpack，然后在该listpack内进行线性扫描找到具体消息。

### 2. 消费者组的实现：状态跟踪

这是实现可靠消息队列的核心。每个消费者组都维护着几个关键状态：

- **last_delivered_id**：一个游标，记录该消费者组**即将要消费的下一条消息的ID**。当一个消费者通过`XREADGROUP`读取消息后，这个游标就会向前移动。
    
- **Pending Entries List（PEL）**：这是一个非常关键的列表。当消息被组内某个消费者读取后，在它被确认（ACK）之前，**该消息的ID和消费者的信息会被记录在PEL中**。这标志着此消息正在处理中。
    
- **消息确认（ACK）**：消费者处理完消息后，必须发送`XACK`命令。服务器会从PEL中移除对应的条目，表示消息已成功处理。
    
- **故障转移**：如果某个消费者崩溃，导致PEL中的消息迟迟未被确认，其他健康的消费者可以使用`XCLAIM`命令将这些消息“认领”过来并重新处理，从而保证了消息不会丢失。
    

### 3. 消息ID的奥秘：有序性与唯一性

每条消息都有一个唯一的ID，默认格式为`<毫秒时间戳>-<序列号>`（例如 `1679218539571-0`）。这种设计保证了：

1. **全局有序**：后生成的消息ID一定大于先生成的消息ID，这使得消息在流中自然按时间排序。
    
2. **唯一性**：即使在同毫秒内产生多条消息，序列号也会递增，确保ID唯一。
    

这种有序的ID是实现范围查询（`XRANGE`）和消费者组进度跟踪的基础。

---

## Redis Stream常用的命令

### 生产者-XADD

**`XADD key [NOMKSTREAM] [MAXLEN|MINID limit] <id> field value [field value ...]`**
- 核心作用：**生产消息**，向指定Stream中添加一条新消息。
- 参数含义：
	- **key**​ (必需)：Stream的名称，即消息队列的名称
	    
	- **`[NOMKSTREAM]`**​ (可选)：如果Stream不存在，**不自动创建**并返回错误。如不设置此参数，默认会自动创建Stream
	    
	- **`[MAXLEN|MINID limit`]**​  (可选)： **Stream长度控制策略**，这是生产环境中非常重要的参数
	    
	    - `MAXLEN`：限制Stream的**最大消息数量**，如`MAXLEN 1000`只保留最近1000条消息
	        
	    - `MINID`：限制Stream的**最小消息ID**，自动清理比指定ID旧的消息
	        
	    - 通常配合`~`使用：`MAXLEN ~ 1000`表示**近似修剪**，性能更好
	        
	    
	- **`<id>`​**   (通常用`*`)：消息ID，`*`表示由Redis自动生成时间戳序列号
	    
	- **field value**​ (必需)：消息内容，支持多个键值对


### 消费者
#### 简单消费者`XREAD`

![[Pasted image 20251120195138.png]]
这是最基础的消费方式，不涉及消费者组的概念。
- **作用**：从一个或多个 Stream 中读取消息。
- **关键参数**：
    - `COUNT`：每次调用最多返回的消息数量。
    - `BLOCK`：没有消息时，阻塞等待的毫秒数。设置为 `0`表示一直阻塞直到消息到达。这是实现“监听”效果的关键。
    - `STREAMS key`：指定从哪个 Stream 读取
    - ID：读取的起始 ID。
        - `0`：表示从 Stream 的第一条消息开始读取。
        - `$`：表示只读取**在当前调用之后**新到达的消息。
			- **消息被漏读**： 如果在处理一条消息的过程中，队列里新增了多条消息，下一次用 `$`读取时，只会拿到最新的一条，导致中间的**消息被漏读**。
				- 适用场景：因此，这种方式仅适用于对消息丢失不敏感或消费者处理速度极快的简单场景。
- **特点**：
	- **消息可回溯：** 消息被存放在消费者共享的消息队列中，不会因为被读取就被取出
	- **一个消息可以被多个消费者读取：** 消息被存放在消费者共享的消息队列中，不会因为被读取就被取出
	- **可以阻塞读取：** 适用BLOCK指定最大阻塞时间
	- **有消息漏读的风险：** 适用$指定读取监听起的新到达消息，会漏掉之前的消息


#### 消费者组
![[3e6d4cac8f9f6e1728261ea474b7b7d9 1.jpeg]]

##### 管理消费者组
包括创建消费者组，加入消费者，移除消费者，删除消费者组
![[3bb81bd03755025038a3538bfc22206f.jpeg]]
##### 从消费组内读取- `XREADGROUP`

 ![[04d55b06f6c444ace7c92eb324fcc516 1.jpeg]]
 **关键点**：多个 `worker`同时执行上述命令，它们会**自动进行负载均衡**，队列中的消息会分摊给不同的 `worker`并行处理，大大加快消费速度。

##### 确认消息 - `XACK`

这是实现“至少消费一次”语义的保证。

- **作用**：消费者处理完消息后，向服务器发送确认。确认后，该消息才会从消费者组的待处理列表（PENDING List）中移除。
- **关键参数**：
    
    - `key group messageID`：指定要确认的消息。
    
- **重要性**：如果消费者获取消息后崩溃，未能发送 `XACK`，那么这条消息会一直留在 PENDING 列表中。管理员可以通过其他命令发现它，并将其重新分派给健康的消费者处理，从而保证消息不丢失。
##### 监控待处理消息 -`XPENDING`

![[287242b017808d822267786e962f2b83.jpeg]]
- **作用**：查看指定消费者组中所有**已读取但未确认**（即处于 `PENDING`状态）的消息的详细信息。
    
1. **基础监控**：`XPENDING key group`
    
    - 返回pending消息的摘要：消息数量、最早消息ID、最晚消息ID、消费者列表
        
    
2. **详细查看**：`XPENDING key group start end count`
    
    - `start end`：ID范围，可以用`-`和`+`表示最小和最大ID
        
    - `count`：返回的消息数量限制
        
    
3. **按消费者筛选**：`XPENDING key group start end count consumer`
    
    - 查看特定消费者的pending消息
        
    
4. **按空闲时间筛选**：`XPENDING key group IDLE min-idle-time ...`
    
    - `IDLE min-idle-time`：筛选空闲时间超过指定毫秒的消息（用于找出卡住的消息）

```
# 查看pending消息摘要
XPENDING mystream mygroup

# 查看前10条pending消息的详细信息
XPENDING mystream mygroup - + 10

# 查找空闲超过5分钟的消息（可能需处理）
XPENDING mystream mygroup IDLE 300000 - + 10
```
## 与 Spring Boot 集成

Redis Stream 是一个功能强大的实时消息流数据结构，非常适合在 Spring Boot 中构建可靠的消息队列和事件驱动系统。下面这个表格汇总了它的核心适用场景，可以帮助你快速建立整体认知。

| 场景分类 | 核心诉求 | 关键 Redis Stream 特性 |
| :--- | :--- | :--- |
| 异步任务处理 | 解耦耗时操作（如邮件发送、文件处理），提升主流程响应速度。 | 消息持久化、消费者组实现多实例负载均衡。 |
| 微服务间通信 | 服务间可靠、异步的消息传递，实现最终一致性。 | 消息有序性、At-Least-Once 投递语义。 |
| 实时事件流处理 | 处理连续的事件流（如用户行为追踪、监控指标）。 | 基于时间戳的严格顺序、支持多个消费者组独立消费。 |
| 日志收集与分发 | 集中收集日志，并分发给多个分析服务。 | 消费者组实现“发布-订阅”模式，同一消息可被不同组多次消费。 |

###  配置和适用（生产者和消费者）


#### 1. 项目依赖与配置
首先，在 `pom.xml` 中添加必要的依赖，并配置 `application.yml`。

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379
    # 根据实际情况配置密码和数据库
    # password: your-passwor
    # database: 0
```

#### 2. 消息生产者
生产者负责向指定的 Stream 发送消息。
注意要统一序列化和反序列化方式
在这样,添加的是一个Map
- RedisTemplate中使用Map保存Message
- lua中则表现为添加多个键值对
那么我们在配置消费者的适合，就要统一序列化方式
- 如果生产者存Map，而消费者的监听器容器指定取String，那么得到的不是一个JSON，而是Map的第一个键值对的值
##### 使用 `RedisTemplate`。
1. 构建`Map<String, Object>`，填充字段和数据
2. 使用语句生产消息：
	- ` redisTemplate.opsForStream() .add(STREAM_KEY, messageMap)`


```java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service
public class MessageProducerService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String STREAM_KEY = "mystream"; // 定义Stream的名称

    public MessageProducerService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * 发送消息到Redis Stream
     * @param field1 消息字段1
     * @param field2 消息字段2
     * @return 返回消息ID
     */
    public String sendMessage(String field1, String field2) {
        Map<String, Object> messageMap = new HashMap<>();
        messageMap.put("field1", field1);
        messageMap.put("field2", field2);
        // 还可以添加其他业务字段，如时间戳、业务ID等
        messageMap.put("timestamp", System.currentTimeMillis());

        // 使用XADD命令添加消息。'*' 表示由Redis自动生成唯一递增的Message ID
        return redisTemplate.opsForStream()
                           .add(STREAM_KEY, messageMap);
    }
}
```
##### 使用lua脚本
![[Pasted image 20251121142528.png]]


#### 3. 消息消费者示例（含消费者组）
这是实现可靠消费和负载均衡的关键。消费者组确保一条消息只会被组内的一个消费者处理 。
#####  lua脚本实现

##### 手动轮询（Pull）
**消费者主动拉取**。一个无限循环不断向Redis询问：“有新消息吗？
![[251a8615e65734639753aec5c0df6f42.jpeg]]
##### 事件驱动
**消息到达时被动触发**。Spring容器在背后监听，当有新消息时，**主动推送**给你的 `onMessage`方法。

配置[[RedisStream的事件驱动型消费者配置]]:**常驻的、后台的消息消费调度器**

