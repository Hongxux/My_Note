
### Caffeine的使用
#### 1. 项目依赖

首先，在项目中引入Caffeine的依赖（以Maven为例）：

```
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version> <!-- 请使用当前最新稳定版本 -->
</dependency>
```

#### 2. 三种创建与使用方式

| 使用方式      | 特点                                         | 适用场景                     |
| --------- | ------------------------------------------ | ------------------------ |
| **手动加载**​ | 完全手动控制`put`和`get`，灵活性最高。                   | 需要精细控制每个缓存操作的场景。         |
| **同步加载**​ | 内置`CacheLoader`，使用`get`方法在缓存不存在时自动加载，线程安全。 | 最常见的场景，缓存逻辑简单且固定。        |
| **异步加载**​ | 返回`CompletableFuture`，异步加载缓存值，避免阻塞。        | 高并发系统，加载缓存值成本高，希望避免阻塞请求。 |

- **手动加载缓存**
    
    这种方式最基础，你需要手动将值放入缓存。
    
    ```
    Cache<String, Object> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();
    
    // 存入缓存
    cache.put("name", "Caffeine");
    // 查找缓存，不存在则返回null
    Object value = cache.getIfPresent("name");
    // 查找缓存，不存在则使用Function加载并存入缓存
    value = cache.get("name", k -> loadDataFromDB(k));
    ```
    
- **同步加载缓存**
    
    通过传入`CacheLoader`，实现“获取-若没有则计算”的原子操作，是推荐的使用方式。
    
    ```
    LoadingCache<String, Object> loadingCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build(new CacheLoader<String, Object>() {
            @Override
            public Object load(String key) throws Exception {
                // 当缓存未命中时，此方法会被自动调用以加载数据
                return loadDataFromDB(key);
            }
        });
    
    // 使用get方法即可，如果缓存不存在，会自动调用load方法
    Object value = loadingCache.get("name");
    // 批量获取
    Map<String, Object> valueMap = loadingCache.getAll(Arrays.asList("key1", "key2"));
    ```
    
- **异步加载缓存**
    
    适合加载过程耗时的场景，避免阻塞调用线程。
    
    ```
    AsyncLoadingCache<String, Object> asyncCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .buildAsync(new CacheLoader<String, Object>() {
            @Override
            public Object load(String key) throws Exception {
                return loadDataFromDB(key);
            }
        });
    
    // 获取一个 CompletableFuture
    CompletableFuture<Object> future = asyncCache.get("name");
    future.thenAccept(value -> {
        // 异步处理获取到的值
        System.out.println(value);
    });
    ```
    
#### 3.核心配置

#####  容量控制
容量控制是缓存的“安全阀”，通过控制缓存的条目和缓存的内存占用，确保缓存不会无限制增长导致内存溢出（OOM）。

- **`maximumSize(long)`**：限制缓存的条目（key-value键值对）
    
    ```
    // 场景：缓存最新10000条用户查询记录
    Cache<String, QueryResult> cache = Caffeine.newBuilder()
        .maximumSize(10000)
        .build();
    ```
    
- **`maximumWeight(long)`与 `weigher`**：
	- `maximumWeight(long)`：限制缓存的占用量
	- `weigher`：每一条目占用量的计算方法
    
    ```
    // 场景：缓存文件内容，总大小不超过100MB
    Cache<String, byte[]> fileCache = Caffeine.newBuilder()
        .maximumWeight(100 * 1024 * 1024) // 100MB
        .weigher((String key, byte[] value) -> value.length)
        .build();
    ```
    

#####  过期策略：

过期策略影响缓存数据和数据库中数据的**一致性程度**

- **`expireAfterWrite`**：**写入后过期**。这是最常用的策略，能**保证数据的最终一致性**。适用于配置信息、商品分类等变化不频繁的静态或准静态数据。例如，商品分类信息可以设置为写入缓存后24小时过期。
    
- **`expireAfterAccess`**：**访问后过期**。这个策略让“活跃”的数据保留更久。非常适合用于缓存用户会话（Session）、热点新闻等，只要数据被频繁访问，就会一直留在缓存中。
    
