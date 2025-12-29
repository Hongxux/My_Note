# 互联网大厂Java并发编程面试考察提纲（自我检查+应答核心）

## 1. 基础概念与线程基础

### 考察核心：理解而非死记，结合实际场景

1. 并发 vs 并行

    - 核心应答：并发是“同一时间段多任务交替执行”（单核CPU），并行是“同一时刻多任务同时执行”（多核CPU）；举例：并发=一人处理多份文件，并行=多人同时处理文件

    - 高频追问：Java中如何体现并发/并行？（线程调度+CPU核心数支持）
	    1. **核心载体**：Java 中并发的基础是`Thread`，并行依赖多核 CPU 和对线程的合理调度（如线程池、并行流）；
	    2. **实现方式**：线程池是生产环境中高效实现并发的主流方式，并行流是便捷的并行计算方式；
	    3. **正确性保障**：`synchronized`等同步机制解决并发下的原子性、可见性问题，是并发程序正确运行的关键（没有同步的并发会导致数据错乱）。

2. [[进程]] vs [[线程]]
    - 核心应答：
	    - 进程是资源分配单位（独立内存空间）
		    - 能实现资源隔离带来的故障隔离和安全隔离，从而提高系统稳定性，但是进程上下文切换开销大，且进程间通信复杂
	    - 线程是调度执行单位（共享进程资源）；
		    - 线程切换开销远小于进程，且线程间共享数据简单，但是要处理并发安全问题
    - 高频追问：Java线程与操作系统线程的关系？
	    1. **核心映射**：主流 HotSpot VM 采用 1:1 模型，1 个 Java 线程对应 1 个操作系统内核线程，Java 线程的本质是 OS 内核线程的上层封装；
		2. **调度权**：Java 线程的调度由操作系统决定，JVM 仅能 “建议” 调度（如`yield()`），无直接控制权；
		3. **开销与优化**：Java 线程的资源开销等同于 OS 线程，因此生产环境需用线程池复用线程，避免频繁创建 / 销毁。
3. 线程生命周期

    - 核心应答：6个状态及转换条件
	    - NEW→RUNNABLE：start()；
	    - RUNNABLE→BLOCKED：锁竞争；
	    - RUNNABLE→WAITING：wait()/join()；
	    - RUNNABLE→TIMED_WAITING：sleep()/wait(long)；
	    - WAITING/TIMED_WAITING→RUNNABLE：notify()/interrupt()；
	    - 所有状态→TERMINATED：执行完毕）

    - 关键细节：RUNNABLE包含“就绪”（等待CPU调度）和“运行中”（占用CPU）；IO阻塞时线程仍处于RUNNABLE（Java不区分IO阻塞状态，由OS管理）

    - 高频追问：sleep()和wait()的区别？（是否释放锁、是否需要notify、所属类）
	    - sleep不释放锁；wait释放锁
	    - sleep不需要notify；wait需要notify

## 2. Java内存模型 (JMM)

### 考察核心：底层原理+实际意义，而非仅记规则

1. 主内存与[[工作内存]]

    - 核心应答：主内存存储共享变量（所有线程可见），工作内存是线程私有副本
	    - 工作内存是对**CPU 缓存 + 寄存器** 的抽象

    - 关键逻辑：线程间通信需通过“工作内存→主内存→工作内存”，JMM定义该过程的规则保证可见性、原子性、有序性

2. Happens-before 原则

    - 核心应答：定义多线程操作的可见性规则，无需同步的前提下，满足Happens-before则操作结果对其他线程可见

    - 核心规则（必背）：程序顺序规则、volatile变量规则、锁规则、线程启动/join规则、传递性

    - 高频追问：为什么需要Happens-before？（避免底层指令重排导致的逻辑错误，简化并发编程）

3. 指令重排与内存屏障

    - 核心应答：指令重排是编译器/CPU为优化性能的乱序执行，可能破坏多线程有序性；内存屏障（LoadLoad/StoreStore/LoadStore/StoreLoad）禁止特定重排，保证可见性和有序性

    - 关联考点：volatile如何通过内存屏障实现禁止重排？（写后加StoreLoad屏障，读后加LoadLoad+LoadStore屏障）

