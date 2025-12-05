- 需求背景：ReentrantReadWriteLock的瓶颈
	- 悲观读写，且共用一个等待队列
	- 在读线程极多时，写线程可能**长期饥饿**（无法获得锁）
- 实现方式：乐观读
	1. 尝试而非阻塞：当线程执行乐观读时（通过 `tryOptimisticRead()`方法），它并不真正加锁，而是立即返回一个类似于“版本号”的 Stamp（戳记）**​
	2. 读取数据：线程接着去读取共享数据。
	3. 验证：在使用数据前，线程必须调用 `validate(stamp)`方法验证这个 Stamp 是否依然有效。验证的本质是：自从我获取这个 Stamp 以来，有没有任何写操作完成？**​
	    - 如果有效：说明数据在读过程中没有被修改，读取成功！整个过程完全没有加锁开销。
	    - 如果无效：说明数据可能已被其他写线程修改，此时读操作需要“降级”为传统的悲观读锁（`readLock()`），然后重新读取数据，以确保数据一致性
- 问题：
	- 确保写操作极其稀少
		- 因为乐观读和验证版本戳之间有窗口期
	- 同一线程重复获取锁会导致死锁
	- 需要正确管理 Stamp 和调用 `validate`方法，容易误用
	- 虽提高了吞吐量，但无法保证线程的公平性。
- 使用示例：
	```
	import java.util.concurrent.locks.StampedLock;
	
	public class Point {
	    private double x, y;
	    private final StampedLock lock = new StampedLock(); // 创建StampedLock实例
	
	    // 使用写锁修改坐标 - 独占操作
	    public void move(double deltaX, doubleY) {
	        long stamp = lock.writeLock(); // 获取写锁，可能阻塞
	        try {
	            x += deltaX;
	            y += deltaY;
	        } finally {
	            lock.unlockWrite(stamp); // 使用正确的stamp释放写锁
	        }
	    }
	
	    // 使用乐观读计算到原点的距离
	    public double distanceFromOrigin() {
	        // 1. 尝试乐观读：不阻塞，立即返回一个戳记（版本号）
	        long stamp = lock.tryOptimisticRead();
	        // 2. 读取数据到局部变量
	        double currentX = x, currentY = y;
	        
	        // 3. 验证：在读数据期间，是否有写操作发生？
	        if (!lock.validate(stamp)) { // 如果验证失败（stamp失效）
	            // 4. 降级为悲观读锁：重新读取数据，此时会阻塞写操作
	            stamp = lock.readLock();
	            try {
	                currentX = x;
	                currentY = y;
	            } finally {
	                lock.unlockRead(stamp);
	            }
	        }
	        // 5. 计算并返回结果
	        return Math.sqrt(currentX * currentX + currentY * currentY);
	    }
	
	    // 使用悲观读锁计算距离（传统方式，对比用）
	    public double distanceFromOriginPessimistic() {
	        long stamp = lock.readLock(); // 获取悲观读锁，会阻塞写操作
	        try {
	            return Math.sqrt(x * x + y * y);
	        } finally {
	            lock.unlockRead(stamp);
	        }
	    }
	}
	```
	- 读锁和写锁都要配合一个版本戳使用
		- 使用悲观读锁
			- 加锁的时候获得版本戳
			- 解锁的时候需要提供这个版本戳
		- 使用乐观读锁
			- 加锁的时候获得版本戳
			- 使用完共享数据后验证版本戳
				- 如果不变，则使用期间数据未被修改，仍然有效
				- 如果改变，则使用期间数据被修改，因此需要降为悲观读锁
		- 使用悲观写锁
			- 加锁的时候获得版本戳
			- 解锁的时候需要提供这个版本戳