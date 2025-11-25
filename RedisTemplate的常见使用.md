RedisTemplate 是 Spring Data Redis 提供的核心工具类，它封装了与 Redis 交互的细节，支持多种数据结构的操作。下面用一个表格总结其核心用法，并补充关键概念和示例。

|功能类别|操作类型|核心方法示例|主要应用场景|
|---|---|---|---|
|**通用键操作**|删除键|`delete(key)`/ `delete(Collection keys)`|清理缓存或无效数据|
||判断键是否存在|`hasKey(key)`|验证缓存有效性|
||设置过期时间|`expire(key, timeout, timeUnit)`|实现缓存自动失效|
|**字符串 (String)**|设置/获取值|`opsForValue().set(key, value)`/ `get(key)`|缓存简单对象、计数器|
||设置带过期时间的值|`opsForValue().set(key, value, timeout, timeUnit)`|缓存会话、验证码|
||递增/递减|`opsForValue().increment(key, delta)`|实现点赞数、访问量统计|
|**哈希 (Hash)**|设置/获取字段值|`opsForHash().put(key, field, value)`/ `get(key, field)`|存储对象属性（如用户信息）|
||获取所有字段|`opsForHash().entries(key)`|获取完整对象|
||批量操作|`opsForHash().putAll(key, map)`|一次性更新多个对象属性|
|**列表 (List)**|左右端推入/弹出|`opsForList().leftPush(key, value)`/ `rightPop(key)`|消息队列、最新消息列表|
||获取范围元素|`opsForList().range(key, start, end)`|分页查询|
|**集合 (Set)**|添加/判断成员|`opsForSet().add(key, values)`/ `isMember(key, value)`|标签、好友共同关注|
||获取所有成员|`opsForSet().members(key)`|获取所有标签|
|**有序集合 (ZSet)**|添加成员（带分数）|`opsForZSet().add(key, value, score)`|排行榜、延迟队列|
||按分数范围查询|`opsForZSet().reverseRange(key, start, end)`|获取排行榜TopN|


1. **两种操作风格**
    
    - **`opsForXxx()`**：这是最常用和灵活的方式。每次操作都需要显式指定 key 。
        
    - **`boundXxxOps()`**：这是一种“绑定键”的操作。先绑定一个特定的 key，后续所有操作都基于这个键，无需重复指定，使代码更简洁 。
        
    
2. **[[Spring Data Redis中序列化方式|序列化]]策略**：RedisTemplate 需要将 Java 对象序列化后才能存储到 Redis。常用的序列化器有：
    
    - `StringRedisSerializer`：用于键和字符串值的序列化，保证人类可读。
        
    - `Jackson2JsonRedisSerializer`：将对象序列化为 JSON 格式，兼容性好。
        
        通常在配置中会为 key 和 value 分别设置不同的序列化方式，以避免乱码 。
3. **有效期设置**:对于缓存，应该设置有效期，使得数据失效

4. [[setnx]]