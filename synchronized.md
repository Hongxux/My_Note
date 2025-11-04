
## 一句话总结

​**synchronized是JVM层面的互斥同步原语，通过对象监视器(Monitor)和锁升级机制，在保证内存可见性、原子性、有序性的同时进行性能优化。​**​

---

## ① 技术定义

​**synchronized**是Java的关键字，用于实现基于**对象监视器(Monitor)​**​ 的互斥同步，通过JVM内置支持提供线程安全的临界区保护。

​**三个核心语义保证**​：

- ​**原子性**​：临界区代码不可分割执行
    
- ​**可见性**​：遵循happens-before原则，解锁前修改刷新到主内存
    
- ​**有序性**​：防止临界区内指令重排序（as-if-serial语义）
    

---

## ② 技术关系网络

### 问题解决链

```
线程安全问题(竞态条件+内存可见性) 
    → synchronized提供互斥访问  
    → 早期性能差(直接重量级锁)  
    → 引入锁升级优化(偏向/轻量/重量级)
```

### 与ReentrantLock的对比关系

|维度|synchronized|ReentrantLock|
|---|---|---|
|​**实现层面**​|JVM内置原语|JDK API实现|
|​**锁获取**​|悲观阻塞，无法中断|支持tryLock、可中断|
|​**公平性**​|非公平|可配置公平/非公平|
|​**性能趋势**​|低竞争时优化更好|高竞争时更可控|

### 易混淆概念辨析

- ​**对象锁**​ vs ​**类锁**​：实例同步方法锁this，静态同步方法锁Class对象
    
- ​**方法同步**​ vs ​**代码块同步**​：字节码实现方式不同(ACC_SYNCHRONIZED vs monitorenter/monitorexit)
    
- ​**可重入性**​：同一线程可重复获取已持有的锁（通过计数器实现）
    

---

## ③ 技术定位

​**技术栈位置**​：JVM → 内存模型(JMM) → 线程同步 → 互斥锁实现

​**底层依赖**​：

- 对象头Mark Word中的锁标志位
    
- 操作系统的互斥量(mutex)和条件变量
    
- JVM的监视器(Monitor)实现机制
    

---

## ④ 设计理念与权衡

### 设计哲学：渐进式优化

```
// 设计目标：在简单性和性能间找到平衡
简单易用(语法级支持) ←→ 高性能(锁升级优化)
    ↓                       ↓
自动锁管理               竞争适应优化
少死锁风险               减少系统调用
```

### 优缺点技术权衡

|优势|代价|
|---|---|
|​**语法简洁**​：关键字级支持|​**功能受限**​：无法实现锁投票、定时锁等|
|​**自动释放**​：防止锁泄漏|​**阻塞不可中断**​：可能造成线程永久等待|
|​**JVM深度优化**​：锁升级、锁消除|​**灵活性差**​：锁粒度控制不够精细|

---

## 2. 经典使用情景分析

### 场景1：单例模式的双重检查锁定

```
public class Singleton {
    private static volatile Singleton instance; // volatile保证可见性
    
    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查，避免不必要的同步
            synchronized (Singleton.class) { // 类锁保护
                if (instance == null) { // 第二次检查，防止重复创建
                    instance = new Singleton(); // 实例化
                }
            }
        }
        return instance;
    }
}
```

​**技术要点**​：

- volatile防止指令重排序导致的"部分初始化对象"问题
    
- 减小锁粒度，只有初始化时需要同步
    

### 场景2：生产者-消费者模型

```
public class BlockingQueue<T> {
    private final Object[] items;
    private int takeIndex, putIndex, count;
    private final Object notEmpty = new Object(); // 条件变量1
    private final Object notFull = new Object();  // 条件变量2
    
    public void put(T item) throws InterruptedException {
        synchronized (notFull) {
            while (count == items.length) {
                notFull.wait(); // 释放锁并等待
            }
            // ... 添加元素
            synchronized (notEmpty) {
                notEmpty.notify(); // 通知消费者
            }
        }
    }
}
```

​**技术要点**​：Object.wait()/notify()必须在外层synchronized块内调用

---

## 3. 工作原理与JVM实现

### 3.1 字节码层面实现

```
// 源代码
public synchronized void method() { /* ... */ }
public void block() { 
    synchronized(this) { /* ... */ } 
}

// 对应字节码
方法级同步：
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    
代码块同步：
  monitorenter   // 进入监视器
  // ... 代码块
  monitorexit    // 正常退出
  exception_table:
    monitorexit  // 异常退出保证释放锁
```

### 3.2 对象内存布局与锁机制

```
Java对象头 (64位JVM, 开启压缩指针)
|----------------------------------------------------------------------|
|  Mark Word (64 bits)                  |  Klass Word (32 bits)       |
|----------------------------------------------------------------------|
| 锁状态   | 内容                                                        |
|---------|------------------------------------------------------------|
| 无锁    | hashcode(25) | 分代年龄(4) | 偏向模式(1) | 锁标志(2)=01       |
| 偏向锁  | 线程ID(54) | 时间戳(2) | 分代年龄(4) | 偏向模式(1) | 锁标志(2)=01 |
| 轻量锁  | 指向栈中锁记录的指针(62)                           | 锁标志(2)=00 |
| 重量锁  | 指向互斥量(mutex)的指针(62)                       | 锁标志(2)=10 |
| GC标记  | 空(不需要记录信息)                               | 锁标志(2)=11 |
```

