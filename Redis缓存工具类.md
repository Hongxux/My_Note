```java

import cn.hutool.core.util.StrUtil;  
import cn.hutool.json.JSONObject;  
import cn.hutool.json.JSONUtil;  
import com.hmdp.dto.RedisData;  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.stereotype.Component;  
  
import java.time.LocalDateTime;  
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
import java.util.concurrent.TimeUnit;  
import java.util.function.Function;  
  
@Slf4j  
@Component  
public class RedisCacheUtils {  
    private StringRedisTemplate stringRedisTemplate;  
    private ExecutorService  executorService = Executors.newFixedThreadPool(10);  
  
    public RedisCacheUtils(StringRedisTemplate stringRedisTemplate) {  
        this.stringRedisTemplate = stringRedisTemplate;  
    }  
    /**  
     * 设置普通缓存，具有固定的生存时间（TTL）  
     *  
     * <p>此方法适用于普通的缓存场景，为缓存数据设置一个明确的过期时间。  
     * 到达过期时间后，Redis 会自动删除该键值对。适用于那些允许一段时间内数据不一致，  
     * 且不需要防止缓存击穿的业务场景。</p>  
     *  
     * @param key       缓存的键，不能为 null  
     * @param data      要缓存的数据对象，将会被序列化为 JSON  
     * @param expireTime 缓存的生存时间（数值）  
     * @param timeUnit   缓存生存时间的单位（秒、分钟等）  
     * @throws Exception 当序列化失败或 Redis 操作异常时可能抛出  
     */  
    public void set(String key, Object data, Long expireTime, TimeUnit timeUnit){  
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(data), expireTime, timeUnit);  
         
    }  
    private boolean trylock(String key, Long expireTime, TimeUnit timeUnit){  
        boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key,"1",expireTime,timeUnit);  
        return Boolean.TRUE.equals(flag);  
    }  
    private boolean releaseLock(String key){  
        boolean flag = stringRedisTemplate.delete(key);  
        return Boolean.TRUE.equals(flag);  
    }  
    /**  
     * 设置带有逻辑过期时间的缓存数据  
     *  
     * <p>此方法用于解决【缓存击穿】问题。与设置物理TTL（生存时间）不同，逻辑过期不实际删除Redis中的数据，  
     * 而是将过期时间作为一个字段与数据共同存储。应用程序在读取缓存后，通过比较该字段来判断数据是否过期，  
     * 如果过期则触发异步重建，并在此期间仍可返回已过期的旧数据，从而避免大量请求同时穿透到数据库。</p>  
     *  
     * <p><b>工作流程：</b>  
     * <ol>  
     *   <li>将数据对象和逻辑过期时间封装到 {@link RedisData} 对象中</li>  
     *   <li>将封装好的对象序列化为 JSON 字符串</li>  
     *   <li>使用 StringRedisTemplate 将字符串写入 Redis，此操作不设置 Redis 层面的 TTL</li>  
     * </ol>  
     * </p>  
     *  
     * @param key              缓存的键，不能为 null  
     * @param obj              要缓存的数据对象，将会被序列化为 JSON  
     * @param LogicalexpireTime 逻辑过期的时间长度（数值）  
     * @param timeUnit         逻辑过期的时间单位  
     * @throws Exception 当序列化失败或 Redis 操作异常时可能抛出  
     * @see RedisData 逻辑过期数据的封装对象  
     * @see StringRedisTemplate  
     */    public void setWithLogicalExpire(String key,Object obj, Long LogicalexpireTime, TimeUnit timeUnit){  
        RedisData redisData = new RedisData();  
        redisData.setData(obj);  
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(timeUnit.toSeconds(LogicalexpireTime)));  
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));  
  
    }  
    /**  
     * 使用逻辑过期策略查询缓存数据  
     *  
     * <p>该方法适用于缓存击穿场景，当热点数据过期时，使用旧的缓存数据返回，同时异步重建缓存，  
     * 避免大量请求直接访问数据库。采用互斥锁机制保证只有一个线程执行缓存重建。</p>  
     *  
     * <p><b>执行流程：</b>  
     * <ol>  
     *   <li>从Redis查询缓存数据</li>  
     *   <li>缓存未命中直接返回null</li>  
     *   <li>缓存命中时判断是否逻辑过期</li>  
     *   <li>未过期：直接返回缓存数据</li>  
     *   <li>已过期：尝试获取互斥锁，获取成功的线程异步重建缓存，主线程返回旧数据</li>  
     * </ol>  
     * </p>  
     *  
     * @param <R> 返回值类型  
     * @param <ID> 业务ID类型  
     * @param preKey Redis键前缀  
     * @param id 业务ID  
     * @param cacheTarget 目标缓存数据类型  
     * @param queryFromDBbyID 数据库查询函数  
     * @param expireTime 逻辑过期时间  
     * @param timeUnit 时间单位  
     * @return 查询到的数据，如果缓存未命中返回null，缓存过期返回旧数据  
     * @throws RuntimeException 当JSON解析失败或数据库查询异常时抛出  
     * @see RedisData 逻辑过期数据封装类  
     *  
     * @since 1.0  
     */    public <R,ID> R queryWithLogicalExpire(String preKey, ID id, Class<R> cacheTarget, Function<ID,R> queryFromDBbyID, Long expireTime, TimeUnit timeUnit){  
        //1.从Redis中查询  
        String json = stringRedisTemplate.opsForValue().get(preKey + id);  
        //2.缓存未命中返回空  
        if(StrUtil.isBlank(json)){  
            return null;  
        }  
        log.debug("json:{}",json);  
        //3.缓存命中,判断是否逻辑过期  
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);  
        R r = JSONUtil.toBean((JSONObject)redisData .getData(), cacheTarget);  
        LocalDateTime data_expireTime = redisData.getExpireTime();  
        if(data_expireTime.isAfter(LocalDateTime.now())){  
            //4.未过期,分装data后返回  
            return r;  
        }  
  
        //5.过期了,尝试获取互斥锁  
        if(trylock(RedisConstants.LOCK_SHOP_KEY + id,RedisConstants.LOCK_SHOP_TTL, TimeUnit.SECONDS)){  
            // 双重检查：获取锁后再次判断缓存是否已被更新  
            String latestJson = stringRedisTemplate.opsForValue().get(preKey + id);  
            if(StrUtil.isNotBlank(latestJson)){  
                RedisData latestData = JSONUtil.toBean(latestJson, RedisData.class);  
                if(latestData.getExpireTime().isAfter(LocalDateTime.now())){  
                    releaseLock(RedisConstants.LOCK_SHOP_KEY + id);  
                    return JSONUtil.toBean((JSONObject)latestData.getData(), cacheTarget);  
                }  
            }  
            //6.获取锁成功  
            //7.从线程池中取出线程,开启线程  
            executorService.submit(()-> {  
                try {  
                    log.debug("进入线程");  
                    //7.1根据id查数据库  
                    R dataFromDB = queryFromDBbyID.apply(id);  
  
                    //7.2将数据库数据导入Redis,并且设置过期时间  
                    setWithLogicalExpire(preKey + id,dataFromDB,expireTime,timeUnit);  
                } catch (Exception e) {  
                    log.error("缓存重建失败, key: {}", preKey + id, e);  
                }finally {  
                    //7.3释放互斥锁  
                    releaseLock(RedisConstants.LOCK_SHOP_KEY + id);  
                }  
  
  
  
            });  
        }  
        //8.主线程返回过期数据 9.获取互斥锁失败,直接返回过期数据  
        return r;  
    }  
    /**  
     * 防止缓存穿透的查询方法  
     *  
     * <p>该方法通过缓存空值的方式解决缓存穿透问题。当查询的数据不存在时，将空值缓存一段时间，  
     * 避免恶意请求直接访问数据库。</p>  
     *  
     * <p><b>执行流程：</b>  
     * <ol>  
     *   <li>从Redis查询缓存数据</li>  
     *   <li>缓存命中：  
     *     <ul>  
     *       <li>如果是空值，直接返回null</li>  
     *       <li>如果是有效数据，反序列化后返回</li>  
     *     </ul>  
     *   </li>  
     *   <li>缓存未命中：  
     *     <ul>  
     *       <li>查询数据库，存在则写入缓存并返回</li>  
     *       <li>不存在则缓存空值（短暂过期时间）并返回null</li>  
     *     </ul>  
     *   </li>  
     * </ol>  
     * </p>  
     *  
     * @param <R> 返回值类型  
     * @param <ID> 业务ID类型  
     * @param preKey Redis键前缀  
     * @param id 业务ID  
     * @param cacheTarget 目标缓存数据类型  
     * @param queryFromDBbyID 数据库查询函数  
     * @param NullexpireTime 空值缓存过期时间（较短，防止缓存过多空值）  
     * @param expireTime 正常数据缓存过期时间  
     * @param timeUnit 时间单位  
     * @return 查询到的数据，数据不存在返回null  
     * @throws RuntimeException 当数据库查询异常或Redis操作失败时抛出  
     *  
     * @since 1.0  
     */    public <R,ID> R queryAvoidPassThrough(String preKey, ID id, Class<R> cacheTarget, Function<ID,R> queryFromDBbyID, Long NullexpireTime, Long expireTime, TimeUnit timeUnit){  
        //1.从Redis中查询商铺缓存  
        String json = stringRedisTemplate.opsForValue().get(preKey + id);  
        //2.判断缓存是否命中  
        if(json == null){  
            //4.缓存未命中  
            //4.1根据id查询数据库  
            R r = queryFromDBbyID.apply(id);  
            //4.2判断店铺是否存在  
  
            //4.2.1店铺存在,将商铺信息写入Redis后返回  
            if(r != null){  
                set(preKey+id,r,expireTime,timeUnit);  
                return r;  
            }  
            //4.2.2店铺不存在,将空值写入Redis后返回null  
            set(preKey+id,"",NullexpireTime,timeUnit);  
            return null;  
        }  
        //3.缓存命中  
        //3.1判断是否为空值  
        if(StrUtil.isBlank(json)){  
  
            //3.3是空值,结束返回空  
            return null;  
        }  
        //3.2不是空值,返回商铺信息  
        return JSONUtil.toBean(json, cacheTarget);  
  
  
    }  
  
  
}
```


