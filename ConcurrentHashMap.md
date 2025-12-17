- HashMap和Hashtable的局限性
	- HashMap：未考虑线程安全
		- 出现数据覆盖，导致数据丢失
			- 起因：
				- 两个线程的put操作指向同一个桶
				- 该位置没有元素，两个线程都认为是空的
			- 行为：各自将新的键值对插入
			- 后果：只有一个线程的写入会生效，另一个线程的写入会被覆盖
				- 由于数据覆盖，导致数据丢失
		- 产生死链，get的时候死循环（JDK7以下才会出现）
			- 发生时机：哈希扩容的时候
			- 现象：
				- HashMap 的内部链表结构会发生变化
				- 多线程环境下并发扩容，可能导致链表形成环形结构
			- 后果：当后续有线程调用 `get`方法遍历这个链表时，就会陷入死循环，导致CPU占用100%
		- 存储元素的个数不准确
			- 原因：size属性的更新也不是原子的
			- 后果：多个线程同时增加和减少 `size`，可能导致最终计算出的 Map 大小与实际元素数量不符
	- Hashtable：线程安全但是并发度低，性能低
		- 锁粒度设计不合理，使用全局锁实现线程安全
			- 【全局锁】的使用方式通过在方法上添加 `synchronized`关键字实现
			- 后果：即使线程操作不同的键（数据上没有竞争），也只有一个线程能进行读写操作
				- 实际上从并发编变成了串行，吞吐量低
- 解决方法： ConcurrentHashMap
	- 设计思想：降低锁的粒度，让不存在数据竞争的操作能够并行进行
	- 实现并发安全的方式：
		- 读写并发：无锁读
			- 表现：所有读操作（`get`）几乎都是无锁的
			- 原因：
				- Node的 value字段用 volatile修饰，保证了可见性
					- 当一个线程修改了某个节点的 `value`或整个链表的结构（比如设置 `next`），这个修改会立即对其他线程可见​
				- 整个哈希桶数组的 `table`引用本身也是 `volatile`的
					- 确保了数组扩容后，所有线程都能看到新的数组引用（ForwardingNode）
		- 写写并发：使用乐观锁或者细粒度锁
			- 没有哈希冲突：使用CAS​ 这种无锁操作来更新。加锁失败重新判断桶状态
			- 发生哈希冲突：只对那个桶的头节点加 `synchronized`锁
				- 锁的粒度从“段”缩小到了“单个桶”，并发度得到了极大提升
		- 扩容的时候读和写：
			- 读安全：扩容完一个桶，将旧桶头节点替换为ForwardingNode，让相关查询指向新表
			- 写安全：
				- 线程开始迁移某个特定桶时，使用 `synchronized`关键字锁住这个桶的头节点
				- 当写操作（如 `put`, `remove`）访问到 `ForwardingNode`节点时，先协助完成数据扩容，再写
	- 协助扩容，并行加速扩容
		- 用sizeCtl标识扩容的状态和活跃的协助扩容
		- 用ForwardingNode标记
			- 标识着扩容的进行，标识则该桶迁移的完成
			- 对于写线程，看到后先协助扩容再插入数据
			- 对于读线程，看到后去新表进行查询
	- ConcurrentHashMap是[[弱一致性]]：
		- 设计目的：在**保证线程安全的同时，尽可能地提升并发性能**
			-
		- 表现
			- 迭代器 (`Iterator`)：
				- 设计目的： 如果追求强一致性迭代器，就必须在创建迭代器的瞬间锁定整个容器，或者在迭代过程中阻止所有写操作。这会导致严重的性能下降和线程阻塞。
				- 弱一致性现象：
					- 可能看到迭代期间的部分更新，但非全部；
					- 可能看到已删除的元素。
				- 原因：
					- 当您获取一个 `ConcurrentHashMap`的迭代器时，它是直接遍历当前的 `table`（Node 数组）。
						- 含义：它从数组索引 0 开始，依次访问每个桶（bucket），然后遍历桶中的链表或红黑树。
						- 不像 `CopyOnWriteArrayList`那样先复制整个底层数组。
					- 扩容带来的“时空错乱”:
						- 迭代器可能已经遍历过了旧数组的索引 `i`，即使这个桶的数据已经被迁移到了新数组，迭代器也**不会回头**再去新数组读取数据，它认为索引 `i`的部分已经处理完毕，因此错过了这次修改
						- 反之，如果迭代器还没有遍历到索引 `i`，它会发现桶里是一个 `ForwardingNode`，于是就会**跳转到新数组对应的位置去继续遍历**，这样它就能看到最新的数据
			- size()/ mappingCount()：返回的是一个近似值。
				- 设计目的：如果要获得一个精确的 `size()`，就需要在统计的瞬间“冻结”整个Map的状态，确保在统计过程中没有任何线程在进行插入或删除操作。
				- 原因：借鉴了 `LongAdder`的**分而治之**思想
					- `size()`方法调用 `sumCount()`，**无锁地**遍历整个CounterCell数组，将 `baseCount`和所有 `CounterCell.value`累加起来
				- 解决措施：如何获得精确值
					1. **外部计数**：使用 `AtomicLong`或 `LongAdder`在业务层手动维护计数器。
					2. **全量扫描**：在低峰期，通过加锁或暂停写服务的方式遍历整个Map（性能代价大）
			- get()操作：可能短暂地读不到最新写入的值。
			- containsValue()等全表扫描：结果可能不反映方法执行期间映射的真实状态。

	- JDK7和JDK8的区别
		- 加锁方式
			- JDK7：采用分段锁(Segment)机制，将哈希表分为多个Segment（默认16个），每个Segment独立加锁
			- JDK8：放弃分段锁，改用CAS + synchronized组合，锁粒度细化到单个桶级别
		- 数据结构：JDK8引入**红黑树**优化长链表查询（链表长度≥8且数组长度≥64时转换）
		- 扩容机制：JDK8支持**多线程协同扩容**，效率远高于JDK7的单线程扩容

