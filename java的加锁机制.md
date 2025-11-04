
## 一句话总结

​**Java加锁机制通过synchronized和Lock接口实现线程同步，核心是建立内存屏障和happens-before关系，在性能与安全性间权衡。​**​

---

## ① 定义

​**Java加锁机制**是一套用于控制多个线程对共享资源并发访问的同步原语，通过建立临界区保证操作的原子性、可见性和有序性，主要包含：

- ​**内置锁**​：`synchronized`关键字（JVM级别）
    
- ​**显式锁**​：`java.util.concurrent.locks.Lock`接口（API级别）
    

---

## ② 关系网络

### 问题解决链

```
竞态条件 
    → 加锁解决(原子性/可见性) 
    → 带来性能瓶颈/死锁  
    → 通过锁优化(偏向/轻量/重量级)、读写锁、CAS解决
```

### 替代/补充关系

- ​**synchronized**​ vs ​**ReentrantLock**​：后者是前者的功能补充（可中断、超时、公平性等）
    
- ​**悲观锁**​ vs ​**乐观锁**​：CAS是加锁的轻量级替代方案
    
- ​**内置锁**​ vs ​**显式锁**​：显式锁提供更细粒度的控制
    

### 易混淆概念辨析

|概念对|区别点|
|---|---|
|​**可重入锁**​ vs ​**不可重入锁**​|同一线程能否重复获取同一把锁|
|​**公平锁**​ vs ​**非公平锁**​|按申请顺序分配还是允许插队|
|​**乐观锁**​ vs ​**悲观锁**​|假设冲突频率低(重试) vs 假设冲突频率高(阻塞)|

---

## ③ 技术定位

​**所属领域**​：并发编程 → 线程同步 → 互斥锁机制

​**技术基础**​：

- 建立在JMM(Java内存模型)的happens-before规则上
    
- 依赖操作系统的互斥量(mutex)和监视器(monitor)机制
    
- 基于AQS(AbstractQueuedSynchronizer)框架实现
    

---

## ④ 设计理念与权衡

### 设计理念

```
// 从简单到复杂的演进路径
synchronized → ReentrantLock → StampedLock → 无锁编程
    ↑              ↑              ↑            ↑
 易用性强      功能丰富      性能优化      极致性能
 功能有限      复杂度高      使用复杂      实现困难
```

### 优缺点权衡矩阵

|锁类型|优势|代价|
|---|---|---|
|​**synchronized**​|JVM内置优化、自动释放、简单安全|功能受限、无法中断|
|​**ReentrantLock**​|可中断、超时、公平性、条件变量|需手动释放、代码复杂|
|​**ReadWriteLock**​|读写分离、读并发高|写饥饿、复杂度高|

---

## 2. 经典使用情景

### 场景1：账户转账操作

```
class BankAccount {
    private final Object lock = new Object();
    private int balance;
    
    public void transfer(BankAccount target, int amount) {
        // 死锁预防：按固定顺序获取锁
        Object firstLock = System.identityHashCode(this) < 
                          System.identityHashCode(target) ? this : target;
        Object secondLock = firstLock == this ? target : this;
        
        synchronized(firstLock) {
            synchronized(secondLock) {
                if (this.balance >= amount) {
                    this.balance -= amount;
                    target.balance += amount;
                }
            }
        }
    }
}
```

​**触发条件**​：多线程并发转账

​**关键特征**​：需要同时锁定多个资源、存在死锁风险

### 场景2：缓存读写分离

```
class ReadWriteCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    
    public V get(K key) {
        rwLock.readLock().lock();  // 允许多个读线程并发
        try {
            return cache.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        rwLock.writeLock().lock();  // 写锁排他
        try {
            cache.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

​**触发条件**​：读多写少、数据一致性要求

​**关键特征**​：读写操作频率差异大、需要保证读写可见性

---

## 3. 工作原理与实现

### synchronized实现原理

```
线程进入synchronized块
    ↓
尝试通过CAS获取锁(偏向锁→轻量级锁)
    ↓
成功: 执行临界区代码 (锁记录指向对象Mark Word)
    ↓  
失败: 自旋尝试 or 进入阻塞队列(重量级锁)
    ↓  
执行完毕: 释放锁，唤醒等待线程
```

​**锁升级过程**​（JVM优化）：

```
无锁 → 偏向锁(单线程重入) → 轻量级锁(自旋CAS) → 重量级锁(OS互斥量)
```

### ReentrantLock的AQS实现

```
// 简化的AQS获取锁流程
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&  // 1. 尝试直接获取
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) { // 2. 加入CLH队列
        Thread.currentThread().interrupt(); // 3. 响应中断
    }
}
```

### 潜在问题与解决方案

|问题|原因|解决方案|
|---|---|---|
|​**死锁**​|循环等待资源|锁顺序化、超时机制(tryLock)、死锁检测|
|​**活锁**​|线程不断重试失败|引入随机退避、限制重试次数|
|​**锁饥饿**​|低优先级线程长期等待|公平锁、优先级调整|
|​**性能瓶颈**​|锁粒度太粗|减小临界区、读写分离、无锁数据结构|

---

## 4. 面试重点与答案

### Q1: synchronized和ReentrantLock的区别？

​**答案**​：

- ​**机制层面**​：synchronized是JVM内置监视器锁，ReentrantLock基于AQS的CLH队列
    
- ​**功能层面**​：ReentrantLock支持可中断、超时获取、公平锁、多个条件变量
    
- ​**性能层面**​：低竞争时synchronized有偏向锁优化，高竞争时ReentrantLock更可控
    
- ​**使用层面**​：synchronized自动释放，ReentrantLock需手动unlock
    

### Q2: 什么是锁的可重入性？为什么重要？

​**答案**​：同一线程可重复获取已持有的锁，避免自死锁。实现原理是通过记录持有线程和重入计数。

### Q3: 锁优化技术有哪些？

​**答案**​：

- ​**JVM级别**​：锁消除(Escape Analysis)、锁粗化、偏向锁、自适应自旋
    
- ​**应用级别**​：减小锁粒度、读写分离、无锁编程(CAS)、ThreadLocal
    

### Q4: 如何选择synchronized和ReentrantLock？

​**答案**​：

- ​**优先synchronized**​：简单场景、锁竞争不激烈、需要自动管理
    
- ​**选择ReentrantLock**​：需要高级功能(超时、公平性)、竞争激烈、需要条件变量
    

### Q5: 什么是AQS？如何工作？

​**答案**​：AbstractQueuedSynchronizer是JUC锁实现的基础框架，通过volatile state表示同步状态，CLH队列管理等待线程，模板方法模式让子类实现tryAcquire/tryRelease。

---

## 技术演进趋势

```
// 从传统锁到现代并发模式的演进
synchronized/Lock → StampedLock(乐观读) → 无锁(CAS) → Actor模型 → 协程
    ↓                   ↓               ↓         ↓          ↓
 阻塞同步        读写优化         原子变量    消息传递    轻量级线程
```

这种设计体现了**分层抽象**和**关注点分离**的软件工程原则，在不同场景下提供合适的并发控制粒度。