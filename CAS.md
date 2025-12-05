- 含义：CompareAndSet，比较和设置两步操作实现原子性
- 实现方式：
	- 在cpu内部保证单核和多核下CAS都具有原子性
- 为什么无锁效率高：主要由于获取锁失败的不同处理策略
	- synchronized：线程进入阻塞状态，锁释放后被唤醒进入锁竞争阶段
		- 好处：主动放弃对cpu时间片的占用
		- 问题：发生至少两次上下文切换
	- cas：线程始终保持运行状态，可以通过自旋循环尝试获取
		- 好处：在低竞争场景下，避免了线程状态切换的开销
		- 问题：在高竞争环境下
			- 如果线程长时间自旋仍不成功，会白白消耗大量CPU资源
			- 而且cpu时间片耗完了，也会进行上下文切换
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
