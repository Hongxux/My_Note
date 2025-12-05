
### 1. **基础内存使用指标**

|参数|说明|正常范围参考|
|---|---|---|
|`used_memory`|Redis 分配器分配的总内存（字节）|根据数据量而定|
|`used_memory_human`|人类可读格式的总内存使用量|如 "1.5G"|
|`used_memory_rss`|系统角度进程占用的物理内存（常驻集大小）|通常略大于 used_memory|
|`used_memory_rss_human`|人类可读的物理内存使用量|如 "1.6G"|
|`used_memory_peak`|Redis 运行以来的内存使用峰值||
|`used_memory_peak_human`|人类可读的内存使用峰值||
|`used_memory_peak_perc`|当前内存使用占峰值的百分比|`used_memory/used_memory_peak`|
|`used_memory_overhead`|服务器运行所需的内务开销||
|`used_memory_startup`|Redis 启动时初始化占用的内存||
|`used_memory_dataset`|实际数据集占用的内存|`used_memory - used_memory_overhead`|

### 2. **内存碎片指标**

|参数|说明|健康指标|
|---|---|---|
|`mem_fragmentation_ratio`|**内存碎片率**（RSS/used_memory）|**1.0-1.5**​ 为健康|
|`mem_fragmentation_bytes`|碎片化的内存字节数|越小越好|
|`allocator_frag_ratio`|分配器级别的碎片率||
|`allocator_frag_bytes`|分配器级别的碎片字节数||
|`allocator_rss_ratio`|分配器RSS比率||
|`allocator_rss_bytes`|分配器RSS字节数||

### 3. **详细内存分配 breakdown**

|参数|说明|
|---|---|
|`used_memory_lua`|Lua引擎使用的内存|
|`used_memory_vm_eval`|Redis函数使用的内存|
|`used_memory_scripts`|脚本缓存使用的内存|
|`used_memory_scripts_human`|人类可读的脚本内存|
|`number_of_cached_scripts`|缓存的脚本数量|
|`used_memory_vm_functions`|Redis函数内存使用|
|`number_of_functions`|定义的函数数量|
|`number_of_libraries`|加载的函数库数量|

### 4. **分配器统计信息**

|参数|说明|
|---|---|
|`allocator_active`|分配器活跃内存|
|`allocator_active_bytes`|活跃内存字节数|
|`allocator_allocated`|分配器已分配内存|
|`allocator_allocated_bytes`|已分配内存字节数|
|`allocator_resident`|分配器常驻内存|
|`allocator_resident_bytes`|常驻内存字节数|

### 5. **数据集相关统计**

|参数|说明|
|---|---|
|`used_memory_dataset_perc`|数据集内存占总内存百分比|
|`cluster_links`|集群链接内存使用（集群模式）|
|`dbCount`|数据库数量|
|`keyCount`|总键数量|
|`expires`|有过期时间的键数量|