- **`refreshAfterWrite`**：**写入后刷新**（类似于逻辑过期机制）。这是一个非常智能的策略。它不会在过期后立即阻塞所有请求去加载新值，而是允许一个线程在访问时异步刷新数据，其他请求在刷新完成前**仍能返回旧的（可能已过时）数据**。这能有效预防**缓存击穿**。非常适合缓存热点数据，如首页的焦点图。
    
- **自定义过期（`Expiry`）**：用于需要**为每个缓存条目设置不同TTL**的复杂场景。例如，根据用户等级（普通用户VIP）决定其配置信息的缓存时长。
	
```
// 假设有一个对象，其内部定义了生存时间
class DataObject {
	String data;
	long ttlInSeconds; // 该数据对象的自定义TTL

	DataObject(String data, long ttl) {//字段中可以有要存入的数据，expireAfterCreateTime，expireAfterUpdateTime，expireAfterReadTime，实现精细化配置
		this.data = data;
		this.ttlInSeconds = ttl;
	}
}

Cache<String, DataObject> cache = Caffeine.newBuilder()
		.expireAfter(new Expiry<String, DataObject>() {
			@Override
			public long expireAfterCreate(String key, DataObject value, long currentTime) {
				// 创建时，使用对象自身的ttl
				return TimeUnit.SECONDS.toNanos(value.ttlInSeconds);
			}
			@Override
			public long expireAfterUpdate(String key, DataObject value, long currentTime, long currentDuration) {
				// 更新时，可以重新设置TTL，这里沿用更新后的值自身的TTL
				return TimeUnit.SECONDS.toNanos(value.ttlInSeconds);
			}
			@Override
			public long expireAfterRead(String key, DataObject value, long currentTime, long currentDuration) {
				// 读取时，不重置过期时间，返回剩余的存活时间
				return currentDuration;
			}
		})
		.build();

cache.put("shortLived", new DataObject("短期数据", 2)); // 2秒后过期
cache.put("longLived", new DataObject("长期数据", 10)); // 10秒后过期

```

#####  引用类型：与GC协作管理内存

通过设置不同的引用类型，您可以将缓存条目的生命周期管理部分交给Java的垃圾回收器（GC）。

- **弱引用（`weakKeys`/ `weakValues`）**：当缓存键（Key）或值（Value）除了被缓存引用外，**没有其他强引用**时，它们可以被GC正常回收。这有助于避免内存泄漏，但缺点是可能导致缓存命中率下降。通常用于缓存键或值本身是其他对象的引用的场景。
    
- **软引用（`softValues`）**：当JVM**内存不足**时，即使存在强引用，软引用的对象也会被GC回收。它可以看作是在`maximumSize`之外的一道保险，但因为它会影响GC性能且行为不确定，**在现代Java开发中通常不推荐使用**。
    

#####  高级特性与生产实践

- **异步缓存（`AsyncLoadingCache`）**：当缓存未命中时，数据加载过程（如调用数据库或远程接口）可能是耗时的IO操作。使用异步缓存可以避免这些操作阻塞调用线程，特别适合高并发服务。它会返回一个`CompletableFuture`对象。
    
#### 4.统计
与Guava Cache的统计一样。

```scss
Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .recordStats()
    .build();
```

通过使用Caffeine.recordStats(), 可以转化成一个统计的集合. 通过 Cache.stats() 返回一个CacheStats。CacheStats提供以下统计方法：

```scss
hitRate(): 返回缓存命中率
evictionCount(): 缓存回收数量
averageLoadPenalty(): 加载新值的平均时间
```
#### 5.写入外部存储
CacheWriter 方法可以将缓存中所有的数据写入到第三方。

```typescript
LoadingCache<String, Object> cache2 = Caffeine.newBuilder()
    .writer(new CacheWriter<String, Object>() {
        @Override public void write(String key, Object value) {
            // 写入到外部存储
        }
        @Override public void delete(String key, Object value, RemovalCause cause) {
            // 删除外部存储
        }
    })
    .build(key -> function(key));
```

