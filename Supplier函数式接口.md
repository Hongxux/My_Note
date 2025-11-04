---
aliases:
  - Supplier
---
### Supplier函数式接口详解

#### 1. 一句话总结

Supplier是一个无参数但返回值的函数式接口，专门用于实现**惰性求值**​（lazy evaluation），避免不必要的计算开销。

#### ① 定义与关系分析

​**定义**​：`Supplier<T>`是`java.util.function`包中的函数式接口，包含抽象方法`T get()`，无参数输入，返回类型为T的值。

​**关系分析**​：

- ​**解决的问题**​：解决了传统编程中**立即求值**导致的资源浪费问题（A问题）
    
- ​**副作用与解决**​：带来了执行时机不确定的副作用，通过明确的`get()`方法调用时机控制来解决（C解决）
    
- ​**替代/增强**​：是对传统直接调用的延迟化增强，支持按需计算
    
- ​**易混淆概念**​：易与`Callable<T>`混淆，但`Supplier`不抛出受检异常，语义更简洁
    

​**定位**​：属于函数式编程中的**生产者接口**，建立在延迟计算理论基础上，是资源优化和性能调优的重要工具。

​**设计理念与权衡**​：

- ​**设计理念**​：遵循"按需计算"原则，体现资源最小化消耗思想
    
- ​**优点**​：减少不必要的对象创建和计算，提高性能
    
- ​**缺点**​：增加了代码复杂度，可能隐藏bug
    
- ​**权衡原因**​：在性能敏感场景下，用复杂度换取执行效率的显著提升
    

#### 2. 经典使用情景

​**场景描述**​：如图片所示的`Objects.requireNonNullElseGet()`场景，当默认值构造开销较大且很少使用时。

​**触发条件**​：

- 对象创建或计算成本较高
    
- 该值可能不会被使用（如图片中的"rarely null"情况）
    
- 需要支持动态生成或随机值
    

​**关键特征**​：

- 无参数，纯生产者角色
    
- 执行延迟到真正需要时
    
- 支持复用和缓存优化
    

#### 3. 工作原理与实现

​**工作原理**​：

```
@FunctionalInterface
public interface Supplier<T> {
    T get(); // 核心方法：延迟执行并返回值
}
```

​**图片示例分析**​：

```
// 立即求值：无论day是否为null都会创建LocalDate
LocalDate hireDay = Objects.requireNonNullElse(day, 
    new LocalDate.of(1970, 1, 1));

// 惰性求值：仅当day为null时才创建LocalDate  
LocalDate hireDay = Objects.requireNonNullElseGet(day,
    () -> new LocalDate.of(1970, 1, 1)); // Supplier延迟计算
```

​**潜在问题与解决措施**​：

|问题类型|具体问题|解决措施|
|---|---|---|
|​**多次调用不一致**​|同一Supplier多次get()返回不同值|明确文档说明或使用缓存|
|​**空值问题**​|`get()`返回null导致NPE|使用`Optional.ofNullable()`包装|
|​**异常处理**​|`get()`中抛出运行时异常|添加try-catch或使用安全的Supplier包装|

​**实现模式对比**​：

```
// 问题模式：可能重复创建昂贵对象
public HeavyObject getHeavyInstance() {
    if (instance == null) {
        instance = createExpensiveObject(); // 昂贵操作
    }
    return instance;
}

// 解决模式：使用Supplier封装
private Supplier<HeavyObject> heavySupplier = () -> createExpensiveObject();

public HeavyObject getHeavyInstance() {
    return heavySupplier.get(); // 按需创建
}
```

#### 4. 面试官关心的问题与答案

​**问题1：Supplier与普通方法调用有什么区别？​**​

​**答案**​：

- 执行时机：普通方法立即执行，Supplier延迟到`get()`调用时执行
    
- 语义差异：Supplier明确表示"生产者"角色，强调延迟计算特性
    
- 复用性：Supplier可以作为参数传递，支持策略模式
    

​**问题2：图片中的例子为什么使用Supplier更优？​**​

​**答案**​：

```
// 传统方式：总是创建LocalDate，即使day不为null
LocalDate.of(1970, 1, 1); // 立即执行

// Supplier方式：仅在需要时创建
() -> new LocalDate.of(1970, 1, 1); // 延迟执行
```

优势在于避免99%情况下的不必要对象创建，符合"rarely null"的业务假设。

​**问题3：Supplier在什么场景下不适合使用？​**​

​**答案**​：

- 计算极其简单且开销可忽略
    
- 需要立即得到结果的场景
    
- 方法有副作用且需要控制执行时机
    
- 对执行性能要求不高的工具类方法
    

​**问题4：如何处理Supplier中的异常？​**​

​**答案**​：

```
// 方式1：包装为运行时异常
Supplier<String> safeSupplier = () -> {
    try {
        return riskyOperation();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
};

// 方式2：返回Optional
Supplier<Optional<String>> safeSupplier = () -> {
    try {
        return Optional.of(riskyOperation());
    } catch (Exception e) {
        return Optional.empty();
    }
};
```

​**问题5：Supplier如何与缓存结合使用？​**​

​**答案**​：

```
// 带缓存的Supplier实现
public class MemoizingSupplier<T> implements Supplier<T> {
    private final Supplier<T> delegate;
    private volatile T value;
    
    public MemoizingSupplier(Supplier<T> delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public T get() {
        if (value == null) {
            synchronized (this) {
                if (value == null) {
                    value = delegate.get();
                }
            }
        }
        return value;
    }
}

// 使用示例
Supplier<ExpensiveObject> cachedSupplier = 
    new MemoizingSupplier<>(() -> createExpensiveObject());
```

​**问题6：Supplier在测试中如何应用？​**​

​**答案**​：

```
// 测试时可以注入不同的Supplier
@Test
void testWithMockSupplier() {
    Supplier<String> mockSupplier = () -> "mock-data";
    Service service = new Service(mockSupplier);
    
    String result = service.process();
    assertEquals("processed-mock-data", result);
}

// 时间相关的测试
@Test 
void testTimeSensitiveLogic() {
    Supplier<Instant> timeSupplier = () -> Instant.parse("2024-01-01T00:00:00Z");
    Scheduler scheduler = new Scheduler(timeSupplier);
    
    assertTrue(scheduler.shouldExecute());
}
```

Supplier的设计体现了**计算与消费分离**的重要原则，通过将"如何生成"与"何时使用"解耦，显著提升了代码的灵活性和性能表现。