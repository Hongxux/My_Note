- Timer实现定时功能的局限性
	- 单线程阻塞：Timer 的单线程模型，所有任务都在一条线程上顺序执行
		- 存在一个执行时间很长的任务，它会直接“卡住”整个调度队列
	- 异常处理脆弱：一旦某个 TimerTask 的 `run`方法中抛出了未捕获的异常，整个 Timer 线程就会立即终止
	- 调度基础依赖绝对时间currentTimeMillis
		- 当系统时间被手动调整或由 NTP 同步时，会导致调度时间混乱
- 解决方法：支持定时与周期性任务：`newScheduledThreadPool(int corePoolSize)`
	- 使用场景：创建一个支持定时及周期性任务执行的线程池
	- 使用的队列：延迟队列`DelayedWorkQueue` 
	- 特点：
		- 基于线程池实现，多线程
			- 任务在不同线程执行，互不干扰：不会出现一个异常，其余全终止情况
			- 任务阻塞，不影响其他任务的定时与调度
		- 调度基础：相对时间​ (`nanoTime`)
			- 不受系统时钟更改影响
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