- 使用示例：
	- 构造函数：可以指定初始容量，扩容的阈值，并发度
		- 设置合理的初始容量：在初期避免扩容
		- 如果初始容量小于并发度，则会将初始容量改为并发度
		- 哈希table是懒惰初始化
			- 【懒惰初始化】在初始化的时候，只计算初始化后的哈希表长，不进行真正的创建哈希表
			- 初始化后的哈希表长必然是2^n：计算哈希表长后找到第一个大于等于2^n作为实际表长
	- put和putIfAbsent
		- 如果存在
			- put覆盖旧值
			- put不覆盖旧值，放弃更新
		- 返回值
			- 为null，此次操作向映射中成功添加了一个新的键值对
			- 不为null，返回的是key 对应的旧值
	- ConcurrentHashMap 的基础操作
		```
		import java.util.concurrent.ConcurrentHashMap;
		
		public class BasicOperationsDemo {
		    public static void main(String[] args) {
		        // 创建 ConcurrentHashMap 实例
		        ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();
		        
		        // 添加元素
		        scores.put("Alice", 90);
		        scores.put("Bob", 85);
		        scores.put("Charlie", 95);
		        
		        // 获取元素
		        Integer aliceScore = scores.get("Alice");
		        System.out.println("Alice's score: " + aliceScore); // 输出: Alice's score: 90
		        
		        // 删除元素
		        scores.remove("Bob");
		        
		        // 检查是否包含键
		        boolean hasCharlie = scores.containsKey("Charlie");
		        System.out.println("Contains Charlie: " + hasCharlie); // 输出: Contains Charlie: true
		    }
		}	
		```
		- ConcurrentHashMap 不允许使用 null 作为键或值
		- size() 方法返回的是近似值，对于并发环境，`mappingCount()`方法更准确
		- 根据预期数据量合理设置初始容量可以避免频繁扩容
	- ConcurrentHashMap 提供了一系列原子操作方法
		```
		import java.util.concurrent.ConcurrentHashMap;
		
		public class AtomicOperationsDemo {
			public static void main(String[] args) {
				ConcurrentHashMap<String, Integer> inventory = new ConcurrentHashMap<>();
				inventory.put("widget", 10);
				inventory.put("gadget", 5);
				
				// putIfAbsent(): 仅当键不存在时放入值
				Integer existing = inventory.putIfAbsent("widget", 20); // 不会覆盖，因为"widget"已存在
				System.out.println("Previous value for widget: " + existing); // 输出: 10
				
				// computeIfAbsent(): 键不存在时通过函数计算值
				inventory.computeIfAbsent("newProduct", k -> 100);
				System.out.println("New product count: " + inventory.get("newProduct")); // 输出: 100
				
				// computeIfPresent(): 键存在时重新计算值
				inventory.computeIfPresent("widget", (k, v) -> v + 5); // 增加5个库存
				System.out.println("Updated widget count: " + inventory.get("widget")); // 输出: 15
				
				// merge(): 合并值
				inventory.merge("widget", 3, Integer::sum); // 当前值15 + 3 = 18
				System.out.println("After merge - widget: " + inventory.get("widget")); // 输出: 18
				
				// replace(): 替换值
				inventory.replace("gadget", 8);
				System.out.println("Replaced gadget count: " + inventory.get("gadget")); // 输出: 8
			}
		}	  
		```
		- 问题：在keyB和KeyA映射到同一个哈希桶的时候可能会死锁
			```
			 map.computeIfAbsent(keyA, k -> { return map.computeIfAbsent(keyB, k2 -> 1); // 内层嵌套调用 });
			```
			- JDK9中拒绝这样递归更新操作
	- 遍历和批量操作
		```
		import java.util.concurrent.ConcurrentHashMap;
		
		public class IterationDemo {
		    public static void main(String[] args) {
		        ConcurrentHashMap<String, Integer> data = new ConcurrentHashMap<>();
		        for (int i = 0; i < 10; i++) {
		            data.put("key" + i, i);
		        }
		        
		        // 1. 使用 forEach 方法
		        data.forEach((key, value) -> 
		            System.out.println(key + " = " + value));
		        
		        // 2. 使用 entrySet 遍历
		        for (var entry : data.entrySet()) {
		            System.out.println(entry.getKey() + " : " + entry.getValue());
		        }
		        
		        // 3. 并行遍历（对于大数据集更高效）
		        data.forEach(2, (key, value) -> 
		            System.out.println("Processing: " + key));
		        
		        // 4. 搜索操作
		        Integer result = data.search(2, (key, value) -> 
		            value > 5 ? value : null);
		        System.out.println("Search result: " + result);
		    }
		}		
		```
		- 遍历操作不保证能反映创建迭代器后所有的修改


