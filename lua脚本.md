


## lua脚本的基础
### lua的数据结构
![[2b0e4c6accedfcb994c75742a4399d93.jpeg]]
### 循环遍历
![[4323d171ecc4799287e45a6e0984efc8.jpeg]]
## lua脚本与nginx交互
### 使用lua脚本前提
1. 使用OpenResty
2. 修改OpenResty的nginx配置文件 ，引入依赖和加入监听 ![[Pasted image 20251125105046.png]]
3. 在/usr/local/openresty/nginx文件夹下创建lua文件，在这个文件夹中编写自己的lua文件
4. 编写完成后`sudo systemctl reload openresty`
### 写数据到Response
![[Pasted image 20251125092608.png]]

### 获取请求参数
![[Pasted image 20251125113848.png]]
### 发送http请求
![[ace830ba01e861cf660f4c689a5bbb1b.jpeg]]

- 


![[91efd3eb88f3314011f734a716bd32ef.jpeg]]

```lua
-- common.lua 封装的函数 
local _M = {} -- 创建一个空的模块表  
  
-- 正确：将函数定义并赋值到模块表_M中  
function _M.read_http_get(path, params)  
    local resp = ngx.location.capture(path, {  
        method = ngx.HTTP_GET,  
        args = params,  
    })  
    if not resp then  
        ngx.log(ngx.ERR, "http not found path:", path, ", args:", args)  
        ngx.exit(404)  
    end  
    return resp.body  
end  
  
-- 重要：最后返回这个模块表  
return _M
```
```lua
--/usr/local/openresty/lualib/ngx/lua/get_shop_by_id.lua 调用封装的函数
local id = ngx.var[1]  
local common = require('common')  
  
local cjson = require('cjson')  
ngx.log(ngx.ERR, "Received request for shop ID: ", id)  
local shopJSON = common.read_http_get("/shop/"..id, nil)  
ngx.log(ngx.ERR, "Response from backend: ", shopJSON)  
ngx.say(shopJSON)
```
### JSON序列化和反序列化
![[Pasted image 20251125093014.png]]


### 操作Redis

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

### lua脚本使用注意
- 拼接字符串使用`..`![[Pasted image 20251120144801.png]]
- redis的get得到的是字符串，要转化成数字![[Pasted image 20251120144825.png]]
- 给lua脚本传入值的时候要对值进行tostring()
- 使用Redis是redis.call而不是Redis.call
编写和使用 Lua 脚本，尤其是在 Redis 这样的环境中，确实需要留意一些常见的“坑”。为了帮助你快速建立整体认知，下表汇总了核心问题领域、关键陷阱及其后果和应对策略。

|问题领域|关键陷阱|潜在后果|核心应对策略|
|---|---|---|---|
|**语言特性**​|变量作用域混淆、`nil`/`false`逻辑、Table 比较与长度|逻辑错误、意外行为|使用 `local`声明变量，理解真假值，使用 `next()`判空，谨慎使用 `#`|
|**Redis 环境**​|集群 key 路由、脚本原子性、参数传递错误|执行失败、性能瓶颈、数据不一致|确保多个 Key 属于同一哈希槽，避免长时操作，使用 `KEYS`/`ARGV`传参|
|**错误与调试**​|缺乏编译期检查、错误信息模糊|调试困难|善用 `pcall`/`xpcall`保护调用，使用 `assert`进行参数校验|
|**性能与维护**​|脚本过于复杂、内存使用不当|阻塞 Redis、内存溢出|保持脚本轻量，避免在脚本内处理大量数据，管理好脚本缓存|

下面我们详细探讨这些关键点，并说明如何规避。

#### 💡 Lua 语言特性与常见陷阱

