- 需求背景：synchronized 的局限性
	- 无法响应中断的阻塞风险：
		- 表现：线程在等待锁的过程中被阻塞，它无法响应其他线程发出的中断请求，只能一直等待下去
		- 响应中断的需求：
			- 实现可取消的任务
			- 避免死锁，可以避免其无限制等待
			- 管理系统关闭
	- 缺乏尝试获取锁的灵活性：
		- 表现：all-or-nothing模式
			- 要么获取成功锁
			- 要么一直阻塞等待锁
		- 需求：当锁被占用时，我们可能希望线程能先去执行其他任务，而不是傻等
	- 不公平竞争：造成饥饿问题
	- 唤醒不够精确：
		- 表现：`Object`的 `wait()`, `notify()`, `notifyAll()`机制，所有等待线程都在同一个等待队列上。这可能导致唤醒不精确
			- 造成惊群效应，造成不必要的CPU竞争
- 解决措施：ReentrantLock
	- 被动避免死等：`lockInterruptibly()`方法加可打断锁，允许在等待锁的过程中响应中断，从而提供了优雅退出的机制
		- 别的线程可以使用interrupt方法打断
		- 需要处理InterruptedException异常
			- 在catch块优雅退出
	- 主动避免死等或对响应时间有时效性要求
		- `tryLock()`
			- 非阻塞
			- 尝试一次立即返回
		- `tryLock(long time, TimeUnit unit)`
			- 限时阻塞
			- 在指定时间内尝试获取锁
		- 支持中断，也要处理InterruptedException异常异常
		- 通过返回值判断是否获取成功锁
			```java
				if(lock.trylock()){
					//获取锁成功
					try{
						//执行相关业务
					}finally{
						lock.unlock()
					}
				}else{
					//获取锁失败，执行相关代码
				}	
			```
	- 可以设置为公平锁：在构造函数中通过 `true`参数来创建公平锁，严格按照 FIFO（先进先出）的顺序分配锁
		- 性能受影响 ，降低并发度
		- 替代方案：使用trylock尝试获取锁，也能避免饥饿
	- 支持多个条件变量：
		- 实现方式：以`Condition`条件对象为单位进行等待和唤醒，允许一个锁关联多个条件变量
			- 相当于：有多个WaitSet。对应不同的唤醒条件
		- 设计目的：实现更精细的线程间通信
			- 避免惊群效应
		- 使用方式：
			1. 创建 `Condition`对象：
				- `Condition conditon= lock.newCondition();`
			2. 等待方：
				- 使用前提：获取锁
				- `await()`：当前线程释放锁并等待，唤醒方式：
					- 被中断
					- 被其他线程通过 `signal()`唤醒
				- `await(long time, TimeUnit unit)`：当前线程释放锁并等待
					- 唤醒方式：
						- 被`signal()`唤醒
						- 被中断
						- 超时。
					- 返回值：表明是否在超时前被唤醒
				- `awaitUninterruptibly()`：当前线程释放锁并等待唤醒方式：
					- 被其他线程通过 `signal()`唤醒
				- 使用模板：
					```
					lock.lock(); // 1. 必须先获取锁
					try {
					    while (!条件满足) { // 2. 必须在循环中检查条件，防止虚假唤醒
					        condition.await(); // 3. 条件不满足，释放锁并等待
					    }
					    // 4. 条件满足，执行后续业务逻辑
					} catch (InterruptedException e) {
					    Thread.currentThread().interrupt(); // 恢复中断状态
					} finally {
					    lock.unlock(); // 5. 务必在finally块中释放锁
					}
					```
			3. 通知方
				- `signal()`:**唤醒**在此 Condition 上等待的**一个**线程（通常是等待时间最长的）
				- `signalAll()`：**唤醒**在此 Condition 上等待的**所有**线程
				```
				lock.lock(); // 1. 必须先获取锁
				try {
					// 2. 改变共享状态，使条件得以满足
					修改共享变量();
					condition.signal(); // 3. 唤醒等待队列中的一个线程
				} finally {
					lock.unlock(); // 4. 释放锁，被唤醒的线程才能有机会获取锁并继续执行
				}
				```

- 实现原理：![[Pasted image 20251203130920.png]]
	
- 基础语法：
```
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockExample {
    private final ReentrantLock lock = new ReentrantLock();

    public void criticalMethod() {
        lock.lock(); // 手动获取锁
        try {
            // 临界区代码
        } finally {
            lock.unlock(); // 必须在finally块中手动释放锁，确保锁一定被释放
        }
    }
}
```