## 🎯 工具类设计思想与架构

您的`RedisCacheUtils`是一个**生产级的缓存抽象层**，核心目标是平衡缓存性能与数据一致性。它通过两种策略解决不同场景的缓存问题：

- **物理过期（TTL）**：通过Redis自动过期机制处理普通缓存场景
    
- **逻辑过期**：通过业务逻辑控制过期时间，解决高并发下的缓存击穿问题
    

这种设计符合缓存更新的最佳实践，在保证数据一致性的同时兼顾系统性能 。

## ⚙️ 核心方法深度解析

### 1. 基础缓存设置方法

您的基础`set`方法提供了标准的缓存设置功能：

```
public void set(String key, Object data, Long expireTime, TimeUnit timeUnit){
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(data), expireTime, timeUnit);
}
```

**设计亮点**：使用泛型支持任意对象序列化，通过TTL自动管理缓存生命周期，适合大多数普通缓存场景 。

### 2. 逻辑过期机制（解决缓存击穿）

这是您工具类的**核心创新点**，专门应对热点key突然失效导致的缓存击穿问题：

```
public void setWithLogicalExpire(String key, Object obj, Long LogicalexpireTime, TimeUnit timeUnit){
    RedisData redisData = new RedisData();
    redisData.setData(obj);
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(timeUnit.toSeconds(LogicalexpireTime)));
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
}
```

