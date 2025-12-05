---
aliases:
  - 对象锁
---
- 需求背景：
	- 避免临界区的竞态条件发生
- 语法：
	- 加到代码块：![[Pasted image 20251130193039.png]]
		- 对象（锁对象）：
			- 实现临界区代码的原子性的前提：确保多个进程锁住的是同一个对象
			- 互斥的范围：使用相同对象锁的线程
		- 临界区：对共享变量的操作指令流
	- 加到方法上：
		- 加到成员方法上：等价于把这个类的当前实例作为锁对象![[Pasted image 20251130194844.png]]
			- 同一个实例的同步实例方法会互斥
			- 不同实例的同步实例方法互不干扰
		- 加到静态方法上：等价于把这个类的类对象作为锁对象![[Pasted image 20251130195030.png]]
			- **全局锁**：所有调用该类任何同步静态方法的线程都会互斥。
				- 即使针对不同实例
			
- 效果：按照一定使用规范才能实现的共享变量可见性和有序性，临界区代码块的原子性
	- 实现临界区代码的原子性
		- 如果没有人获取锁，则直接获取对象锁，执行临界区代码
		- 如果发现这个对象锁被别人获取了，则进入阻塞状态，直至锁释放，进行锁的争抢
	- 实现共享变量的可见性：
		- 可见性的实现前提：这个共享变量的所有读写操作都被同一锁对象监控（Monitor）
		- 进入synchronized代码块的时候：从主内存更新工作内存的共享变量为最新值
		- 退出synchronized代码块的时候：把工作内存中的共享变量都更新到主内存中
	- 实现共享变量的有序性
		- 实现有序性的条件：这个共享变量的所有读写操作都被同一锁对象监控（Monitor）
		- 在synchronized中会发生重排序，但是如果所有读写操作都被同一锁对象监控，则可以实现最终的有序性
- 可见性和有序性的实现原理
	- 在获取锁的时候加入读屏障
	- 在释放锁的时候加入写屏障
- 原子性的实现原理：
	- ​**字节码层面**​：同步方法使用`ACC_SYNCHRONIZED`标志，同步块使用`monitorenter`/`monitorexit`指令
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
	- ​**JVM层面**​：基于对象监视器(Monitor)，每个对象关联一个Monitor对象
	    
	- ​**操作系统层面**​：重量级锁使用操作系统的互斥量(mutex)，涉及用户态/内核态切换
	- 锁升级：
		- 目的：根据竞争的激烈程度，找到合适的上锁方式
			- 越重量的锁，开销越大
		- 锁升级方式：**无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁**。
		- 锁升级特点：过程是单向的，不可逆
	- [[JAVA对象头]]中的Mark Work： 表明锁的类型，以及锁的引用
	- 无锁：有锁和无锁的性能差距有十几倍
		- 触发时机：锁对象不可能被共享
	- [[偏向锁]]：与轻量锁相比，减少重入锁的时候，CAS获取锁和释放锁的消耗
	- [[轻量锁]]：与重量级锁相比，减少线程阻塞和线程唤醒带来的上下文切换的消耗
	- 重量级锁：[[Monitor]]（监视器、管程）
		- 每个JAVA对象都可以关联一个Monitor对象
		- 获取锁，释放锁，等待，通知，本质上都是在与JAVA对象所关联的Monitor对象进行交互




    
- ​**可见性**​：遵循happens-before原则，解锁前修改刷新到主内存
    
- ​**有序性**​：防止临界区内指令重排序（as-if-serial语义）
    

---

### 与ReentrantLock的对比关系

|维度|synchronized|ReentrantLock|
|---|---|---|
|​**实现层面**​|JVM内置原语|JDK API实现|
|​**锁获取**​|悲观阻塞，无法中断|支持tryLock、可中断|
|​**公平性**​|非公平|可配置公平/非公平|
|​**性能趋势**​|低竞争时优化更好|高竞争时更可控|



    


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







    



### Q3: synchronized和ReentrantLock在性能上的对比？

​**答案**​：

- ​**低竞争场景**​：synchronized更优（偏向锁、轻量级锁优化）
    
- ​**高竞争场景**​：ReentrantLock更优（可控制的自旋、避免直接阻塞）
    
- ​**发展趋势**​：synchronized经过持续优化，性能差距逐渐缩小
    

### Q5: synchronized如何保证可见性和有序性？

​**答案**​：

- ​**可见性**​：遵循监视器锁规则，解锁前修改刷新到主内存，加锁时清空工作内存
    
- ​**有序性**​：as-if-serial语义保证单线程执行结果，内存屏障防止重排序跨越同步边界
    
