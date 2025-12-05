- 需求背景
	- 串行任务同步的需求：为了提升效率，有一些任务需要串行化执行
		- 实现串行化执行，在实际开发中一般使用线程池技术
			- 【线程池技术】线程不会轻易结束
				- 无法使用join同步线程
		- 原先方法的局限性： `wait()/notifyAll()`
			- 需要复杂的同步代码（`synchronized`）和共享对象管理
				- 容易出错
				- 代码可读性差
	- 需要同时起步的情况
- 解决措施：CountDownLatch
	- 提供了一个与具体线程解耦的、基于任务的同步机制
		- 不需要等待任务线程结束才能得到通知
	- 是对`wait()/notifyAll()`的更加简介，线程安全的封装
	- 主线程提供`countDown()`，所有线程同时开始执行任务
- 使用示例：
	```
		import java.util.concurrent.CountDownLatch;
	import java.util.concurrent.TimeUnit;
	
	public class BasicExample {
	    public static void main(String[] args) throws InterruptedException {
	        // 1. 创建一个计数器为3的CountDownLatch
	        CountDownLatch latch = new CountDownLatch(3);
	
	        for (int i = 1; i <= 3; i++) {
	            final int taskId = i;
	            new Thread(() -> {
	                try {
	                    // 模拟任务执行时间
	                    TimeUnit.SECONDS.sleep(taskId);
	                    System.out.println("任务 " + taskId + " 完成");
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                } finally {
	                    // 2. 每个任务完成后，计数器减1 (务必放在finally块中确保执行)
	                    latch.countDown();
	                    System.out.println("当前计数器: " + latch.getCount());
	                }
	            }).start();
	        }
	
	        System.out.println("主线程等待所有任务完成...");
	        // 3. 主线程在此等待，直到计数器减为0
	        latch.await();
	        System.out.println("所有任务已完成，主线程继续执行");
	    }
	}
	```
- 实现原理：基于AQS
	- 将state取计数器的语义
		- 主要依赖 volatile 变量读和 CAS 操作
			- 避免了昂贵的锁竞争，在高并发场景下性能表现优异