- 实现基础：内部类和属性
	- 属性
		- table： 主哈希表，`volatile`修饰的 `Node`数组
			- `volatile`保证了数组引用的内存可见性，确保任何线程都能看到最新的数组状态。
		- sizeCtl： `volatile int`变量
			- 核心作用：协调初始化和扩容。
			- 不同值有特定含义：
				- 为 **-1**​ 时表示表格正在初始化
					- 采用懒加载，使用的时候才进行初始化
					- 初始化过程中通过 CAS 操作将 `sizeCtl`设置为 -1，
						- 采用[[犹豫模式|Balking]]模式，确保只有一个线程能执行初始化
				- 为 **-N**​ 时表示有 **N-1**​ 个线程正在进行扩容；
					- 通过这个判断当前线程是否为最后一个线程，是否要进行收尾工作
				- 为正数时，通常表示扩容的阈值或初始容量
		- nextTable：
			- 使用时机：哈希扩容
				- 只有在扩容时才不为空
			- 作用：指向新构建的哈希表数组。
		- transferIndex：扩容协调指针，标记下一个需要迁移的桶索引
			- 协作扩容的时候为旧的哈希表表长
			- 支持多线程协同扩容
				- transferIndex分配迁移任务段
					- 每次减去一个预设的步长（16），分发给协作线程，在这个范围为线程协作哈希扩容的范围
				- 迁移完成的桶会被标记为 `ForwardingNode`
	- 内部类：
		- Node：最基本的哈希节点类，用于存储键值对（继承自Entry）
			- 特点：它的 `val`和 `next`字段都用 `volatile`修饰
				- 确保了多线程环境下读取到的值是最新的，并且链表结构变更对其他线程立即可见
				- 使得读操作通常不需要加锁就能安全进行
		- ForwardingNode：扩容时使用
			- 使用时机：某个桶的数据迁移完成
			- 使用方式：在完成迁移的桶放置一个 `ForwardingNode`节点
				- 写操作线程发现了这个节点，则进行协作扩容，再写入
			- 存储内容：持有对新表 `nextTable`的引用，哈希取值为-1
				- 读操作线程发现这种节点，查询操作会被转发到新表上执行
					- 实现扩容期间查询操作的正确性
		- TreeNode​ 和 TreeBin
			- 使用时机：当桶内链表长度超过阈值（默认为8）时，链表会转换为红黑树，[[红黑树转化的阈值]]
				- 当哈希表长没有到64的时候，会先尝试扩容
				- 当哈希表长大于等于64的时候，才会进行链表转树操作
					- 如果桶内链表长度小于6，会进行树转链表操作
			- 作用：解决了红黑树平衡过程中线程安全问题
				- TreeNode代表红黑树的节点，
				- TreeBin是一个容器，用于包装整棵红黑树的根节点
					- 【包装】不仅有红黑树的根节点，还有一个读写状态锁
					- 【读写状态锁】通过 `WAITER`、`WRITER`、`READER`等状态控制对树的并发访问，
						- 多个读线程可以同时进行，
						- 写操作会进行适当的阻塞或等待
	- get函数：
		1. 计算哈希与定位桶
		    1. 它对 key 的哈希码进行一次特殊的 `spread`运算，
			    - 哈希值分布更均匀，减少冲突
			    - 哈希值变成正整数
			2. 确保哈希表已经创建，且数量不为空
				- 为空直接返回null
				- 在下次写的时候创建哈希表
		    3. 然后通过 `(table.length - 1) & h)`这个操作快速定位到 key 所在的哈希桶（数组下标）
		2. 定位到桶后，根据头结点，进行不同的查找方式
			- 【获得头结点的方式】使用了 `Unsafe.getObjectVolatile`
				- 确保每次读取到的都是主内存中的最新值，而不是线程工作内存中的缓存副本
			- 判断一个节点为目标节点的方式：
				- 先比较哈希值，若一致再比较 key。比较key的方式如下
					- 如果是同一个对象（比较引用地址 `==`）或者是值是相等的（比较值 `equals`）
					- 则都认为是同一个值
		    - 桶为空：最简单的情况，直接返回 `null
		    - 头节点即为目标节点：返回节点的value
			- 头节点不为目标节点：根据桶内的数据结构进行遍历
				- 桶内是ForwardingNode（头节点的哈希值是 `MOVED`（-1））：调用 `ForwardingNode`节点的 `find`方法，去新表查找
			    - 桶内是链表（头结点哈希≥0）：顺序遍历进行查找
			    - 桶内是红黑树（头节点的哈希值-2且节点类型为 `TreeBin`）：调用 `TreeBin`节点的 `find`方法。
					- 该方法内部会根据红黑树当前的锁状态智能选择遍历方式：
						- 判断写锁是否被占用：(lockState & WRITER) != 0（WRITER值为1）
						- 写锁被占用，调用 `LockSupport.park(this)`将自身挂起
						- 写锁未被占用，获取读锁，遍历红黑树
	- put函数
		- 内部只调用了putVal方法，核心内容都在这上面：
			- 有一个参数boolean onlyIfAbsent，这里传入了false
				- false表示写入覆盖
				- true表示如果有值则不进行更新
			- 不允许存入null的键和值：避免二义性，调用方无法区分这两种情况
				-  key 不存在于 map 中
				-  key 存在，但对应的 value 为 `null`
		- 整体流程：自旋（重试）机制进行写入
		1. 数组初始化：如果数组未初始化， 调用`initTable`，然后再次进入循环
			 - 方法返回时确保数组初始化完毕（被自己或者被别人）
		2. 定位与根据情况插入
			1. **空桶插入**：发现这个桶是空的（`null`）
				- 通过CAS将新节点赋值到空桶上。如果 CAS 失败：
				    - 代表着：有其他线程"抢先一步"占用了这个空桶
					    - 桶的状态不确定了，需要重新判断 
				    - 采取措施：不阻塞直接进入下一轮自旋，重新判断桶的状态
			2. **协助扩容**:目标桶的头节点的 `hash`字段等于 `MOVED`（-1）
			    - 调用 `helpTransfer`方法协作扩容
			3. 哈希冲突处理：桶不为空，且不在扩容状态，说明发生了哈希冲突
				1. 对桶的头节点进行同步加锁（`synchronized`），并且再次确认头结点没有被移动（快速判断状态+二次判断保证正确性的模式）
					- 细粒度锁：仅仅锁住一个桶
					- 保证在修改链表或树结构时的线程安全
				2. 将新节点插入到链表或红黑树中，根据onlyIfAbsent决定是否要覆盖写
					- 链表操作：【fh>=0】遍历链表，binCount记录链表节点数（用来插入后判断）
						- 如果找到相同的 key，则根据 `onlyIfAbsent`参数决定是否覆盖旧值。
						- 如果没找到，则将新节点插入到链表末尾。
					- 红黑树操作：如果桶的结构已经是红黑树（头节点是 `TreeBin`类型），则通过 `TreeBin`内部的方法进行红黑树的插入或更新操作。![[Pasted image 20251204192133.png]]
						- putTreeVal：如果key已经存在于节点中，会返回对应的TreeNode
							- 根据onlyIfAbsent决定要不要进行覆盖
		3. 插入完成后：
		    1. 判断是否要红黑树化
			2. 通过 `addCount`方法增加元素总数，并且判断是否要扩容
	- initTable方法：返回时确保数组初始化完毕（被自己或者被别人）
		- 整体架构：方法通过一个 `while`循环不断检查 `table`是否已被初始化。如果发现 `table`不为空，则直接返回，避免不必要的操作
		1. 首先检查sizeCtl
			- sizeCtl < 0：其他线程正在执行初始化或扩容
				- 调用 `Thread.yield()`暂时让出CPU时间片，然后再次进入循环检查状态
			- `sizeCtl >= 0`：进行竞争初始化权
		2. 竞争初始化权
			- 【竞争方式】通过 CAS 操作，尝试将 `sizeCtl`的值从当前值（`sc`）改为 -1
			- 失败：表示在其准备期间，已有其他线程抢先一步将 `sizeCtl`设置为 -1 并开始了初始化
				- 措施：继续循环
					- 下一次循环时会因 `sizeCtl < 0`而进入等待状态。
			- 成功：执行初始化与安全发布
		3. 执行初始化与安全发布
			- 整个初始化逻辑被包裹在 `try-finally`块中，
				- 确保无论初始化过程是否发生异常，都能正确恢复 `sizeCtl`的状态
			1. 再次检查 `table`是否为空（二次检查）
				- 防止在竞争过程中已被其他线程初始化
			2. 选择初始容量
				- 默认16
				- 构造 `ConcurrentHashMap`时可以指定了初始容量
			3. 计算扩容阈值`n * 0.75`，并赋值给sizeCtl
	- addCount方法：高效、线程安全地更新元素数量​ 和 在必要时触发扩容操作
		1. 分段计数
			- 分段计数的需求背景：计数的并发程度极高，不适合用CAS+volitile计数
				- 竞争激烈，会导致严重的性能瓶颈
			- 借鉴了 `LongAdder`的思想，采用**分段计数**的策略，将竞争分散
			- 流程：
				1. 尝试 CAS 更新 `baseCount`
				2. 转向 分段计数
		2. 扩容检查与协作
			1. 检查阈值：它会调用 `sumCount()`获取当前元素总数的近似值 `s`，然后判断 `s`是否大于或等于扩容阈值sizeCtl
			2. 协助扩容：如果表已经在扩容中（由其他线程发起），调用 `transfer`方法协助进行数据迁移。
			3. 发起扩容：如果没有线程在扩容，但当前元素数已达到阈值，当前线程会通过 CAS 设置 `sizeCtl`状态，然后作为主扩容线程发起扩容操作 `transfer`
	- transfer方法：它允许多个线程协同工作，将旧数组中的元素安全地迁移到新数组（通常是原长度的两倍），而在此过程中其他线程的读写操作仍能正常进行
		-  协作与进度感知
			- 读操作：会直接转向 `ForwardingNode`所指向的新数组 `nextTable`上进行查询
				- 获取到最新的数据
			- 写操作：helpTransfer
				1. 先帮忙数据迁移
				2. 再执行自己的插入
		1. 主扩容线程创建新的哈希表
			- 哈希表的大小为原数组的两倍
		2. 任务分片与工作领取：自旋尝试直至领取成功
			- CAS操作原子性地将 `transferIndex`减少一个预设的步长，获取一个数据段的任务
				- 步长：最小为16
				- 领取任务的长度：从 `transferIndex`新值到旧值
		3. 数据迁移：从段范围内的数组尾部向头部逐个处理每个桶
			- 处理空桶：使用 CAS 放置 `ForwardingNode`​ 节点作为标记
				- 表示此桶已处理完毕
				- 放置方式：CAS
			- 处理已处理桶：直接跳过
				- 【已处理的判断方式】头节点是 `ForwardingNode`
			- 处理实际数据桶：
				1. 使用 `synchronized`关键字锁住该桶的头节点
					- 防止在迁移过程中其他线程进行写操作
				2. 进行迁移：迁移完成后，在原桶位置设置 `ForwardingNode`标记
					- 链表迁移：将原链表中的节点根据其哈希值的高位（通过 `hash & n`计算）拆分成“低位链表”和“高位链表”（与 HashMap 类似）
						- 低位链表放入新数组的相同索引位置
						- 高位链表放入“原索引 + 旧数组长度”的位置
					- 红黑树迁移：同样根据高位进行拆分。拆分后若节点数过少，则会将其转换为链表
		4. 扩容的完成与收尾
			- 收尾数组的选取：最后一个退出扩容的线程
				- sizeCtl的低16位有效值表示活跃的扩容线程数
					- 第一个线程触发扩容时，设置 `sizeCtl`为2
					- 后续每个协作线程，通过 CAS 对`sizeCtl`操作
						-  加入时加1
						- 退出时减1
			- 收尾工作内容：
				1. 将 `nextTable`设置为新的 `table`
				2. 更新 `sizeCtl`为新的扩容阈值（新容量的0.75倍）