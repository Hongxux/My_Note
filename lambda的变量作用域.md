​

​**一、核心知识点**​

1. ​**闭包机制**​：
    
    - Lambda是闭包：包含代码块 + 捕获的自由变量值
        
    - 捕获时机：Lambda创建时复制变量值（非引用）
        
    - 存储方式：编译器生成匿名类存储捕获值
        
    
2. ​**有效final规则**​：
    
| ​**变量类型**​ | 是否可捕获           | 示例                               |
| ---------- | --------------- | -------------------------------- |
| 基本类型局部变量   | 仅当final或等效final | `int x=1;`✅ `x++;`❌              |
| 对象引用       | 引用不可变，对象状态可修改   | `list.add()`✅ `list=new List()`❌ |
| 方法参数       | 同局部变量规则         | `void m(final String s)`✅        |
| 类字段        | 无限制             | `this.field`✅                    |
|            |                 |                                  |
    
3. ​**作用域冲突**​：
    
    - 禁止遮蔽：Lambda参数/变量不能与外部局部变量同名
        
    - `this`指向：始终指向创建Lambda的类实例，而非Lambda本身
        
    

​**二、重点内容**​

- ​**捕获本质**​：值捕获（非引用捕获），与JavaScript闭包不同
    
- ​**并发安全**​：有效final规则防止多线程竞争条件
    
- ​**错误模式**​：
    
    ```
    // 典型错误1：修改捕获变量
    int count = 0;
    button.addActionListener(e -> count++); // 编译错误
    
    // 典型错误2：捕获变化变量
    for (int i=0; i<5; i++) {
        executor.submit(() -> System.out.println(i)); // 编译错误
    }
    ```
    

---

### 第三部分：面试官关心的问题及答案

​**问题1：什么是Lambda的自由变量？如何捕获？​**​

​**答案**​：

自由变量是Lambda表达式外部的非参数变量。捕获机制：

1. 编译器检测Lambda使用的自由变量
    
2. 检查是否符合有效final条件
    
3. 将变量值复制到Lambda生成的匿名类实例中
    
4. 运行时使用副本值而非原变量
    

​**问题2：为什么要有"有效final"限制？​**​

​**答案**​：

主要出于**并发安全**考虑：

- 避免多个Lambda实例竞争修改同一变量
    
- 防止外部方法修改导致Lambda行为不一致
    
- 确保捕获的值在Lambda延迟执行时仍有效
    
    深层原因：Java采用值捕获而非引用捕获的设计选择
    

​**问题3：Lambda中如何修改"捕获"的变量？​**​

​**答案**​：

三种合法方案：

1. ​**使用原子类**​：
    
    ```
    AtomicInteger count = new AtomicInteger(0);
    button.addActionListener(e -> count.incrementAndGet());
    ```
    
2. ​**封装在对象中**​：
    
    ```
    class Counter { int value; }
    Counter counter = new Counter();
    button.addActionListener(e -> counter.value++);
    ```
    
3. ​**使用数组**​：
    
    ```
    int[] count = {0};
    button.addActionListener(e -> count[0]++);
    ```
    

​**问题4：Lambda的`this`和匿名内部类的`this`有何区别？​**​

​**答案**​：

|​**特性**​|Lambda表达式|匿名内部类|
|---|---|---|
|`this`指向|外围类实例|内部类实例自身|
|作用域|与方法作用域相同|独立类作用域|
|内存开销|通常较小（JVM优化）|标准对象开销|

示例：

```
class MyClass {
    void method() {
        // Lambda: this指向MyClass实例
        Runnable r1 = () -> System.out.println(this); 
        
        // 匿名类: this指向Runnable实例
        Runnable r2 = new Runnable() {
            public void run() {
                System.out.println(this); // 不是MyClass.this
            }
        };
    }
}
```

​**问题5：如何解决捕获变化变量的需求？​**​

​**答案**​：

使用局部副本：

```
for (int i=0; i<5; i++) {
    int finalI = i; // 创建有效final副本
    executor.submit(() -> System.out.println(finalI)); // 合法
}
```

原理：每个循环迭代创建新的`finalI`变量，各自被不同Lambda捕获。