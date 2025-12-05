---
aliases:
  - 批处理
  - Pipeline
tags:
---
## 需求分析
![[Pasted image 20251125172146.png]]
![[Pasted image 20251125172514.png]]
![[Pasted image 20251125172605.png]]

- Redis服务器的执行耗时远远小于网络传输的耗时，以至于可以忽略不计执行命令耗时
	- Redis服务器的执行耗时是微秒级别
	- 网络传输的耗时是毫秒级别
- 解决方法：N条命令批处理，例如mset和hset
	- ![[Pasted image 20251125172647.png]]
		- 但是不要一口气传输太多的命令
			- 带宽占用太多会导致阻塞 
## Pipeline（单机模式下）
MSET虽然可以批处理，但是却只能操作部分数据类型，因此如果有对复杂数据类型的批处理需要，建议使用Pipeline功能:
![[Pasted image 20251125173646.png]]
- 批处理时不建议一次携带太多命令
- Pipeline的多个命令之间不具备原子性
	- 如果有原子性要求可以使用m型命令（如mset，hmset）
## 集群模式下的批处理

如MSET或Pipeline这样的批处理需要在一次请求中携带多条命令，而此时如果Redis是一个集群，那批处理命令的多个key必须落在一个插槽（hash slot）中，否则就会导致执行失败![[Pasted image 20251125194725.png]]

- 解决方案：![[Pasted image 20251125195627.png]]
	-


下面这个表格概括了 Spring 提供的核心批处理方式及其特点。

| 批处理方式              | 核心方法                                     | 主要特点                                       | 适用场景                  |
| ------------------ | ---------------------------------------- | ------------------------------------------ | --------------------- |
| **原生批处理命令**​       | `redisTemplate.opsForValue().multiSet()` | 自动进行 Slot 分组并行执行，**性能极高**，但不支持为每个键单独设置过期时间 | 需要最高性能的纯数据存储          |
| **Pipeline 管道操作**​ | `redisTemplate.executePipelined`         | 可设置复杂操作（如过期时间），自动分组，性能优秀，灵活性高              | 需要设置过期时间或执行多种命令的复杂批处理 |
| **集群事务支持**​        | `redisTemplate.execute(SessionCallback)` | 提供基本的事务保证（注意 Redis 事务无回滚机制），性能较低           | 需要一定事务语义的批处理操作        |


###  Spring 的智能批处理的原理

1. **计算 Slot**：对传入的所有 Key 计算其对应的 Redis 集群哈希 Slot。
    
2. **按节点分组**：根据 Slot 与节点的映射关系，将这些 Key 分组到其对应的实际 Redis 节点上。
    
3. **并行执行**：**为每个节点组创建一个连接，并行地在各个节点上执行批处理命令**。这正是我们想要的手动优化策略的自动化实现。
    

###  使用 Spring 工具进行批处理

#### 1. 原生批处理命令 (推荐用于简单设置)

这是最快捷的方式，适合简单的键值对批量设置。

```
@Autowired
private RedisTemplate<String, Object> redisTemplate; // 或 StringRedisTemplate

public void batchSetWithSpring() {
    Map<String, Object> keyValueMap = new HashMap<>();
    // 准备数据，这些key可能会分布在集群的不同节点上
    keyValueMap.put("user:1001:name", "Alice");
    keyValueMap.put("product:2002:price", 299.99);
    keyValueMap.put("config:site:title", "MySite");

    // Spring 会自动处理集群分发。底层会进行Slot计算、分组，并尽可能并行处理 
    redisTemplate.opsForValue().multiSet(keyValueMap);

    // 对应的批量获取
    List<String> keys = Arrays.asList("user:1001:name", "product:2002:price", "config:site:title");
    List<Object> values = redisTemplate.opsForValue().multiGet(keys);
    System.out.println(values);
}
```

**优点**：代码简洁，性能接近极致。

**缺点**：无法在批量操作中为每个键设置复杂的属性（如过期时间）。

#### 2. Pipeline 管道操作 (推荐用于复杂操作)

当您需要更复杂的操作，例如为每个键设置过期时间时，Pipeline 是更好的选择。Spring 的 Pipeline 在集群环境下同样会自动处理 Slot 分组 。

```
public void batchSetWithPipelineAndExpire() {
    Map<String, Object> keyValueMap = new HashMap<>();
    keyValueMap.put("user:1001:name", "Alice");
    keyValueMap.put("product:2002:price", 299.99);
    // ... 添加更多数据

    Long expireTime = 3600L; // 过期时间1小时

    // 使用 Pipeline 执行
    List<Object> results = redisTemplate.executePipelined(new RedisCallback<Object>() {
        @Override
        public Object doInRedis(RedisConnection connection) throws DataAccessException {
            // 获取Spring配置的序列化器，确保序列化方式一致
            RedisSerializer keySerializer = redisTemplate.getKeySerializer();
            RedisSerializer valueSerializer = redisTemplate.getValueSerializer();

            for (Map.Entry<String, Object> entry : keyValueMap.entrySet()) {
                // 序列化Key和Value
                byte[] serializedKey = keySerializer.serialize(entry.getKey());
                byte[] serializedValue = valueSerializer.serialize(entry.getValue());

                // 设置键值对，并指定过期时间
                connection.set(serializedKey, serializedValue, Expiration.seconds(expireTime), RedisStringCommands.SetOption.UPSERT);
            }
            return null; // 在Pipeline回调中，返回值必须为null
        }
    });
    // results 包含每个操作（本例中是每个set操作）的返回状态列表
    System.out.println("Pipeline 操作完成，结果数: " + results.size());
}
```

**优点**：功能强大，可以执行任意 Redis 命令并设置过期时间等属性，且仍享有批处理性能。

**缺点**：代码稍复杂，需要处理序列化。

