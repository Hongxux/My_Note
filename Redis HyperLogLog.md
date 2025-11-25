以一定的误差率来获得低内存占用量
使用场景:**统计目标是基数（去重计数）、数据量巨大、允许微小误差**

|特性维度|HyperLogLog|传统 Set / Bitmap|
|---|---|---|
|**内存占用**​|**固定约 12KB**​|随元素数量线性增长|
|**统计结果**​|**近似值**，标准误差约 0.81%|精确值|
|**是否存储元素本身**​|否，仅用于计数|是|
|**处理海量数据能力**​|极强，可统计最多 2^64 个不重复元素|受内存限制严重|
|**典型应用**​|独立访客(UV)统计、大规模数据去重|需要精确成员查询的小数据集|

### 🔮 核心原理简介

HyperLogLog 的精妙之处在于它运用了概率统计原理。当您添加一个元素时，HyperLogLog 会使用哈希函数将其转换为一个64位的比特串 。这个比特串可以看作是一连串的伯努利试验（比如抛硬币），算法通过统计**比特串开头连续0的最大数量**来估算基数 。简单理解就是，您观察到一次“抛了10次才出现正面”的罕见事件，那么大概率已经进行了非常多次的抛掷试验。

为了降低误差，HyperLogLog 使用了 **16384 (2^14) 个寄存器（桶）**​ 来进行分桶平均 。它将哈希值的一部分用于定位桶，另一部分用于计算连续0的个数，最后通过调和平均公式综合所有桶的信息，得出最终的基数估计值 。这种设计使得它在占用极小内存的同时，能处理海量数据的去重统计。

### ⚙️ 核心命令与用法

Redis 为 HyperLogLog 提供了三个核心命令，均以 `PF`开头（以致敬该算法的创始人 Philippe Flajolet）。

|命令|语法|作用与示例|
|---|---|---|
|**PFADD**​|`PFADD key element [element ...]`|向 HyperLogLog 中添加元素。如果添加后基数估计值发生变化，返回 1，否则返回 0。【例】`PFADD uv:2025-01-01 user1 user2`|
|**PFCOUNT**​|`PFCOUNT key [key ...]`|返回 HyperLogLog 的基数估计值。可同时查询多个 key，返回它们并集的基数。【例】`PFCOUNT uv:2025-01-01`或 `PFCOUNT uv:mon uv:tue uv:wed`|
|**PFMERGE**​|`PFMERGE destkey sourcekey [sourcekey ...]`|将多个 HyperLogLog 合并到一个新的 HyperLogLog 中，用于计算多日/多源的合并基数。【例】`PFMERGE uv:weekly uv:mon uv:tue`|

### 💡 主要应用场景

HyperLogLog 的用武之地是那些**允许微小误差、需要处理海量数据、且以节省内存为首要目标**的基数统计场景 。

- **网站/应用的独立访客数统计**：这是最经典的场景。统计每天、每月的 UV，无需存储具体的用户ID，极大节省空间 。
    
- **大规模流式数据去重**：例如，统计搜索关键词的不同个数、监控日志中不同错误类型的数量等 。
    
- **数据库查询或API调用结果去重**：可用于快速估算不同结果的大致数量，避免昂贵的 `DISTINCT COUNT`查询 。
    

### ⚠️ 重要限制与注意事项

选择 HyperLogLog 前，请务必了解其局限性：

- **结果是近似值**：存在约 0.81% 的标准误差，不适合需要精确计数的场景 。
    
- **无法查询具体元素**：由于不存储原始数据，您无法回答“用户A今天是否访问过？”这类问题 。
    
- **不支持删除操作**：向 HyperLogLog 添加元素后，无法从中移除某个元素。它是一个“只增”的数据结构 。
    

### 📝 代码示例

以下是在 Spring Boot 项目中通过 RedisTemplate 使用 HyperLogLog 的简单示例：

```
@Component
public class UvStatisticsService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public void recordUv(String date, String userId) {
        String key = "uv:" + date;
        // 添加用户ID到当天的UV统计中
        redisTemplate.opsForHyperLogLog().add(key, userId);
    }

    public long getUv(String date) {
        String key = "uv:" + date;
        // 获取当天的UV估计值
        return redisTemplate.opsForHyperLogLog().size(key);
    }

    public long getWeeklyUv() {
        // 合并一周的UV数据，并返回总UV
        List<String> keys = Arrays.asList("uv:2025-01-01", "uv:2025-01-02", ...);
        String unionKey = "uv:weekly:union";
        redisTemplate.opsForHyperLogLog().union(unionKey, keys.toArray(new String[0]));
        return redisTemplate.opsForHyperLogLog().size(unionKey);
    }
}
```

### 💎 总结

当您的业务满足**统计目标是基数（去重计数）、数据量巨大、允许微小误差**这三个条件时，HyperLogLog 就是一个非常强大的工具。它用极小的内存开销解决了大规模数据集的去重统计难题。

希望这些信息能帮助您更好地理解和使用 HyperLogLog。如果您对某个具体场景的实现有更多疑问，我很乐意继续探讨。