- 需求背景：CountDownLatch的局限性
	- 多个线程在某个点集合，然后同时开始下一阶段：任务线程之间互相等待
		- 示例：
			- 在大型系统（如微服务）启动时，可能需要并行加载多种资源，如数据库连接、缓存数据、配置文件等
				- 集合是为了安全性：保证资源准备完毕，才开始服务
			- 在多人在线游戏或分布式仿真系统中，需要维持一个所有玩家或节点共享的统一状态
				- 集合是为了公平性
			- 并行计算与分阶段任务
				- 集合是避免读取到其他线程尚未计算完成的、不完整的中间结果，导致数据错误
		- 特点：
			- 是任务线程之间互相等待
				- CountDownLatch是单向等待，一般都是主线程等待工作线程
	- 一个大任务可能有多个阶段需要同时开始
		- CountDownLatch计数器归零后无法重置
			- 只能使用一次 ，多个阶段需要多个计时器
	- 在唤醒所有等待线程之前，可能需要准备下一阶段所需的资源
- 解决措施：CyclicBarrier
	- 线程之间相互等待
		- 在到达集合点后，用await()声明“我到了”，并安心等待其他工作线程
		- 由底层框架处理复杂的同步通知逻辑，极大简化了代码
	- 可循环使用
		- 屏障在所有线程到达后会自动或手动重置，可用于多阶段任务。
		- 无需为每个阶段创建新的同步对象
	- CyclicBarrier的构造函数允许传入一个 Runnable任务（称为 barrierAction）。
		- 执行的时机：当所有线程到达屏障时，在唤醒所有线程之前，
		- 执行的线程：最后一个到达屏障的线程
		- 作用：在阶段切换时执行一些共享状态的更新、日志记录或初始化下一阶段所需资源等操作
- 使用示例：多阶段任务与屏障操作
	```
	import java.util.concurrent.*;
	
	public class MultiStageExample {
	    // 保存每个线程的计算结果
	    private static ConcurrentHashMap<String, Integer> resultMap = new ConcurrentHashMap<>();
	    // 假设有3个工作线程
	    private static final int WORKER_COUNT = 3;
	
	    public static void main(String[] args) {
	        // 1. 创建CyclicBarrier，并指定屏障开放后执行的“合并结果”任务
	        CyclicBarrier barrier = new CyclicBarrier(WORKER_COUNT, () -> {
	            // 此Runnable任务将在所有工作线程到达屏障后，由最后一个到达的线程执行
	            System.out.println("\n所有线程已完成计算，开始合并结果...");
	            int totalResult = 0;
	            for (Integer value : resultMap.values()) {
	                totalResult += value;
	            }
	            System.out.println("最终结果为: " + totalResult);
	        });
	
	        // 2. 创建线程池并提交任务
	        ExecutorService executor = Executors.newFixedThreadPool(WORKER_COUNT);
	        for (int i = 0; i < WORKER_COUNT; i++) {
	            final int workerId = i;
	            executor.submit(() -> {
	                try {
	                    // 第一阶段：每个线程独立计算
	                    int partialResult = (int) (Math.random() * 100);
	                    resultMap.put("Worker-" + workerId, partialResult);
	                    System.out.println("Worker-" + workerId + " 计算完成，局部结果: " + partialResult);
	                    
	                    // 到达屏障，等待其他工作者
	                    barrier.await();
	                    
	                    // 第二阶段：所有线程继续执行其他工作（如果需要）
	                    System.out.println("Worker-" + workerId + " 开始第二阶段工作");
	                    // ... 可以继续使用barrier.await()进行更多阶段的同步
	                    
	                } catch (Exception e) {
	                    e.printStackTrace();
	                }
	            });
	        }
	        executor.shutdown();
	    }
	}
	```
	-  **异常处理**：`barrier.await()`会抛出 `InterruptedException`（线程中断）和 `BrokenBarrierException`（屏障被破坏）。
		- `BrokenBarrierException`的原因：
			- 一个等待中的线程被中断，所有其他等待中的线程都会收到 `BrokenBarrierException`，屏障会被置为“损坏”状态
			- 通过 `reset()`方法可以强制重置一个屏障，所有当前正在屏障处等待的线程立即抛出 `BrokenBarrierException`
	-  **设置超时**：可以使用 `await(long timeout, TimeUnit unit)`方法避免无限期等待。如果超时，当前线程会抛出 `TimeoutException`，同时屏障也会被标记为损坏

	-  **重置屏障**：通过 `reset()`方法可以强制重置一个屏障，但此操作会使得所有当前正在屏障处等待的线程立即抛出 `BrokenBarrierException`，需谨慎使用