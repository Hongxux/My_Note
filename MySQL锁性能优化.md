
## 一、SQL和索引层面优化（效果最直接）

### 1. ​**索引优化 - 核心中的核心**​

```
-- ❌ 坏例子：没有合适索引，导致全表扫描+锁全表
UPDATE users SET status = 'inactive' WHERE create_time < '2023-01-01';

-- ✅ 优化：为查询条件添加索引
CREATE INDEX idx_create_time ON users(create_time);
-- 现在只锁定满足条件的行，而不是全表
```

​**索引设计原则**​：

- ​**覆盖索引**​：让查询只需访问索引，避免回表
    

```
-- 覆盖索引示例
CREATE INDEX idx_age_name ON users(age, name);
SELECT id, name FROM users WHERE age = 20; -- 只需扫描索引
```

- ​**唯一索引优先**​：等值查询时，唯一索引直接退化为行锁，锁范围最小
    

```
-- 手机号设为唯一索引，锁范围最小化
ALTER TABLE users ADD UNIQUE INDEX idx_phone(phone);
```

### 2. ​**SQL写法优化**​

```
-- ❌ 锁范围过大
UPDATE orders SET status = 'shipped' WHERE user_id = 100;

-- ✅ 精确锁定，使用主键或唯一索引
UPDATE orders SET status = 'shipped' WHERE id IN (
    SELECT id FROM orders WHERE user_id = 100 AND status = 'pending'
);

-- ❌ 模糊查询锁范围大
SELECT * FROM products WHERE name LIKE '%apple%' FOR UPDATE;

-- ✅ 使用更精确的查询条件
SELECT * FROM products WHERE name = 'apple' FOR UPDATE;
```

## 二、事务设计优化（减少锁持有时间）

### 1. ​**短事务原则**​

```
// ❌ 坏例子：长事务，锁持有时间过长
@Transactional
public void processOrder(Long orderId) {
    // 1. 查询订单（开始持有锁）
    Order order = orderDao.selectForUpdate(orderId);
    
    // 2. 复杂的业务逻辑（锁一直被持有）
    validateOrder(order);
    callExternalService(order);  // 外部调用，耗时！
    calculateTax(order);
    
    // 3. 更新订单（几分钟后才释放锁）
    orderDao.update(order);
}

// ✅ 优化：拆分事务，锁持有时间最小化
public void processOrder(Long orderId) {
    // 第一阶段：快速锁定+验证
    Order order = quickValidateAndLock(orderId);
    
    // 第二阶段：无锁处理复杂逻辑
    processBusinessLogic(order);
    
    // 第三阶段：快速更新
    updateOrderStatus(orderId);
}

@Transactional
public Order quickValidateAndLock(Long orderId) {
    Order order = orderDao.selectForUpdate(orderId);
    validateOrder(order);  // 快速验证
    return order;  // 立即提交事务，释放锁
}

// 无事务，不持有锁
public void processBusinessLogic(Order order) {
    callExternalService(order);  // 耗时操作
    calculateTax(order);
}

@Transactional
public void updateOrderStatus(Long orderId) {
    orderDao.updateStatus(orderId, "processed");
}
```

### 2. ​**事务隔离级别优化**​

```
-- 默认REPEATABLE READ防止幻读但锁范围大
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 对于可接受幻读的场景，使用READ COMMITTED
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- 注意：需要确保业务能接受幻读风险

-- 特定查询使用更低的隔离级别
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM large_table;  -- 统计查询，不需要精确
```

## 三、架构层面优化

### 1. ​**读写分离**​

```
// 使用不同的数据源
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource masterDataSource() {
        // 主库：写操作
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource slaveDataSource() {
        // 从库：读操作
        return DataSourceBuilder.create().build();
    }
}

@Service
public class OrderService {
    
    // 读操作走从库，完全不竞争主库的行锁
    @ReadOnly
    public Order getOrder(Long orderId) {
        return orderDao.selectById(orderId);
    }
    
    // 写操作走主库
    public void updateOrder(Order order) {
        orderDao.update(order);
    }
}
```

### 2. ​**批量操作优化**​

