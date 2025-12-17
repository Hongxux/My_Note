Sring（字符串）是 Redis 中最基本、最常用的数据类型。你可以将其理解为一个键值对（key-value），其中 key 永远是字符串类型，而 value 在 String 类型中，可以是**字符串、整数或浮点数**。最重要的是，Redis 的 String 是**二进制安全**的，这意味着它可以存储任何形式的字节数据，包括文本、JSON、图片或序列化后的对象，其最大容量为 **512MB**。

|特性维度|说明|
|---|---|
|**存储内容**|文本、数字（整数/浮点数）、二进制数据（如图片、序列化对象）|
|**最大容量**|512 MB|
|**关键特性**|二进制安全（可存储任意格式数据）、所有操作均为原子性|
|**经典场景**|缓存、计数器、分布式锁、Session存储|

### 底层设计与高效原理

String 类型的底层实现是 **[[SDS]] (Simple Dynamic String，简单动态字符串)**

为了进一步优化内存，Redis会根据存储的值自动选择三种编码方式：
- int：存储的值是**8字节长整型范围内的整数**
	- 存储特点：直接将数值存储在![[Pasted image 20251127091808.png]]指针位置（指针位置也是八字节）
- embstr：存储的字符串长度**小于等于44字节**（Redis 5.0+）
	- 44字节作为限制的原因：不会产生内存碎片
		- redisObject和存储44字节的字符串SDS结构，加起来共占用64字节
		- Redis底层使用JMolloc的内存分配模式，分配内存的单位为2^ n字节
	- 存储特点：`redisObject`和SDS结构分配在**一块连续内存**![[Pasted image 20251127091124.png]]
	- 存储优势：
		- 内存分配效率高：申请内存时只需要调用一次内存分配函数（避免多次内核态和用户态切换）
		- 缓存局部性高：当程序访问这个对象时，CPU有很大概率能通过一次缓存行（Cache Line）加载就将所有相关数据读取到高速缓存中
	- 存储模式升级：`embstr`编码是只读的。一旦对其进行修改，无论结果是否超过44字节，都会被转换为`raw`编码。原因：
		- 内存重分配：分配一句足够大的内存，本质和raw没有区别
		- 修改开销：长字符串被修改的可能性更大，
- raw：存储的字符串长度**大于44字节**
	- 存储特点：`redisObject`和SDS结构存放在**两块独立内存**![[Pasted image 20251127091941.png]]


- 启发：
	- 合理设置字符串长度：如果可能，尽量使用较短的字符串（≤44字节）或整数，以利用 `embstr`和 `int`编码的高效性
	- 警惕大 Key：虽然 SDS 有扩容策略，但一个巨大的 String Key（接近 512MB）仍然会占用大量连续内存，影响系统性能。需要避免存储过大的单一值
###  核心命令与使用示例

String类型的命令非常丰富，可以分为几个功能类别。

#### 1. 基本值的设置与获取

- **`SET key value`**/ **`GET key`**: 最基础的设置和获取值。
    
- **`MSET key1 value1 key2 value2...`**/ **`MGET key1 key2...`**: 批量设置或获取多个键值对，能有效减少网络往返时间，提升效率。
    
- **`SETNX key value`**: **当且仅当key不存在时**才设置值。这是实现**分布式锁**的基石命令。
    
- **`SETEX key seconds value`**: 设置值的同时，指定其过期时间（秒级）。
    

**示例：基础操作与分布式锁**

```
> SET name "Alice"
OK
> GET name
"Alice"
> MSET age 30 city "Beijing"
OK
> MGET name age city
1) "Alice"
2) "30"
3) "Beijing"
> SETNX my_lock "unique_identifier"  # 尝试获取锁
(integer) 1  # 获取成功
> SETNX my_lock "another_identifier"
(integer) 0  # 获取失败，因为锁已存在
> SETEX user_session:123 3600 "session_data"  # 设置会话，1小时后过期
OK
```

#### 2. 数字操作

String类型可以直接对数字进行**原子性**的增减操作，非常适合计数场景。

- **`INCR key`**/ **`DECR key`**: 将key中存储的数字值增加1/减少1。
    
- **`INCRBY key increment`**/ **`DECRBY key decrement`**: 按指定步长增减。
    
- **`INCRBYFLOAT key increment`**: 增加一个浮点数。
    

**示例：文章点赞计数**

```
> SET article:1001:likes 0
OK
> INCR article:1001:likes  # 用户点赞
(integer) 1
> INCRBY article:1001:likes 5  # 批量点赞（例如分享带来的增长）
(integer) 6
> DECR article:1001:likes  # 用户取消点赞
(integer) 5
```

#### 3. 字符串操作

- **`APPEND key value`**: 向指定key的值后追加字符串。
    
- **`GETRANGE key start end`**: 获取字符串的子串，支持负数索引（-1表示最后一个字符）。
    
- **`SETRANGE key offset value`**: 从指定偏移量开始，覆盖原字符串的一部分。
    
- **`STRLEN key`**: 获取字符串值的长度。
    

**示例：字符串操作**

```
> SET msg "Hello"
OK
> APPEND msg ", World!"
(integer) 13
> GET msg
"Hello, World!"
> GETRANGE msg 0 4
"Hello"
> STRLEN msg
(integer) 13
```

###  主要应用场景

1. **缓存（Cache）**
    
    这是String最典型的用途。将数据库查询结果、复杂计算结果等序列化（如JSON格式）后存入Redis，后续请求可直接读取，极大提升响应速度，降低后端压力。通常会给缓存设置过期时间（使用`SETEX`或`SET key value EX seconds`）。
    
2. **计数器（Counter）**
    
    - 利用`INCR`、`DECR`等命令的**原子性**，可以安全地实现文章阅读量、视频播放次数、用户点赞数、网站访问量等统计，无需担心多线程或多进程环境下的并发问题。
    - 还可以用于解决超卖问题，decr后发现返回值<0则说明超卖了，这次下单失败
    
    
3. **分布式锁（Distributed Lock）**
    
    利用`SETNX`的互斥特性（只有key不存在时才能设置成功），可以实现简单的分布式锁。通常配合过期时间（`PX`参数）使用，防止锁无法释放。
    
    ```
    SET lock:resource001 "unique_identifier" NX PX 10000  # 获取锁，并设置10秒过期
    ```
    
4. **共享Session（Session Store）**
    
    在分布式或集群环境中，将用户登录状态（Session）集中存储在Redis中，可以解决Session一致性问题，实现单点登录。
    