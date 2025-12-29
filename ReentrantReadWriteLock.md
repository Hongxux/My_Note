- 需求背景：
	- 存在读多写少的情况：如商品信息查询、配置读取、缓存访问等，读操作的频率远远高于写操作
	- 读写互斥的必要性 
		- 写入可能破坏数据一致性
		- 读写同时进行，读取的时候可能读取到不一致状态
	- 读读互斥的不合理性：
		- 读多写少情况：同一时间，读读的情况多，读写的情况少
		- 读读并不会修改数据，不会出现读到不一致状态的问题
- 解决方案：读写锁--放松同一时刻读读之间的限制
	- 核心规则：使用两把锁
		- 读锁 (ReadLock)：**共享锁**。
			- 【共享】只要没有线程持有写锁，多个线程可以同时获取读锁并执行读操作。
			- 目的：在没有线程进行写操作的时候，读读之间不互斥，提高并发能力 
		- 写锁 (WriteLock)：独占锁
			- 【独占】其他任何线程（无论是想要读还是写）都无法再获取读锁或写锁，必须等待该写锁释放。
			- 目的：保证写操作的原子性和可见性
				- 原子性：避免不一致
				- 可见性：修改可见
	- 支持锁降级：写锁可降级为读锁（持有写锁时可获取读锁，再释放写锁），反之不行；
- 问题:
	- 写线程饥饿：
		- 原因：
			- 读多写少且读操作非常频繁，导致写锁难以介入
			- 写锁的获取需要等待所有读锁释放，内部实现比ReentrantLock复杂
		- 解决措施：StampedLock
	- 读锁升级成死锁会出现死锁：线程因等待自己释放读锁而永久阻塞
	- 读锁（ReadLock）不支持 Condition，写锁（WriteLock）完全支持
		1. **写锁（WriteLock）支持 Condition**：因为写锁是独占锁，符合 AQS Condition 对 “独占持有锁” 的底层要求；
		2. **读锁（ReadLock）不支持 Condition**：
			- 底层原因：Condition 依赖独占锁语义，而读锁是共享锁，`isHeldExclusively()`返回 false，无法通过 Condition 的校验；
			- 语义原因：共享锁的 “多线程持有” 与 Condition 的 “独占唤醒执行” 语义冲突；
---
- 实现原理：
    - 同步状态（state）定义：**高 16 位存读锁计数，低 16 位存写锁重入次数**：
	    - 低 16 位 = 0：写锁未被持有；
	    - 高 16 位 > 0：有线程持有读锁（数值 = 读锁总持有数）；
	    - 写锁可重入：低 16 位累加；读锁可重入：高 16 位累加。
	- 核心钩子方法实现：拆分为`ReadLock`（共享）和`WriteLock`（独占），分别实现 AQS 的共享 / 独占钩子：
		- **WriteLock（独占）**：
		    - tryAcquire：检查读锁是否被持有（高 16 位 > 0）或写锁被其他线程持有→失败；否则 CAS 修改低 16 位（重入则 + 1）；
		    - tryRelease：低 16 位 - 1，减到 0 则释放，返回 true 触发唤醒；
		- **ReadLock（共享）**：
		    - tryAcquireShared：
			    - 检查写锁是否被其他线程持有
				    - 如果是：失败（返回 - 1）加入同步队列等待
				    - 否则 AS 增加高 16 位（读锁计数 + 1），返回 1（成功）；
		    - tryReleaseShared：
			    - 高 16 位 - 1，如果不为0则返回false，不触发唤醒传播
			    - 如果减到 0 则返回 true，触发 AQS 的唤醒传播。
	-  关键设计
		- 锁降级：写锁可降级为读锁（持有写锁时可获取读锁，再释放写锁），反之不行；
		- 读锁共享：基于 AQS 的共享模式，多个读线程可同时获取，释放时触发唤醒传播。
- 使用示例：就是获取了两把锁，一把读锁，一把写锁
	```
	import java.util.concurrent.locks.ReentrantReadWriteLock;
	import java.util.concurrent.locks.Lock;
	
	public class ReadWriteLockDemo {
	    // 1. 创建读写锁实例
	    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
	    // 2. 从中分离出读锁和写锁
	    private final Lock readLock = rwLock.readLock();
	    private final Lock writeLock = rwLock.writeLock();
	    // 共享数据，需要被保护
	    private String sharedData = "Hello, World";
	
	    // 读数据的方法 - 使用读锁
	    public String readData() {
	        readLock.lock(); // 获取读锁
	        try {
	            // 多个线程可以同时进入此区域，只要没有写锁被持有
	            return sharedData;
	        } finally {
	            readLock.unlock(); // 务必在finally块中释放锁
	        }
	    }
	
	    // 写数据的方法 - 使用写锁
	    public void writeData(String newData) {
	        writeLock.lock(); // 获取写锁
	        try {
	            // 同一时间只有一个线程可以进入此区域
	            sharedData = newData;
	        } finally {
	            writeLock.unlock();
	        }
	    }
	}
	```
	- 读锁特点
		- 不支持条件变量
		- 不支持锁升级：持有读锁的情况下去获取写锁，会导致死锁

	- 写锁特点
		- 支持条件变量
		- 支持锁降级：
			- 含义：持有写锁的时候，获取读锁，再释放写锁
			- 设计目的：
				- 确保你修改数据，释放写锁后，其他线程无法修改数据
				- 本线程后续读取到的数据是自己修改后的最新值
					- 防止数据不一致
	- 二者特点
		- 支持可重入
		- 都可以配置公平模式和非公平模式