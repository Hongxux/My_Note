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
	- CAS尝试获取锁的插队机制，能减少上下文的切换，提高性能
	- 被动避免死等：`lockInterruptibly()`方法加可打断锁，允许在等待锁的过程中响应中断，从而提供了优雅退出的机制
		- 别的线程可以使用interrupt方法打断
		- 需要处理InterruptedException异常
			- 在catch块优雅退出
		- 实现原理：
			- 不可中断：标准的 `lock()`方法内部调用的是 AQS 的 `acquire(int arg)`方法。该方法在线程被挂起（`LockSupport.park()`）后，如果被中断，**只会设置一个中断标志位，而不会立即抛出异常或返回**。线程会继续被阻塞，直到被正常唤醒后，才会检查中断标志，并在获取锁后才响应中断
			- 可中断：使用 `lockInterruptibly()`方法，它内部调用 `acquireInterruptibly(int arg)`。该方法在线程被中断时，会**立即抛出 `InterruptedException`**，让调用者可以立刻处理中断，实现了对中断的快速响应
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
		- 实现原理：
			- 非公平锁：允许插队
				- 线程在调用 `lock()`时，会**直接尝试**使用 CAS 抢夺 `state`，不管队列里有没有其他线程在等待 。如果抢夺失败，才会加入队列。即使在队列中排队时，被唤醒的线程在获取锁前，也会和新来的线程一起竞争，这可能导致“插队”现象，
			- 公平锁：不允许插队
				- 如果队列中有等待的线程，则不进行插队
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
		- 实现原理：
			- **等待 (`await()`)**：当线程调用 `condition.await()`时：
			    1. 会释放当前持有的锁。
			    2. 将当前线程包装成 Node 节点，加入与该 `Condition`关联的**条件队列**（注意，这是另一个队列，不同于 AQS 的同步队列）。
			    3. 将线程挂起，等待被唤醒
			- **唤醒 (`signal()`)**：当线程调用 `condition.signal()`时：
			    1. 会将条件队列中的**第一个等待节点**转移到 AQS 的**同步队列的末尾**，使其重新具备竞争锁的资格。
			    2. 被转移的线程在同步队列中被唤醒后，会重新尝试获取锁。只有成功获取锁后，才会从 `await()`方法中返回
- 问题 ：
	- 忘记手动释放锁（最常见的致命坑）和其衍生问题：重入次数不匹配
	- Condition 条件变量使用不当
		- `await()` 后未调用 `signal()`/`signalAll()`，导致线程永久等待；
		-  未在循环中检查条件，引发 “虚假唤醒”（线程被唤醒后，条件仍不满足）。
----

实现原理：基于AQS![[Pasted image 20251203130920.png]]
- 同步状态state：**锁的重入次数**
- 核心钩子方法实现：仅实现`tryAcquire`（获取）、`tryRelease`（释放）：
	- **tryAcquire（独占获取）**：分公平 / 非公平：
	    - 非公平（默认）：直接 CAS 抢锁（state 从 0→1），抢不到再检查是否是当前线程重入（state+1）；
	    - 公平：先检查队列是否有等待线程，有则排队，无则 CAS 抢锁（保证 FIFO）；
	- **tryRelease（独占释放）**：
	    - 当前线程持锁时，state-1；
	    - 只有 state 减到 0 时，才真正释放锁（清空独占线程标记），返回 true 触发 AQS 唤醒后继。
- 关键设计
	- 可重入：`tryAcquire`中判断 “当前线程是否是已持有锁的线程”，是则直接 state+1；
	- 公平 / 非公平：核心差异在`tryAcquire`是否检查队列（公平模式必须等队列空才抢锁）。
	  



---
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