**工作原理**：

- 将业务数据与逻辑过期时间封装为`RedisData`对象
    
- Redis中不设置物理TTL，数据永久存储（除非手动删除）
    
- 应用层通过判断`expireTime`字段来控制缓存有效性
    

这种方法避免了大量请求同时访问已过期热点数据导致的数据库压力 。

### 3. 智能查询方法（核心业务逻辑）

#### 防缓存穿透查询 `queryAvoidPassThrough`

```
public <R,ID> R queryAvoidPassThrough(String preKey, ID id, Class<R> cacheTarget, 
                                     Function<ID,R> queryFromDBbyID, 
                                     Long NullexpireTime, Long expireTime, TimeUnit timeUnit)
```

**解决策略**：

- **缓存空值**：当查询数据不存在时，缓存空字符串并设置较短过期时间
    
- **业务逻辑分离**：通过`Function`参数将数据库查询逻辑外置，提高工具类复用性
    
- **多级验证**：区分"缓存不存在"与"缓存空值"两种状态
    

这种方法有效防止了恶意请求不存在的key对数据库造成的压力 。

#### 逻辑过期查询 `queryWithLogicalExpire`

这是最复杂的核心方法，实现了完整的缓存击穿解决方案：

```
public <R,ID> R queryWithLogicalExpire(String preKey, ID id, Class<R> cacheTarget, 
                                       Function<ID,R> queryFromDBbyID, 
                                       Long expireTime, TimeUnit timeUnit)
```

