
^ed53aa

1. **Redis模块引入与初始化**
```
 -- 引入Redis模块
local redis = require("resty.redis")

-- 初始化Redis对象
local red = redis:new()

-- 设置超时时间（单位：毫秒）
red:set_timeout(1000, 1000, 1000)  -- 连接、发送、接收超时
```
2. **连接池管理封装**
```
-- 释放连接到连接池（非真正关闭）
local function close_redis(red)
    local pool_max_idle_time = 10000  -- 连接空闲时间（毫秒）
    local pool_size = 100             -- 连接池大小
    local ok, err = red:set_keepalive(pool_max_idle_time, pool_size)
    if not ok then
        ngx.log(ngx.ERR, "放入Redis连接池失败：", err)
    end
end
```
3.  **Redis查询函数封装**
```
local function read_redis(ip, port, key)
    -- 建立连接
    local ok, err = red:connect(ip, port)
    if not ok then
        ngx.log(ngx.ERR, "连接redis失败：", err)
        return nil
    end
    
    -- 执行查询
    local resp, err = red:get(key)
    if not resp then
        ngx.log(ngx.ERR, "查询Redis失败：", err, " key=", key)
        return nil
    end
    
    -- 处理空值情况
    if resp == ngx.null then
        resp = nil
        ngx.log(ngx.ERR, "查询Redis数据为空，key=", key)
    end
    
    -- 释放连接到连接池
    close_redis(red)
    return resp
end 
```

4. 实际应用
```
-- 在OpenResty中使用示例
local function get_user_info(user_id)
    local user_data = read_redis("127.0.0.1", 6379, "user:" .. user_id)
    if not user_data then
        -- 缓存未命中，从数据库查询，去后端tomcat中查询，后端查询本地缓存，本地缓存没有则查询数据库，并且更新缓存（Redis缓存和JVM本地缓存Caffeine）
        user_data = query_database(user_id)
        ·
    end
    return user_data
end
```
## lua脚本和Redis的交互
- 好处：
	- 将多个命令整合到一个脚本中，由 Redis 单线程**原子执行**，中途不会被其他命令打断
		- 事务中的命令只是被排队，在 `EXEC`时统一执行，无法根据中间结果决定后续执行哪些命令。
	- 将多个操作打包成一个脚本，**一次发送、一次返回**，显著减少网络往返次数（RTT）
	- 利用 Lua 语言的控制结构（如条件、循环），可编写**灵活逻辑**，扩展 Redis 功能，实现更复杂的业务模式
- 问题：
	- lua是执行的时候不会被打断，但是出错时不会自动回滚：如果脚本执行到一半报错，之前已执行的命令**不会撤销**
		- 示例：
			```
			-- 错误的期望：三条命令要么全成功，要么全不执行
			redis.call('SET', 'key1', 'value1') -- 执行成功
			local data = some_undefined_variable -- 脚本出错，停止执行
			redis.call('SET', 'key2', 'value2') -- 这行不会执行
			-- 结果：key1 已被设置，但 key2 未设置，数据不一致[9](@ref)。
			```
		- 解决措施：脚本内做好严谨的校验，在写操作前通过读操作进行条件判断，避免无效的部分写操作。
	- Lua脚本会**阻塞**Redis单线程。一个缓慢或存在死循环的脚本会让整个Redis服务暂停，其他所有命令都必须等待
	- 在Redis Cluster模式下，所有在Lua脚本中操作的Key必须位于**同一个哈希槽（hash slot）**，否则会返回 `CROSSSLOT`错误
		- 解决措施：使用 **Hash Tag**，即通过 `{}`让不同的Key落在同一个槽。
	- 数据类型转换的陷阱
		- 表现：
			- 在Lua中，Redis的 `nil`(空结果) 会被转换为 `false`布尔值
			- 即使Redis中存储的是整数，通过 `redis.call('GET', key)`获取的值在Lua中也是**字符串类型**，需要显式调用 `tonumber()`转换才能进行数学运算
			- 通过 `ARGV`传入的参数，在Lua中永远是字符串类型
		- 解决措施：进行严格的类型检查和转换
	- 安全性问题
### lua脚本操作Redis
![[Pasted image 20251119144029.png]]

### 在Redis中调用lua脚本
![[Pasted image 20251119144331.png]]
![[Pasted image 20251119144725.png]]