## 3. synchronized 关键字

### 考察核心：底层实现+锁升级+性能优化，是高频考点

1. 实现基础

    - 核心应答：同步方法依赖方法区的ACC_SYNCHRONIZED标记，同步代码块依赖monitorenter/monitorexit字节码；底层依赖对象监视器（Monitor），Monitor与对象头Mark Word关联

    - Mark Word结构：存储对象哈希码、分代年龄、锁状态（无锁/偏向锁/轻量级锁/重量级锁），不同锁状态下存储内容不同（如偏向锁存储线程ID，轻量级锁存储锁记录指针）

2. 锁升级过程（无锁→偏向锁→轻量级锁→重量级锁）

    - 无锁：对象刚创建，无线程竞争，Mark Word存储哈希码

    - 偏向锁：单线程重复获取锁时，避免CAS操作（节省CAS开销）；通过Mark Word记录线程ID，后续线程获取时直接对比ID，无需竞争

        - 锁撤销/升级触发条件：有其他线程竞争时，撤销偏向锁（批量重偏向/批量撤销）

        - 批量重偏向：同一锁被多个线程交替持有，避免频繁撤销/升级（节省锁切换开销）；触发条件：偏向锁撤销次数达到阈值

        - 批量撤销：锁被多个线程同时竞争，直接撤销偏向锁，后续默认使用轻量级锁（假设后续仍有竞争）

    - 轻量级锁：多线程交替竞争，无长时间阻塞（节省Monitor切换的内核态开销）；通过CAS将锁记录指针存入Mark Word，竞争失败则自旋重试

        - 不升级为重量级锁的情况：自旋次数未达阈值（默认10次），或CPU核心数充足

    - 重量级锁：多线程长时间竞争，自旋失效；依赖OS互斥量（Mutex），线程阻塞进入内核态（开销最大）

3. 高频追问

    - synchronized与ReentrantLock的区别？（公平性、可中断、超时获取、Condition、底层实现）

    - 偏向锁为什么能提升性能？（避免无竞争场景下的CAS操作和Monitor调用）

    - 轻量级锁的自旋机制是什么？（忙等，避免线程阻塞切换开销）

## 4. Lock 体系与 AQS

### 考察核心：AQS是基础，Lock子类需讲清与AQS的关联+自身特性

1. AQS（AbstractQueuedSynchronizer）

    - 需求背景与定位：解决并发工具“线程排队、阻塞唤醒、状态同步”的重复开发问题，提供通用框架

        - 框架职责：封装CLH队列（双向链表）、CAS操作、阻塞/唤醒（LockSupport）、中断处理等共性逻辑

        - 使用者职责：定义state变量含义、实现钩子方法（tryAcquire/tryRelease等）

    - 核心抽象：

        - state变量：volatile int类型，存储同步状态（如锁重入次数、许可数），通过CAS保证原子修改（compareAndSetState）

        - CLH队列：双向链表选型原因（支持快速入队、出队、删除取消节点，O(1)操作），用于存储获取资源失败的线程

        - Node节点：核心属性（thread、prev/next、waitStatus、nextWaiter）；waitStatus含义（SIGNAL=-1：需唤醒后继；CANCELLED=1：节点取消；PROPAGATE=-3：共享模式唤醒传播；0：初始状态；CONDITION=-2：条件队列）

        - 钩子方法：独占（tryAcquire/tryRelease）、共享（tryAcquireShared/tryReleaseShared）、isHeldExclusively，由子类实现专属逻辑

    - 核心方法闭环：

        - 独占模式：

            - acquire（获取资源）：tryAcquire→失败→addWaiter（CAS快速入队，失败则enq循环入队）→acquireQueued（循环检查前驱是否为头节点，是则重试获取，否则通过shouldParkAfterFailedAcquire确保前驱为SIGNAL后阻塞）

                - 中断处理：acquire忽略中断（仅记录），acquireInterruptibly直接抛异常

                - 高效阻塞：通过LockSupport.park()阻塞，唤醒后重新竞争资源

            - release（释放资源）：tryRelease→成功→unparkSuccessor（从队尾向前找有效后继，避免next引用失效；判断是否唤醒：头节点waitStatus≠0则唤醒）

        - 共享模式：

            - acquireShared：tryAcquireShared（返回≥0成功）→失败→doAcquireShared（共享节点入队，阻塞逻辑复用）→唤醒后触发setHeadAndPropagate（传播唤醒）

            - releaseShared：tryReleaseShared→成功→doReleaseShared（循环CAS保证唤醒原子性，处理头节点SIGNAL/0/PROPAGATE三种场景，实现链式传播）

    - 基于AQS的实现差异：

        - ReentrantLock：独占模式，state=重入次数，实现tryAcquire（公平/非公平）、tryRelease

        - ReentrantReadWriteLock：读写分离，state高16位读计数、低16位写计数，读锁共享、写锁独占

        - Semaphore：共享模式，state=许可数，实现tryAcquireShared/tryReleaseShared

        - StampedLock：非AQS子类，复用CLH队列/CAS/阻塞逻辑，state为64位（版本戳+锁状态）

        - CountDownLatch：共享模式，state=计数，一次性唤醒

        - CyclicBarrier：间接基于AQS（ReentrantLock+Condition），可循环复用

