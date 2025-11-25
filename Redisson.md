![[Pasted image 20251119152043.png]]
### Redisson的配置
#### 1.引入依赖
![[Pasted image 20251120132401.png]]
#### 2.配置RedissonConfig连接
![[Pasted image 20251120134635.png]]
#### 3.使用RLock的模板
```
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

@Service
public class RobustLockService {

    private final RedissonClient redissonClient;

    public RobustLockService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    public void performTaskWithLock(String resourceKey) {
        // 1. 获取锁实例
        RLock lock = redissonClient.getLock("robust-lock:" + resourceKey);
        
        boolean acquired = false;
        try {
            // 2. 尝试加锁（非阻塞式，包含等待超时和锁超时）
            // 参数：最大等待时间，锁自动释放时间，时间单位
            acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
            
            if (!acquired) {
                // 3. 处理获取锁失败的情况
                throw new TimeoutException("获取分布式锁超时，请稍后重试");
            }
            
            // 4. 成功获取锁后，二次确认（可选，但更安全）
            if (lock.isHeldByCurrentThread()) {
                // 5. 在这里执行业务逻辑
                System.out.println("成功获取锁，执行关键业务代码...");
                // ... your business logic here
            }
            
        } catch (InterruptedException e) {
            // 处理线程中断
            Thread.currentThread().interrupt(); // 恢复中断状态
            System.err.println("线程在等待锁时被中断");
        } catch (TimeoutException e) {
            // 处理获取锁超时
            System.err.println(e.getMessage());
        } catch (Exception e) {
            // 处理其他业务逻辑异常
            System.err.println("业务执行出现异常: " + e.getMessage());
        } finally {
            // 6. 【关键】在finally块中安全释放锁
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
                System.out.println("锁已安全释放");
            }
        }
    }
}
```
##### **合理配置锁超时**

- **对于执行时间不确定的长任务**：使用 `lock()`或 `tryLock(... -1, ...)`，依赖看门狗自动续期，避免业务未完成锁超时。
    
- **对于执行时间可预估的短任务**：明确设置一个合理的 `leaseTime`（略大于平均执行时间），这样可以减轻看门狗负担，系统更简洁
##### finnaly块处解锁
**务必在 finally 块中解锁**：这是使用任何锁都必须遵守的“铁律”。为确保万无一失，在调用 `unlock()`前，最好使用 `lock.isHeldByCurrentThread()`方法进行校验，避免非锁持有者尝试解锁而抛出 `IllegalMonitorStateException`

### RLock的特性
|特性分类|核心机制|价值与优势|
|---|---|---|
|**可重入性**​|通过 Redis Hash 结构记录线程唯一标识和重入次数<br><br>。|同一线程可多次获取锁，避免死锁，简化嵌套调用编程模型<br><br>。|
|**自动续期与看门狗**​|未指定锁超时时间时，看门狗机制会后台定时检查并续期，防止业务未完成锁超时<br><br>。|提升锁的安全性，减少因业务执行时间不确定导致的锁意外释放风险<br><br>。|
|**多样化的锁类型**​|支持公平锁、非公平锁、联锁 (MultiLock)、红锁 (RedLock) 等<br><br>。|适应不同业务场景对公平性、可靠性的差异化需求<br><br>。|
|**尝试获取与超时**​|提供 `tryLock`方法，可设置最大等待时间，避免线程无限期阻塞<br><br>。|提高系统响应性和韧性，在无法获取锁时执行降级策略。|
|**原子性操作**​|核心的加锁、解锁逻辑通过 Lua 脚本在 Redis 端原子性执行|
### Redisson的原理
![[Pasted image 20251119201112.png]]
#### [[基于Redis实现分布式锁]]
#### Redisson[[可重入锁]]的原理
#### Redisson可[[重试获取锁]]
#### Redisson的[[看门狗默认时间]]
#### 解决主从不一致问题
##### 主从不一致发生的原因
![[Pasted image 20251119201614.png]]
在主从同步设置锁的时候，主节点发生宕机，而新的主节点还没有保存锁的相关信息，从而出现并发安全问题


解决方案：放弃一主多重，改成多主多从，设置锁向多个主节点，获取锁也是去多个主节点，这称为[[联锁]]（multilock）
- **将多个独立的锁（RLock）逻辑上捆绑在一起，形成一个统一的锁**，从而实现对多个资源进行原子性的加锁与解锁操作
- `MultiLock`的核心是提供“全有或全无”的加锁语义，这对于需要强一致性的操作至关重要。