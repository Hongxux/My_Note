- 需求背景：JMM 的核心矛盾
	- 既要允许编译器 / CPU 做指令重排序、工作内存缓存等优化（提升性能），又要保证多线程程序的**行为可预测**。
- 解决措施：happens-before 原则
	- 本质是给多线程程序的 “内存可见性 + 有序性” 制定一套 “最小约束规则”：
		- 对**虚拟机 / 编译器**：允许做任何优化（重排序、缓存），但**不能违反 happens-before 规则**（即优化后的执行结果，必须和遵守规则的执行结果一致）；
		- 对**程序员**：无需关心底层的缓存、重排序细节，只要保证代码符合 happens-before 规则，就能确保多线程下的可见性和有序性，且代码在所有平台上行为一致。
	- 通俗类比：happens-before 就像交通规则 —— 交警（虚拟机）允许司机（CPU / 编译器）灵活驾驶（比如超车、变道），但必须遵守 “红灯停、绿灯行” 等核心规则；司机不用管每条路的具体路况（底层差异），只要遵守规则，就能保证安全（程序正确）。
- 含义：如果操作 A happens-before 操作 B，那么 A 操作对内存的修改，对 B 操作是 “可见且有序” 的。
	- 可见：A 修改的变量，B 能读到最新值；
	- 有序： A 的执行顺序在 B 之前
- 保证happens-before实现的规则
	- 同一线程：在**同一个线程**中，按照代码顺序，前面的操作 happens-before 于后续的任何操作
		```
			public void example() {
		    int x = 10;     // 操作A
		    int y = x + 1;  // 操作B
		}
		```
			- 
	- **监视器锁规则**：对一个锁的**解锁**操作 happens-before 于后续对**同一个锁**的加锁操作。
		- 解锁时，线程的工作内存会同步到主内存；加锁时，线程会从主内存加载最新值。
		```
		public class LockExample {
		    private int count = 0;
		    private final Object lock = new Object();
		
		    public void increment() {
		        synchronized (lock) { // 加锁
		            count++;
		        } // 解锁（操作A）
		    }
		
		    public int getCount() {
		        synchronized (lock) { // 加锁（操作B）
		            return count;
		        }
		    }
		}
		```
	- **volatile**：对一个 `volatile`变量的**写**操作 happens-before 于后续对这个变量的**读**操作
		```
		public class VolatileExample {
		    private volatile boolean flag = false;
		    private int data = 0;
		
		    public void writer() {
		        data = 42;      // 操作A
		        flag = true;    // 操作B: volatile写
		    }
		
		    public void reader() {
		        if (flag) {     // 操作C: volatile读
		            System.out.println(data); // 保证输出 42
		        }
		    }
		}
		```
	- 对变量的默认值(0，false，null)的写，对其它线程对该变量的读可见
		- 避免读取到完全随机、不确定的值
		- 对变量的默认值(0，false，null)的写的含义：发生在一个线程创建对象时
			1. **分配内存**：JVM 为对象分配一块内存空间。
			2. **设置默认值（本条规则生效）**：JVM 会将这块内存中所有字段设置为默认值。此时，`value`是 `0`，`name`是 `null`。**这个操作的结果对其他线程是立即可见的**。
			3. **执行构造函数**：然后才执行构造函数中的代码（包括实例变量的显式初始化），`value`被赋值为 `42`，`name`被赋值为 `"Java"`。
	- **线程启动规则**：对共享变量的写，对线程开始后（start）该变量的读可见
		```
		static int config = 0;
		
		public static void main(String[] args) {
			config = 100; // 操作A
			Thread t = new Thread(() -> {
				System.out.println(config); // 保证输出 100（操作B）
			});
			t.start(); // 操作C
		}
		```
	- **线程终止规则**：线程结束前的写，对其他得知它结束的线程的读可见的
		- 得知的方式：isAlive()和join()
		    ```
		    static int result;
		    
		    public static void main(String[] args) throws InterruptedException {
		        Thread worker = new Thread(() -> {
		            result = 42; // 操作A
		        });
		        worker.start(); // 操作B
		        worker.join();  // 操作C：主线程等待worker线程终止
		        System.out.println(result); // 保证输出 42（操作D）
		    }
		    ```
	- **线程打断规则**：线程 t1 打断 t2(imnterupt)前对变量的写，对于其他线程得知 t2 被打断后对变量的读可见
		- 得知被打断的方式：t2.interruptedt和2.isInterrupted
	- **传递性规则**:如果操作 A happens-before 操作 B，操作 B happens-before 操作 C，那么可以推导出操作 A happens-before 操作 C。
		示例：在之前的 `VolatileExample`中：
		
		- 根据程序次序规则：`data = 42`(A) happens-before `flag = true`(B)。
		    
		- 根据 volatile 变量规则：`flag = true`(B) happens-before `if(flag)`(C)。
		    
		- 因此，根据传递性规则：`data = 42`(A) happens-before `if(flag)`(C)，进而 happens-before `System.out.println(data)`。这就保证了线程B能看到正确的 `data`值