2. ReentrantLock

    - 核心特性：独占、可重入、支持公平/非公平、可中断、超时获取、Condition

    - 高频追问：默认非公平的原因？（减少线程切换，提升吞吐量）；可重入实现？（tryAcquire中判断当前线程是否为独占线程，是则state+1）

3. ReentrantReadWriteLock

    - 核心特性：读共享、写独占、可重入、锁降级（写→读）

    - 高频追问：读锁为何不支持Condition？（Condition依赖独占锁，共享语义冲突）；与StampedLock的区别？（StampedLock支持乐观读，无可重入，性能更高）

4. StampedLock

    - 核心特性：乐观读、悲观读、写锁，无可重入，性能优于读写锁

    - 核心实现：乐观读无锁（获取版本戳→操作→验戳），悲观读/写锁入队阻塞

    - 高频追问：乐观读的适用场景？（读多写极少，读操作耗时短）；ABA问题？（版本戳递增，避免ABA）

5. Condition接口

    - 核心特性：绑定独占锁，支持多条件队列、可中断、超时等待

    - 底层实现：AQS的ConditionObject，维护独立条件队列，await()释放锁→入条件队列，signal()转移到同步队列

    - 高频追问：与Object.wait()/notify()的区别？（多条件队列、精准唤醒、支持中断/超时）

## 5. volatile 关键字

### 考察核心：三大特性+底层实现+适用场景

1. 核心特性：

    - 可见性：写操作后通过内存屏障强制刷新到主内存，读操作前强制从主内存加载

    - 禁止指令重排：通过内存屏障（写后StoreLoad，读后LoadLoad+LoadStore）禁止特定重排

    - 不保证原子性：仅修饰单个变量，复合操作（i++）仍需锁或原子类

2. 适用场景：状态标志位（如boolean flag）、双重检查锁的instance修饰

3. 高频追问：volatile如何禁止重排？（内存屏障）；volatile与synchronized的区别？（原子性、开销、适用场景）

## 6. CAS 与原子类

### 考察核心：底层实现+问题解决+原子类原理

1. CAS：

    - 核心定义：Compare And Swap，比较内存值与预期值，相等则更新为新值，原子操作（依赖CPU的cmpxchg指令）

    - ABA问题：内存值从A→B→A，CAS误判为未修改；解决方案：AtomicStampedReference（添加版本戳）

    - 高频追问：CAS的缺点？（自旋开销、ABA问题、只能修饰单个变量）

