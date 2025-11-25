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
