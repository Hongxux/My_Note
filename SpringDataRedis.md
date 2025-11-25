在 Spring Boot 中使用 Redis 可以极大提升应用性能，特别是在缓存和数据存取方面。下面这个表格汇总了核心的使用步骤和要点，帮助你快速建立整体认知。

| 关键环节         | 核心操作                                                                | 说明与常用配置                                                                                                                                    |
| ------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **1. 项目依赖**  | 在 `pom.xml`中添加 `spring-boot-starter-data-redis`依赖                   | 引入 Spring Data Redis 支持，默认集成 Lettuce 客户端。                                                                                                  |
| **2. 服务连接**  | 在 `application.properties`或 `application.yml`中配置 Redis 服务器地址、端口、密码等 | 基础配置：`spring.redis.host`, `spring.redis.port`。连接池配置（如使用 Lettuce）：可配置 `max-active`, `max-idle`等参数。                                          |
| **3. 数据操作**  | 注入并使用 `RedisTemplate`或 `StringRedisTemplate`                        | `RedisTemplate`可处理任意对象（需序列化），`StringRedisTemplate`主要处理字符串。可通过 `opsForValue()`, `opsForHash()`等操作不同数据结构。                                    |
| **4. 序列化配置** | 自定义 `RedisTemplate`的序列化方式（常见配置）                                     | 解决默认 JDK 序列化可能带来的可读性差等问题。常见配置：Key 使用 `StringRedisSerializer`，Value 使用 `GenericJackson2JsonRedisSerializer`或 `Jackson2JsonRedisSerializer`。 |
| **5. 声明式缓存** | 使用 `@EnableCaching`, `@Cacheable`, `@CacheEvict`等注解                 | 方便地将方法返回值缓存或从缓存中清除数据。                                                                                                                      |

### 🔧 详细配置与操作

#### 1. 添加依赖与基础配置

在 Maven 项目的 `pom.xml`中添加依赖后，在 `application.yml`中进行基础配置和连接池配置的例子如下：

```
spring:
  redis:
    host: localhost # Redis 服务器地址
    port: 6379      # Redis 服务器端口
    password:       # 密码（如果没有可不设置）
    timeout: 5000   # 连接超时时间（毫秒）
    lettuce:
      pool:
        max-active: 8   # 连接池最大连接数
        max-idle: 8     # 连接池中的最大空闲连接
        min-idle: 0     # 连接池中的最小空闲连接
```

#### 2. 自定义 RedisTemplate 解决序列化问题
[[Spring Data Redis中序列化方式]]

#### 3. 使用 Spring Cache 注解

Spring Cache 抽象简化了缓存使用。首先在启动类上添加 `@EnableCaching`开启缓存支持。

然后在方法上使用缓存注解：

- `@Cacheable`：在方法执行前检查缓存，若存在则直接返回缓存值。
    
    ```
    @Service
    public class ProductService {
        @Cacheable(value = "product", key = "#id") // 缓存名为"product"，key为方法参数id
        public Product findProductById(String id) {
            // 模拟从数据库查询耗时操作
            return productRepository.findById(id);
        }
    }
    ```
    
- `@CacheEvict`：用于删除缓存数据，常用于更新或删除操作后。
    
    ```
    @CacheEvict(value = "product", key = "#id") // 删除指定key的缓存
    public void updateProduct(Product product) {
        productRepository.update(product);
    }
    ```
    
    还可以通过自定义 `CacheManager`来配置统一的缓存行为，例如设置默认过期时间。
    

#### 4. [[RedisTemplate的常见使用]]
### 💡 主要应用场景

1. **数据缓存**：这是最经典的场景。将数据库热点数据或耗时计算结果存入 Redis，显著提高查询速度和系统吞吐量。
    
2. **Session 共享**：在分布式或集群环境中，将用户 Session 集中存储到 Redis，实现多服务实例间的 Session 共享。
    
3. **分布式锁**：利用 Redis 的原子操作（如 `SET key value NX PX timeout`）实现简单的分布式锁，控制分布式系统对共享资源的访问。
    
4. **计数器/限流**：利用 Redis 中 `String`类型的 `INCR`、`INCRBY`等原子性命令实现计数器功能，可用于统计点击量、点赞数，或结合过期时间实现接口访问频率限制。
    

### ⚠️ 使用建议与注意事项

- **序列化一致性**：确保生产环境和消费环境对 Redis 中存储的序列化对象结构（类定义）保持一致，否则在反序列化时可能出现异常。
    
- **缓存失效策略**：合理设置缓存过期时间，避免**缓存雪崩**（大量 key 同时失效）和**缓存穿透**（频繁查询一个不存在的数据）。对于不存在的值也可进行短时间缓存。
    
- **连接池配置**：根据应用并发量合理配置连接池参数（如 `max-active`, `max-wait`），避免资源等待或浪费。
    
- **事务支持**：RedisTemplate 支持事务，但需要注意在 Redis 集群环境下事务的限制。
    
- **大 Key 优化**：避免单个 Key 的 Value 过大（如超过 10KB），可能影响网络传输和 Redis 性能。可考虑对 Value 进行压缩或拆分。