### 3.3 锁升级详细流程

```
// 锁状态变迁路径（JDK 6+ 优化）
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
     ↑        ↑           ↑
   初次同步  出现竞争     自旋失败
   
// 具体升级条件：
1. 偏向锁：-XX:+UseBiasedLocking开启，且无其他线程竞争
2. 轻量级锁：有竞争但很快能获取（CAS自旋成功）  
3. 重量级锁：自旋失败，线程进入阻塞队列
```

### 3.4 潜在技术问题与解决方案

#### 问题1：锁竞争激烈导致性能退化

```
// 反例：粗粒度锁
public class Counter {
    private int count1, count2;
    public synchronized void incrementBoth() { // 不必要的大范围同步
        count1++;
        count2++; 
    }
}

// 正例：锁分离/减小粒度
public class BetterCounter {
    private int count1, count2;
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    public void increment1() { synchronized(lock1) { count1++; } }
    public void increment2() { synchronized(lock2) { count2++; } }
}
```

#### 问题2：死锁

```
// 死锁示例
Thread1: synchronized(A) { synchronized(B) { ... } }
Thread2: synchronized(B) { synchronized(A) { ... } }

// 解决方案：锁排序
private final Object firstLock, secondLock;
public void orderedLock() {
    if (System.identityHashCode(obj1) < System.identityHashCode(obj2)) {
        firstLock = obj1; secondLock = obj2;
    } else {
        firstLock = obj2; secondLock = obj1;
    }
    synchronized(firstLock) {
        synchronized(secondLock) {
            // 临界区
        }
    }
}
```

#### 问题3：锁消除优化误判

```
// JVM可能进行锁消除的场景
public String createString() {
    StringBuffer sb = new StringBuffer(); // 局部变量，线程封闭
    sb.append("hello");                   // JVM可能消除同步操作
    return sb.toString();
}
```

​**预防**​：理解逃逸分析原理，避免过度同步局部对象

---

## 4. 面试技术深度考察

### Q1: synchronized的底层实现原理？

​**答案**​：

- ​**字节码层面**​：同步方法使用`ACC_SYNCHRONIZED`标志，同步块使用`monitorenter`/`monitorexit`指令
    
- ​**JVM层面**​：基于对象监视器(Monitor)，每个对象关联一个Monitor对象
    
- ​**操作系统层面**​：重量级锁使用操作系统的互斥量(mutex)，涉及用户态/内核态切换
    

### Q2: 锁升级的具体过程和触发条件？

​**答案**​：

1. ​**偏向锁**​：第一个线程访问时，CAS设置线程ID到Mark Word
    
2. ​**轻量级锁**​：出现竞争时，在原线程栈创建锁记录(Lock Record)，通过CAS将Mark Word指向锁记录
    
3. ​**重量级锁**​：自旋失败(默认10次)后，向操作系统申请互斥量，线程进入阻塞队列
    

### Q3: synchronized和ReentrantLock在性能上的对比？

​**答案**​：

- ​**低竞争场景**​：synchronized更优（偏向锁、轻量级锁优化）
    
- ​**高竞争场景**​：ReentrantLock更优（可控制的自旋、避免直接阻塞）
    
- ​**发展趋势**​：synchronized经过持续优化，性能差距逐渐缩小
    

### Q4: 什么是锁的"自适应自旋"？

​**答案**​：JVM根据锁的历史竞争情况动态调整自旋次数。如果之前自旋很快成功，则增加自旋次数；如果很少成功，则减少甚至直接阻塞。

### Q5: synchronized如何保证可见性和有序性？

​**答案**​：

- ​**可见性**​：遵循监视器锁规则，解锁前修改刷新到主内存，加锁时清空工作内存
    
- ​**有序性**​：as-if-serial语义保证单线程执行结果，内存屏障防止重排序跨越同步边界
    

### Q6: 偏向锁的优缺点和适用场景？

​**答案**​：

- ​**优点**​：无竞争时几乎零开销
    
- ​**缺点**​：撤销需要安全点(STW)，批量撤销影响性能
    
- ​**适用**​：明确单线程访问的场景，如早期StringBuffer（JDK9+已弃用偏向锁）
    

---

## 技术演进趋势

随着硬件发展（多核普及）和并发模式变化，synchronized持续优化：

- ​**JDK6**​：引入偏向锁、轻量级锁、自适应自旋
    
- ​**JDK15**​：逐步弃用偏向锁（因维护成本高、收益下降）
    
- ​**未来方向**​：更细粒度的锁优化、与协程结合
    

synchronized体现了**渐进式优化**和**实用主义**的设计哲学，在保证正确性的前提下，通过运行时优化平衡性能与复杂度。