1. **变量作用域与局部变量**
    
    在 Lua 中，变量默认是全局的。忘记使用 `local`关键字声明变量，会意外创建全局变量，可能引发难以调试的冲突和错误。**务必在声明变量时使用 `local`**​ 。
    
    ```
    -- 错误示例：创建了全局变量 globalVar
    function myFunction()
        globalVar = 10
    end
    
    -- 正确示例：使用局部变量
    function myFunction()
        local localVar = 10
    end
    ```
    
2. **条件判断中的“真假”值**
    
    Lua 中只有 **`nil`**​ 和 **`false`**​ 被视为假（false），**数字 0、空字符串 `''`都被视为真**（true）。这和其他一些语言不同，需要特别注意 。
    
    ```
    if 0 then
      print("This will be printed because 0 is true in Lua!")
    end
    ```
    
3. **Table 的比较与长度操作符 `#`**
    
    - **比较**：在 Lua 中，`{} == {}`的结果是 `false`，因为比较的是两个 table 的内存地址（引用），而不是内容。要比较内容，需要遍历字段 。
        
    - **长度 `#`**：`#`操作符返回的是 **数组部分**​ 从索引 1 开始连续正整数元素的最大索引。如果数组中有 `nil`空洞（例如 `{1, nil, 3}`），`#`的结果将不可预测。判断 table 是否为空，更安全的方式是使用 `next(t) == nil`。
        
    
4. **字符串与数字的自动转换**
    
    Lua 会在字符串和数字之间自动转换，但有时需要显式转换以避免意外。使用 `tonumber()`和 `tostring()`进行显式转换是良好的实践 。
    

#### 🔧 Redis 环境中 Lua 脚本的特殊考量

1. **Redis 集群与 Key 分布**
    
    在 Redis 集群模式下，一个 Lua 脚本中操作的所有 **Key 必须位于同一个哈希槽 (hash slot)**​ 中，否则脚本会执行失败。可以通过使用 **哈希标签 (hashtag)**​ 来确保多个 Key 被路由到同一个槽，例如 `KEYS[1] = "order:{123}"`, `KEYS[2] = "stock:{123}"`。
    
2. **脚本的原子性与阻塞**
    
    Lua 脚本在 Redis 中是**原子性**执行的，期间不会执行其他命令。因此，**必须避免在脚本中执行耗时过长的操作**（如循环遍历大量数据），否则会长时间阻塞 Redis，影响其他请求。如果脚本执行超时（默认 5 秒），可能会收到 `BUSY`错误 。
    
3. **参数传递：`KEYS`与 `ARGV`**
    
    向脚本传递参数时，**必须使用 `KEYS`数组传递所有键，使用 `ARGV`数组传递其他参数**。这是 Redis 的规范，尤其是在集群环境下至关重要。要确保传递的 `KEYS`数量与 `numkeys`参数一致 。
    
    ```
    -- 正确做法
    local value = redis.call('get', KEYS[1])
    local increment = tonumber(ARGV[1])
    ```
    
4. **脚本缓存与 `EVALSHA`**
    
    使用 `SCRIPT LOAD`预加载脚本会返回一个 SHA1 校验和，之后可以使用 `EVALSHA`通过这个校验和执行脚本，减少网络传输。但需要注意，Redis 不保证脚本缓存的持久化（如重启后丢失）。因此，客户端代码需要能处理 `NOSCRIPT`错误，并回退到使用 `EVAL`命令 。
    

#### 🛡️ 错误处理与调试

1. **使用保护式调用**
    
    使用 `pcall`(protected call) 或 `xpcall`来调用可能出错的函数，可以捕获异常而非导致整个脚本中止 。
    
    ```
    local success, result = pcall(function()
        return redis.call('some_risky_command')
    end)
    if not success then
        -- 处理错误，result 为错误信息
        return nil
    end
    ```
    
2. **使用 `assert`进行参数校验**
    
    在脚本开始处使用 `assert`校验参数或前置条件，可以在问题发生时快速定位 。
    
    ```
    local value = assert(tonumber(ARGV[1]), "Increment must be a number")
    ```
    

