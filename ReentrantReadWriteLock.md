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
- 实现原理：
	- 实现基础：
		- state：
			- 高 16 位记录读锁数量
			- 低 16 位记录写锁重入次数
		- 读锁和写锁公用一个阻塞队列
			- 但是会标记每个节点是独占锁还是共享锁
	- 写锁的获取：
		- 检查状态：首先检查整个 state值(c)。
			- 如果 c != 0，说明有锁被占用。此时需进一步判断：
				- 如果写锁计数 w = 0，说明有线程持有读锁，获取失败。
				- 如果写锁计数 w != 0，但持有写锁的线程不是当前线程，获取失败。
				- 如果写锁计数 w != 0且当前线程是持有者，则进行重入判断。
					- 重入检查：检查重入次数是否会超过最大值（低16位最大值 2^16 - 1）。
			- 如果c 
		- 获取失败：
			- 调用 writerShouldBlock()方法。
				- 在非公平锁中，该方法直接返回 false，允许插队，以提高吞吐量。
				- 在公平锁中，该方法会检查同步队列中是否有前驱节点在待 (hasQueuedPredecessors())，有则需排队。
		- 获取成功：
			- 通过 CAS 操作将 state值加 1（实际上是低16位加1）。
			- 设置独占线程为自己
	- 读锁的获取：可重入锁
		1. **检查写锁**：
			- 如果当前有**其他线程**持有写锁 (`exclusiveCount(c) != 0`且 `getExclusiveOwnerThread() != current`)，则读锁获取立即失败。这确保了“读写互斥”。
				- **例外**：如果当前线程自身持有写锁，是允许获取读锁的，这就是**锁降级**的关键
		2. **阻塞策略**：调用 `readerShouldBlock()`方法。这个方法的实现是**公平性的核心**：
		    - **公平模式**：检查队列中是否有前驱节点在等待，有则返回 `true`，当前读线程需要排队。
		    - **非公平模式**：为实现一定的“写锁友好性”，会检查同步队列中第一个等待节点是否是**写锁**（`apparentlyFirstQueuedIsExclusive()`）。如果是，则读锁应该阻塞，以避免写线程饥饿。
		3. **计数与重入**：如果不需要阻塞且读锁计数未超限，则尝试通过 CAS 更新 `state`的高16位（`c + SHARED_UNIT`）。
			- 成功后，需要记录每个读线程的重入次数。这里使用了 `ThreadLocal`和一个用于缓存最近一个读线程计数的 `cachedHoldCounter`来优化性能
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