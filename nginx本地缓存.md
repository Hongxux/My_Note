

### 1. **配置开启共享字典**

在`nginx.conf`的http块中添加：

```
http {
    lua_shared_dict item_cache 100m;  # 开辟100M共享内存区域
    # 可定义多个不同用途的共享字典
    lua_shared_dict api_cache 50m;
    lua_shared_dict session_store 20m;
}
```

### 2. **基本操作示例**

```
-- 获取共享字典对象
local item_cache = ngx.shared.item_cache

-- 存储数据（带过期时间）
local success, err, forcible = item_cache:set("user:123", "user_data", 3600)  -- 过期时间1小时

-- 读取数据
local user_data = item_cache:get("user:123")

-- 删除数据
item_cache:delete("user:123")

-- 原子性递增/递减
local new_val = item_cache:incr("counter", 1, 0)  -- 键,步长,初始值
```

## 💡 高级特性与最佳实践

### **原子操作保证多Worker一致性**

```
-- 安全的数据更新模式
local function update_cache(key, new_value, ttl)
    local dict = ngx.shared.item_cache
    local success, err = dict:safe_set(key, new_value, ttl)
    if not success then
        ngx.log(ngx.ERR, "缓存更新失败: ", err)
    end
    return success
end
```

### **缓存失效策略**

```
-- 带版本控制的缓存管理
local cache_key = "data_v2_" .. entity_id  -- 版本化键名
local data = item_cache:get(cache_key)
if not data then
    -- 缓存未命中，重新加载
    data = load_from_database(entity_id)
    item_cache:set(cache_key, data, 1800)  -- 缓存30分钟
end
```

## 🚀 实际应用场景

### **1. 分布式计数器**

```
-- 实现请求限流
local limit_count = ngx.shared.api_limit
local key = "api_" .. ngx.var.remote_addr
local current = limit_count:get(key) or 0

if current > 100 then  -- 每分钟最大100次请求
    ngx.exit(429)  -- 太多请求
else
    limit_count:incr(key, 1, 1, 60)  -- 60秒后自动过期
end
```

### **2. 会话共享**

```
-- 多Worker间共享用户会话
local session_data = {
    user_id = 123,
    login_time = os.time(),
    permissions = {"read", "write"}
}

local session_id = ngx.md5(tostring(123) .. os.time())
ngx.shared.session_store:set(session_id, cjson.encode(session_data), 7200)  -- 2小时过期
```

## ⚠️ 重要注意事项

1. **内存管理**：共享字典大小固定，需监控使用情况
    
2. **序列化存储**：复杂数据需要JSON序列化后存储
    
3. **性能考量**：适合存储热点数据，不适合大数据量存储
    

这个功能非常适合实现高效的多Worker数据共享和缓存机制！