```
-- ❌ 低效的批量更新
START TRANSACTION;
UPDATE large_table SET status = 'new' WHERE id = 1;
UPDATE large_table SET status = 'new' WHERE id = 2;
-- ... 1000次更新
COMMIT;  -- 锁持有时间很长

-- ✅ 优化1：单条SQL批量更新
UPDATE large_table SET status = 'new' 
WHERE id IN (1, 2, 3, ..., 1000);  -- 一次锁定，快速完成

-- ✅ 优化2：分批处理，每批小事务
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void batchUpdate(List<Long> ids) {
    int batchSize = 100;
    for (int i = 0; i < ids.size(); i += batchSize) {
        List<Long> batch = ids.subList(i, Math.min(i + batchSize, ids.size()));
        updateBatch(batch);  // 每个批次独立事务
    }
}
```

## 四、并发控制策略

### 1. ​**乐观锁替代悲观锁**​

```
-- 悲观锁（行锁）
SELECT * FROM products WHERE id = 1 FOR UPDATE;
UPDATE products SET stock = stock - 1 WHERE id = 1;

-- 乐观锁（无行锁，通过版本号控制）
ALTER TABLE products ADD version INT DEFAULT 0;

-- 业务逻辑
UPDATE products 
SET stock = stock - 1, version = version + 1 
WHERE id = 1 AND version = @current_version;
-- 如果受影响行数为0，说明版本冲突，重试或报错
```

```
// Java中的乐观锁实现
@Transactional
public boolean updateWithOptimisticLock(Long productId, int quantity) {
    Product product = productDao.selectById(productId);
    int oldVersion = product.getVersion();
    
    int updated = productDao.updateStock(
        productId, quantity, oldVersion, oldVersion + 1
    );
    
    if (updated == 0) {
        // 版本冲突，重试或抛出异常
        throw new OptimisticLockException("数据已被修改，请重试");
    }
    return true;
}
```

### 2. ​**队列化处理（削峰填谷）​**​

```
// 使用队列避免高并发锁竞争
@Service
public class OrderService {
    
    @Autowired
    private RedisTemplate redisTemplate;
    
    public void createOrder(OrderRequest request) {
        // 1. 快速校验（无锁）
        quickValidate(request);
        
        // 2. 放入队列，异步处理
        redisTemplate.opsForList().rightPush("order_queue", request);
        
        // 3. 立即返回，告诉用户订单处理中
        return SubmitResult.success("订单提交成功，处理中...");
    }
    
    @Async
    public void processOrderQueue() {
        while (true) {
            // 从队列顺序取出处理，避免并发锁竞争
            OrderRequest request = redisTemplate.opsForList()
                .leftPop("order_queue", 10, TimeUnit.SECONDS);
            if (request != null) {
                processSingleOrder(request);  // 单个订单处理
            }
        }
    }
}
```

## 五、监控和诊断工具

### 1. ​**实时锁监控**​

```
-- 查看当前锁信息
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 性能Schema监控
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- 查看长时间运行的事务（可能是锁问题的根源）
SELECT * FROM information_schema.INNODB_TRX 
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;  -- 超过60秒的事务
```

### 2. ​**慢查询日志分析**​

```
-- 开启慢查询日志
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;  -- 超过1秒的查询

-- 分析可能造成锁竞争的慢查询
mysqldumpslow -s t /var/log/mysql/slow.log
```

## 六、最佳实践总结

|优化方向|具体措施|效果评估|
|---|---|---|
|​**索引优化**​|覆盖索引、唯一索引、避免全表扫描|⭐⭐⭐⭐⭐（效果最明显）|
|​**事务优化**​|短事务、合理隔离级别、避免长事务|⭐⭐⭐⭐⭐|
|​**架构优化**​|读写分离、批量操作、队列化处理|⭐⭐⭐⭐|
|​**并发策略**​|乐观锁、重试机制、限流降级|⭐⭐⭐|
|​**监控预警**​|锁监控、慢查询分析、性能预警|⭐⭐⭐|

​**核心原则**​：​**用最短的时间、最小的粒度、处理最快的业务**。

记住这个优化优先级：​**索引优化 > 事务设计 > 架构调整 > 并发控制**。通常80%的性能问题通过前两项优化就能解决。

需要我针对某个特定场景进行更详细的展开吗？