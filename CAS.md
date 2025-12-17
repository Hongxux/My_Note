- 含义：CompareAndSet，比较和设置两步操作实现原子性
- 实现方式：
	- 在cpu内部保证单核和多核下CAS都具有原子性
	- 
- 为什么无锁效率高：主要由于获取锁失败的不同处理策略
	- synchronized：线程进入阻塞状态，锁释放后被唤醒进入锁竞争阶段
		- 好处：主动放弃对cpu时间片的占用
		- 问题：发生至少两次上下文切换
	- cas：线程始终保持运行状态，可以通过自旋循环尝试获取
		- 好处：在低竞争场景下，避免了线程状态切换的开销
		- 问题：在高竞争环境下
			- 如果线程长时间自旋仍不成功，会白白消耗大量CPU资源
			- 而且cpu时间片耗完了，也会进行上下文切换
- 问题：
	- ABA问题
		- 含义：CAS操作只检查“值”是否变化，而无法感知“状态”的变化
		- 解决措施：引入“版本号”的概念
	- 线程通过循环不断重试CAS操作（自旋），在竞争激烈时大量消耗CPU资源，却无法完成有效工作
		- 解决措施：
			- 限制自旋次数，到达阈值后转化为悲观锁
			- 退避策略：在CAS失败后，不立即重试，而是让线程等待一个随机的时间（指数退避），这有助于降低冲突的概率。
- 适用场景：线程数少、多核 CPU

- 使用方式：![[Pasted image 20251202104722.png]]
	1. 获取当前值作为预期值
	2. 计算得到新的值
	3. 进行CAS操作（以下三步都是原子性的）
		1. 比较当前值和内存中的值
		2. 如果相等，则交换内存中的值和新的值，返回true
		3. 如果不相等，则不交换，返回false
![[Pasted image 20251119110214.png]]\
![[Pasted image 20251119110226.png]]


Java的 `java.util.concurrent.atomic`包下提供了一系列原子类（如 `AtomicInteger`, `AtomicLong`），它们内部就是通过CAS来实现乐观锁的
```
import java.util.concurrent.atomic.AtomicInteger;

public class Counter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void safeIncrement() {
        int current;
        int next;
        do {
            current = count.get(); // 获取当前值作为预期原值 (A)
            next = current + 1;   // 计算新值 (B)
            // 尝试CAS更新。如果current此时仍等于内存中的值，则更新为next。
            // 如果不相等，说明被其他线程修改过，则循环重试。
        } while (!count.compareAndSet(current, next)); 
    }
    
    public int getCount() {
        return count.get();
    }
}
```