如果你有多级缓存的情况下，这个方法还是很实用。


### 实现原理

Caffeine的高性能源于其精巧的算法和并发设计。

#### 1. 淘汰算法：W-TinyLFU

这是Caffeine取代Guava Cache成为“王者”的核心原因，解决了传统LRU和LFU的缺陷。
![[Pasted image 20251124204527.png]]
- **LRU问题**：只关注数据的“新鲜度”（最近是否使用）
	- 无法应对**扫描性操作**（如一次性的全表查询），可能导致热点数据被冲掉。
    
- **LFU问题**：只关注数据的“热度”（访问频率）
	- 无法及时**淘汰掉陈年的热点数据**，缺乏时间敏感性。
    
- **[[W-TinyLFU]]的解决方案**


#### 2. 高性能读写：缓冲队列与异步机制

Caffeine认为**读操作是频繁的，写操作是偶尔的**。

- **读缓冲**：读操作完成后，会将本次读信息放入一个**无锁的环形队列**，然后立即返回。由后台线程**异步**地从队列中取出记录，更新在`Count-Min Sketch`中的频率。这将耗时的频率更新操作与读请求解耦，极大提升了读并发性能。
    
- **写缓冲**：与读缓冲类似，写事件也会被放入一个**多生产者单消费者队列**，由后台线程异步处理，从而提升写并发能力。
    

#### 3. 过期管理：时间轮算法
![[Pasted image 20251124211526.png]]
如果为每个缓存项都启动一个`TimerTask`，性能开销巨大。Caffeine使用了**时间轮**来高效管理大量定时任务。

- **工作原理**：可以把它想象成一个时钟，盘面分成多个格子（桶），每个格子代表一个时间区间。将过期时间相近的缓存项放入同一个格子。后台线程只需每隔一个时间区间检查当前格子内的数据并清理即可。这避免了频繁的全局遍历，将过期清理的复杂度从O(n)降至接近O(1)。
    

### 💯 第四部分：面试必备知识点梳理

**1. 必考基础**

- 能说出Caffeine的几种创建方式（手动、同步、异步）。
    
- 清楚核心配置策略（`maximumSize`, `expireAfterWrite/Access`, `refreshAfterWrite`）及其应用场景。
    
- 知道如何开启统计功能（`.recordStats()`）并通过`cache.stats().hitRate()`查看命中率。
    

**2. 原理深度（区分水平的关键）**

- **能清晰对比LRU、LFU和W-TinyLFU**，说明W-TinyLFU如何取长补短。
    
- 能简要说明**Count-Min Sketch**的原理和优势（空间换时间，允许误差的频率统计）。
    
- 理解**读/写缓冲**和**时间轮**是如何提升并发性能和降低管理开销的。
    

**3. 场景与陷阱**

- **缓存雪崩/穿透/击穿**：虽然Caffeine是本地缓存，但这些概念是通用的。能结合Caffeine的特性（如`LoadingCache`的`get`方法是原子的）谈解决方案。
    
- **何时使用本地缓存**：Caffeine适合缓存**数据量不大、更新不频繁、访问频繁**的数据。对于**数据量大、需要分布式共享**的场景，应使用Redis等分布式缓存，或采用**多级缓存**架构（Caffeine作L1，Redis作L2）。
    

**4. 与Spring集成**

- 了解如何在Spring Boot中通过`@Cacheable`, `@CacheEvict`等注解无缝集成Caffeine。
    
    ```
    @Configuration
    @EnableCaching
    public class CacheConfig {
        @Bean
        public CacheManager cacheManager() {
            CaffeineCacheManager cacheManager = new CaffeineCacheManager();
            cacheManager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(10, TimeUnit.MINUTES));
            return cacheManager;
        }
    }
    
    @Service
    public class UserService {
        @Cacheable(value = "users", key = "#id")
        public User findById(Long id) {
            // ... 查询数据库
        }
    }
    ```
    

为了让你对Caffeine的核心工作机制有一个更直观的整体印象，特别是其高并发读写和高效过期清理的关键设计，你可以参考下面的序列图，它概括了这些核心过程的交互：