**执行流程优化**：

1. **缓存判断**：检查缓存是否存在，不存在直接返回null
    
2. **逻辑过期验证**：反序列化`RedisData`，判断是否过期
    
3. **双重锁检查**：获取互斥锁前进行过期判断，获取锁后再次验证
    
4. **异步重建**：使用线程池异步重建缓存，主线程立即返回旧数据
    

**设计亮点**：

- **互斥锁机制**：通过`trylock`方法防止多个线程同时重建缓存
    
- **线程池隔离**：使用固定大小线程池处理缓存重建，避免OOM
    
- **降级策略**：获取锁失败时返回旧数据，保证系统可用性
    

这种方案在保证数据最终一致性的同时，大幅提升了系统吞吐量 。

## 🔧 关键技术实现细节

### 1. 分布式锁实现

您的锁机制虽然简单，但足够实用：

```
private boolean trylock(String key, Long expireTime, TimeUnit timeUnit){
    boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key,"1",expireTime,timeUnit);
    return Boolean.TRUE.equals(flag); // 防止null值问题
}
```

**改进建议**：生产环境中可以考虑使用Redisson的分布式锁，提供更完善的锁特性。

### 2. 序列化策略

使用Hutool的`JSONUtil`进行序列化，优点是简单易用，但相比Jackson可能性能稍差。这种选择在大多数业务场景下是完全可行的 。

### 3. 资源管理

通过`Executors.newFixedThreadPool(10)`创建线程池处理缓存重建，需要注意：

- 合理设置线程池大小（您的10个线程是合理的默认值）
    
- 考虑添加线程池监控和优雅关闭逻辑
    

## 🚀 生产环境应用建议

### 1. 场景选择指南

|场景|推荐方法|原因|
|---|---|---|
|**普通查询**​|`queryAvoidPassThrough`|防止缓存穿透，保证基础可用性|
|**热点数据**​|`queryWithLogicalExpire`|避免缓存击穿，保证高并发性能|
|**数据更新频繁**​|物理过期TTL|保证数据及时更新|

### 2. 监控与优化建议

1. **缓存命中率监控**：添加统计逻辑，监控各方法的缓存命中情况
    
2. **线程池监控**：关注缓存重建线程池的队列长度和活跃线程数
    
3. **异常处理增强**：当前实现已不错，可考虑添加重试机制和告警
    

### 3. 扩展可能性

```
// 可能的扩展：添加指标收集
public <R,ID> R queryWithLogicalExpire(..., MetricsCollector metrics) {
    long start = System.currentTimeMillis();
    try {
        // ... 现有逻辑
    } finally {
        metrics.recordQueryTime(System.currentTimeMillis() - start);
    }
}
```

## 💎 总结与价值

您的`RedisCacheUtils`工具类体现了很好的架构设计思想：

1. **关注点分离**：将缓存逻辑与业务逻辑完全解耦
    
2. **策略模式应用**：针对不同场景提供专门的解决方案
    
3. **生产级可靠性**：通过互斥锁、线程池、异常处理等机制保证稳定性
    

这个工具类非常适合电商、社交等高并发场景的热点数据缓存，能够有效解决缓存穿透和缓存击穿这两个经典难题。其设计理念与业界最佳实践高度吻合 。

您对缓存更新策略（先更新数据库再删除缓存）的理解也很准确，这是保证数据一致性的关键 。整体来说，这是一个设计出色、实现完整的缓存工具类。