2. 原子类：

    - AtomicInteger/AtomicLong：基于CAS实现自增（getAndIncrement）、自减等操作

    - 其他原子类：AtomicReference（引用类型）、AtomicStampedReference（解决ABA）、LongAdder（高并发下性能优于AtomicLong，分段累加）

    - 高频追问：LongAdder为什么高性能？（减少CAS竞争，分段存储累加值）

## 7. 线程池

### 考察核心：ThreadPoolExecutor是重点，参数配置+流程+生命周期+风险

1. ThreadPoolExecutor

    - 核心参数（7个）：核心线程数、最大线程数、空闲线程存活时间、时间单位、工作队列、线程工厂、拒绝策略

        - 参数配置：CPU密集型（核心线程数=CPU核心数+1）、IO密集型（核心线程数=2*CPU核心数）；最大线程数失效场景（工作队列是无界队列，如LinkedBlockingQueue）；工作队列选择（有界队列避免OOM，无界队列适合任务量稳定）

    - 工作流程：提交任务→核心线程未满则创建核心线程→核心线程满则入工作队列→队列满则创建非核心线程→线程数达最大则执行拒绝策略

    - 生命周期：RUNNING（接收+执行任务）→SHUTDOWN（不接收新任务，执行队列任务）→STOP（不接收新任务，中断正在执行任务，清空队列）→TIDYING（所有任务执行完毕，线程数为0，执行terminated()）→TERMINATED（终止）

        - TIDYING意义：统一资源清理（如关闭连接、释放内存）

    - 拒绝策略：AbortPolicy（抛异常，默认）、CallerRunsPolicy（调用者执行）、DiscardPolicy（丢弃任务）、DiscardOldestPolicy（丢弃队列头任务）

2. 常见线程池：

    - FixedThreadPool：核心线程数=最大线程数，无界队列；风险：任务过多导致OOM；适用场景：任务量稳定、CPU密集型

    - CachedThreadPool：核心线程数=0，最大线程数=Integer.MAX_VALUE，同步队列；风险：线程数过多导致OOM；适用场景：短期、轻量级任务

    - ScheduledThreadPool：

        - 痛点：
	        - 解决普通线程池的痛点：解决定时/延迟任务的精确执行（普通线程池无法精准控制延迟）
	        - 解决Timer定时器的痛点：单线程
		        - 耗时任务会卡住整个调度队列
		        - 出现未捕获异常整个队列崩溃
		        - 调度依赖系统时间

        - 实现：基于DelayedWorkQueue（延迟队列），任务按延迟时间排序

        - 智能延迟策略：避免忙等，通过LockSupport.parkNanos()精准阻塞，保障定时精度；解决痛点：避免线程空循环消耗CPU，保证任务按时执行

3. 高频追问：

    - 线程池为什么要使用工作队列？（缓冲任务，减少线程创建销毁开销）

    - 如何避免线程池OOM？（使用有界队列，合理设置最大线程数）

    - 核心线程为什么不会被回收？（核心线程的空闲时间无限制）

## 8. 并发工具类（CountDownLatch、CyclicBarrier、Semaphore）

### 考察核心：功能+实现+区别+使用场景

1. 核心功能：

    - CountDownLatch：倒计时门闩，一次性，一组线程等另一组线程完成（如主线程等子线程）

    - CyclicBarrier：同步屏障，可循环，一组线程互相等待全部到达后一起执行（如多线程分阶段任务）

    - Semaphore：信号量，控制并发线程数（如限流、资源池控制）

2. 核心区别：

    - 复用性：CountDownLatch一次性，CyclicBarrier可循环

    - 触发条件：CountDownLatch是“计数为0”，CyclicBarrier是“所有线程到达”

    - 底层实现：CountDownLatch直接基于AQS，CyclicBarrier间接基于AQS（Lock+Condition）

3. 适用场景：

    - CountDownLatch：启动流程中等待所有组件初始化完成

    - CyclicBarrier：多线程并行计算，最后汇总结果

    - Semaphore：接口限流、数据库连接池控制并发连接数

## 9. 并发容器

