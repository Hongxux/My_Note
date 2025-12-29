- 需求背景：
	- Timer实现定时功能的局限性
		- 单线程阻塞：Timer 的单线程模型，所有任务都在一条线程上顺序执行
			- 存在一个执行时间很长的任务，它会直接“卡住”整个调度队列
		- 异常处理脆弱：一旦某个 TimerTask 的 `run`方法中抛出了未捕获的异常，整个 Timer 线程就会立即终止
		- 调度基础依赖绝对时间currentTimeMillis
			- 当系统时间被手动调整或由 NTP 同步时，会导致调度时间混乱
	- ThreadPoolExecutor+DelayQueue的局限性，不支持周期性任务
		- 需要额外的操作让队列入队
- 解决方法：支持定时与周期性任务：`newScheduledThreadPool(int corePoolSize)`
	- 使用的队列：延迟队列`DelayedWorkQueue` 
		- **基于堆的优先级队列**：内部使用数组实现小顶堆，按任务的「下次执行时间」排序结果越小，优先级越高。
		- **阻塞与唤醒**：线程从队列获取任务时，若任务未到执行时间，会计算等待时间并阻塞（通过 `Condition.awaitNanos(...)`），避免「忙等」；当有新任务加入或任务取消时，会唤醒阻塞的线程重新排序堆。
		- **无界特性**：队列容量理论上无上限（仅受内存限制），因此 `ScheduledThreadPoolExecutor` 的拒绝策略（默认 `AbortPolicy`）仅在以下场景触发：线程池已关闭后提交任务，或任务本身无法被添加到队列（极罕见）。
	- 特点：
		- 基于线程池实现，多线程
			- 任务阻塞，不影响其他任务的定时与调度
		- 调度基础：相对时间​ (`nanoTime`)
			- 不受系统时钟更改影响
		- 异常处理对周期性任务的影响
			- 若周期性任务抛出**未捕获异常**，该任务会被自动取消，后续周期性执行将终止
			- 解决措施：在任务内部捕获所有异常，或通过 `ScheduledFuture.get()` 捕获
		- 智能的延迟策略
			- **固定频率**的痛点：**任务重叠**：让同一个任务的两个实例**并发执行**（一个还未结束，另一个又开始）。这可能会导致数据竞争和状态混乱。
			- 设计思想：实现固定延迟，保证正确性和安全性
			- 具体实现：
				- **不会并发执行**：这是基本保证。即使执行时间超过了周期，同一个任务的两次执行也**绝不会重叠**。
				- **补偿策略**：当某次执行超时后，它会计算**应该执行的次数**在当前执行结束后，**立即安排下一次执行**，然后后续的执行会尝试按照原始计划时间进行
					- 问题：导致连续执行，形成“追赶效应”。
	- 使用的示例：
		```
		ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(3);
		// 延迟5秒执行
		scheduledThreadPool.schedule(new Task(), 5, TimeUnit.SECONDS);
		// 延迟1秒后，每3秒固定频率执行一次（不论任务执行时间）
		scheduledThreadPool.scheduleAtFixedRate(new Task(), 1, 3, TimeUnit.SECONDS);
		// 延迟1秒后，每次任务执行完成后，间隔3秒再开始下一次执行
		scheduledThreadPool.scheduleWithFixedDelay(new Task(), 1, 3, TimeUnit.SECONDS);
		```

----
