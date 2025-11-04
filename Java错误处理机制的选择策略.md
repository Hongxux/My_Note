

## 一、三种机制的定位与选择困境

### 核心区别矩阵

| 机制                   | 目标场景     | 错误性质    | 生产环境行为 | 适用阶段  |
| -------------------- | -------- | ------- | ------ | ----- |
| ​**断言(Assert)​**​    | 内部逻辑错误检测 | 致命、不可恢复 | 默认禁用   | 开发/测试 |
| ​**异常(Exception)​**​ | 外部条件错误处理 | 可捕获恢复   | 始终生效   | 全生命周期 |
| ​**日志(Logging)​**​   | 状态记录与监控  | 信息性记录   | 按级别输出  | 全生命周期 |

### 选择决策树

![[Pasted image 20251022212055.png]]

​**断言的核心定位**​：

- ✅ ​**适用场景**​：定位测试期的内部程序错误
    
- ❌ ​**不适用**​：生产环境错误处理、方法契约约定的错误条件
    

> 📌 形象比喻：断言像"近海救生衣"——开发阶段穿戴，生产环境"抛入大海"

## 二、排序方法案例的契约分析

### 方法契约明确时的处理

```
/**
 * @throws IllegalArgumentException if fromIndex > toIndex
 * @throws ArrayIndexOutOfBoundsException if fromIndex < 0 or toIndex > a.length
 */
static void sort(int[] a, int fromIndex, int toIndex) {
    // 契约已定义 → 必须使用异常
    if (fromIndex > toIndex) 
        throw new IllegalArgumentException("fromIndex > toIndex");
    if (fromIndex < 0 || toIndex > a.length)
        throw new ArrayIndexOutOfBoundsException("索引越界");
}
```

​**为什么不能用断言？​**​

1. ​**契约约束**​：文档承诺特定异常行为，调用方依赖此契约
    
2. ​**生产环境需求**​：参数验证必须在运行时持续生效
    
3. ​**调用方权利**​：用户有权得知参数错误的具体原因
    

### 契约未定义时的谨慎选择

```
// 原始契约未提及null处理 → 不应急于添加断言
static void sort(int[] a, int fromIndex, int toIndex) {
    // ❌ 危险做法：单方面加强契约
    // assert a != null; // 调用方可能依赖当前行为
    
    // ✅ 保守做法：维持现有行为或通过文档明确
    if (a == null) {
        // 根据现有行为决定：静默返回? 抛出NPE?
    }
}
```

​**设计原则**​：不破坏向后兼容性，契约变更需通过文档明确通知调用方。

## 三、契约演变与断言引入条件

### 契约明确化后的断言使用

```
/**
 * @param a 待排序数组（必须非空）
 */
static void sort(int[] a, int fromIndex, int toIndex) {
    // 契约明确后 → 可使用断言
    assert a != null : "数组不能为null"; // 开发期检查
    
    // 契约定义的仍用异常
    if (fromIndex > toIndex) 
        throw new IllegalArgumentException("fromIndex > toIndex");
}
```

​**断言引入的触发条件**​：

1. ​**契约明确化**​：文档清晰定义参数要求
    
2. ​**纯内部错误**​：违反条件表明**调用方bug而非环境问题**
    
3. ​**性能敏感**​：检查开销大且生产环境可假设满足
    

### 渐进式契约强化策略

```
// 阶段1：宽松契约（无null检查）
public void process(Object data) {
    // 现有逻辑
}

// 阶段2：文档明确但不断言
/**
 * @param data 输入数据（必须非null）
 */
public void process(Object data) {
    if (data == null) {
        log.warn("收到空参数，可能调用方错误");
        return; // 保持兼容
    }
}

// 阶段3：启用断言（下一个主要版本）
public void process(Object data) {
    assert data != null : "违反契约: data不能为null";
    // 正式要求调用方遵守
}
```

## 四、前置条件(Precondition)概念解析

### 前置条件的定义与特性

​**前置条件**​：方法执行前必须满足的条件集合，是方法与调用方之间的契约基础。

```
/**
 * 计算平方根
 * @param x 输入值（前置条件: x ≥ 0）
 * @return 平方根结果
 * @throws IllegalArgumentException 如果x < 0（前置条件违反）
 */
public double sqrt(double x) {
    // 前置条件检查
    if (x < 0) throw new IllegalArgumentException("x不能为负数");
    return Math.sqrt(x);
}
```

### 违反前置条件的行为谱系

|违反程度|典型响应|适用场景|
|---|---|---|
|​**轻微违反**​（可恢复）|返回默认值/空值|用户输入验证|
|​**中度违反**​（应告知）|抛出受检异常|业务规则校验|
|​**严重违反**​（程序错误）|抛出非受检异常|内部逻辑检查|
|​**致命违反**​（不可继续）|断言失败|开发期契约验证|

### 前置条件与断言的结合模式

```
public class DatabaseService {
    /**
     * 执行查询
     * @param sql 查询语句（前置条件: 非空且有效）
     * @param params 参数（前置条件: 与占位符数量匹配）
     */
    public ResultSet executeQuery(String sql, Object[] params) {
        // 开发期：严格断言检查
        assert sql != null && !sql.trim().isEmpty() : "SQL不能为空";
        assert params != null : "参数数组不能为null";
        assert params.length == countPlaceholders(sql) : "参数数量不匹配";
        
        // 生产期：基础异常保障
        if (sql == null || sql.trim().isEmpty()) {
            throw new IllegalArgumentException("SQL语句无效");
        }
        
        return doExecute(sql, params);
    }
}
```

## 五、实战决策框架

### 四象限决策模型

```
public class ErrorHandlingDecision {
    
    /**
     * 错误处理策略选择器
     * @param isInternalBug 是否程序内部错误（true=断言）
     * @param isInContract 是否方法契约定义（true=异常）
     * @param needsRecovery 是否需要恢复（true=受检异常）
     * @param isProduction 是否生产环境（true=禁用断言）
     */
    public void handleError(boolean isInternalBug, boolean isInContract, 
                          boolean needsRecovery, boolean isProduction) {
        
        if (isInternalBug && !isProduction) {
            // 内部错误+开发环境 → 断言
            assert condition : "内部逻辑错误";
        } else if (isInContract) {
            // 契约定义的行为 → 异常
            if (needsRecovery) {
                throw new CheckedException("可恢复错误");
            } else {
                throw new UncheckedException("程序错误");
            }
        } else {
            // 其他情况 → 日志记录
            logger.warn("非致命问题，继续执行");
        }
    }
}
```

### 黄金法则总结

1. ​**断言用于"永不应发生"的条件**​（内部一致性检查）
    
2. ​**异常用于方法契约约定的错误**​（参数验证、业务规则）
    
3. ​**前置条件变更需要通信和版本管理**​
    
4. ​**生产环境安全优于开发期便利**​
    

> 最终目标：通过恰当的错误处理机制，构建**可靠、可维护、高性能**的Java应用程序。