### 考察核心：ConcurrentHashMap是重点，底层实现+特性+坑

1. ConcurrentHashMap（JDK1.8）

    - 底层实现：数组+链表/红黑树， synchronized+CAS保证并发安全（替代1.7的分段锁）

    - 核心特性：

        - 弱一致性：读操作不加锁（弱一致性读），迭代器不抛出ConcurrentModificationException（弱一致性迭代器）

        - size不准确：因为并发修改，size()通过估算得出，非实时准确

        - 协作扩容：多线程参与扩容，提高扩容效率；流程：发现数组满→触发扩容→创建新数组→线程拆分迁移数据→迁移完成后替换旧数组

        - 树化/反树化：链表长度≥8且数组长度≥64则树化（提升查询效率）；树节点数≤6则反树化（节省空间）

    - 高频追问：为什么用synchronized而非ReentrantLock？（JDK1.8中synchronized优化后性能接近，且节省内存）；与HashMap的区别？（并发安全、弱一致性）

2. CopyOnWriteArrayList

    - 核心实现：读写分离，写操作复制原数组（add/set等），读操作直接读原数组

    - 特性：读无锁、线程安全、弱一致性；适用场景：读多写少（如配置缓存）

    - 风险：写操作开销大（复制数组），内存占用高

3. BlockingQueue

    - 核心实现：ArrayBlockingQueue（有界，数组）、LinkedBlockingQueue（可选无界，链表）、SynchronousQueue（无缓冲，直接传递）、DelayQueue（延迟队列）

    - 适用场景：生产者-消费者模型（ArrayBlockingQueue/LinkedBlockingQueue）、线程池任务传递（SynchronousQueue）

## 10. 经典问题与实践

### 考察核心：代码实现+原理+问题排查

1. 生产者-消费者模型

    - 实现方式：

        - wait()/notify()：需同步锁，注意wait()在循环中判断条件（避免虚假唤醒）

        - BlockingQueue：无需手动处理锁和唤醒，简化代码（如ArrayBlockingQueue.put()/take()）

    - 高频追问：为什么wait()要在循环中？（避免虚假唤醒，重新检查条件）

2. 死锁

    - 产生条件：资源互斥、持有并等待、不可剥夺、循环等待

    - 诊断工具：jstack（查看线程堆栈，识别死锁线程和持有的锁）

    - 避免方法：顺序加锁（按固定顺序获取锁）、尝试锁（tryLock()，超时放弃）、定时锁（lockInterruptibly()）

## 11. ThreadLocal

### 考察核心：底层实现+内存泄漏+使用场景

1. 核心功能：线程本地存储，变量在每个线程有独立副本，线程隔离

2. 底层实现：

    - 每个Thread持有ThreadLocalMap对象，key是ThreadLocal（弱引用），value是线程私有副本

    - ThreadLocalMap：数组+Entry（key为弱引用），解决哈希冲突（开放地址法）

3. 内存泄漏风险：

    - 原因：ThreadLocal被回收（弱引用），但ThreadLocalMap的Entry仍持有value，且Thread未结束→value无法回收

    - 解决方案：使用后调用remove()方法（删除Entry，释放value）

4. 适用场景：Spring事务管理（存储连接）、日志MDC（存储线程上下文信息）、避免方法参数传递

5. 高频追问：ThreadLocal的key为什么是弱引用？（避免ThreadLocal未回收导致的内存泄漏）；与Synchonized的区别？（ThreadLocal是线程隔离，Synchronized是线程共享互斥）

## 自我检查通用标准

1. 每个知识点能“一句话定义+底层实现+使用场景+1个高频问题答案”；

2. 能清晰区分相似技术（如CountDownLatch vs CyclicBarrier、ReentrantLock vs synchronized）；

3. 能结合业务场景给出技术选型（如限流用Semaphore、读多写少用ReentrantReadWriteLock）；

4. 能说出核心风险与解决方案（如ThreadLocal内存泄漏→remove()、线程池OOM→有界队列）。
> （注：文档部分内容可能由 AI 生成）