
设计背景
- HashMap并发安全问题
	- **数据覆盖导致丢失**：多线程同时 put 指向同一空桶时，各自插入导致写入覆盖，从而出现数据丢失
	- **死循环问题**（JDK7以下）：并发扩容时链表可能形成环形结构，get 操作陷入死循环
		- 根源：根源是**多线程同时修改一个链表+头插法反转链表顺序**
	- **元素计数不准确**：size 属性更新非原子，多线程并发导致计数错误
- Hashtable的性能瓶颈
	- **全局锁机制**：synchronized 修饰方法，即使操作不同键也需串行执行
	- **并发度极低**：从并发编程退化为串行执行，吞吐量严重受限

ConcurrentHashMap 的设计目标：在保证线程安全的同时，**最大化并发性能**

---
特点：
1. **弱一致性读**：
	- 需求背景：
		- 强一致性的代价：如果读操作也加锁 / 用 CAS 保证强一致性，会导致读性能大幅下降（ConcurrentHashMap 核心场景是高并发读、低并发写）。
		- 业务场景适配：大部分并发场景（如缓存、配置存储）不需要「实时强一致性」，只需要「最终一致性」（能接受短暂读到旧值，但不能读到脏数据）。
	- 设计目的：
		- 最大化读性能（无锁读），满足高并发读的场景需求。
		- 不读到脏数据是底线
	- 原因：
		- get方法读取的时候，会先把数组、节点等核心数据「拷贝到局部变量」，然后基于局部变量完成遍历
2. 弱一致的迭代器：
	- 需求背景： 如果追求强一致性迭代器，就必须在创建迭代器的瞬间锁定整个容器，或者在迭代过程中阻止所有写操作。这会导致严重的性能下降和线程阻塞。
	- 弱一致性现象：
		- 迭代器创建后，遍历过程中即使其他线程修改（增 / 删 / 改）Map，迭代器**不会抛出任何异常**；
		- 迭代器可能遍历不到「创建后新增」的元素，也可能读到「创建后被修改 / 删除」的旧值；
		- 迭代器遍历的元素一定是「完整、无脏数据」的（不会读到改了一半的节点），只是非实时最新。
	- 原因：迭代器基于「数组快照」遍历，不跟踪实时数据
		- ConcurrentHashMap 迭代器创建时，会**拷贝当前 `table` 数组的引用**（由 `volatile` 保证此时的数组引用是最新的），但一旦拷贝完成，迭代器后续所有遍历操作都基于这个「快照引用」，不再感知 `table` 的后续变化

3. size()/ mappingCount()：返回的是一个近似值。
	- 设计目的：如果要获得一个精确的 `size()`，就需要在统计的瞬间“冻结”整个Map的状态，确保在统计过程中没有任何线程在进行插入或删除操作。
	- 原因：借鉴了 `LongAdder`的**分而治之**思想
		- `size()`方法调用 `sumCount()`，**无锁地**遍历整个CounterCell数组，将 `baseCount`和所有 `CounterCell.value`累加起来
	- 解决措施：如何获得精确值
		1. **外部计数**：使用 `AtomicLong`或 `LongAdder`在业务层手动维护计数器。
		2. **全量扫描**：在低峰期，通过加锁或暂停写服务的方式遍历整个Map（性能代价大）			  
---
**核心机制**

1. **无锁读：**所有读操作（`get`）几乎都是无锁的
	- 需求背景：实现读读、读写均可并发
	- 实现方式：依靠 **volatile变量**（Node的val和next）的内存语义
		- Node节点中的val被声明为volatile：保证读线程总能获取到写线程更新后的最新value
		- Node节点中的next Node被声明为volatile：保证能识别到新增的节点的
		- 哈希表和新哈希表是volatile的，确保扩容开始和完成修改能被看见

2. **节点锁：**
	- 需求背景：
		- 前置痛点：
			- HashTable的synchronzied锁方法的全局锁降低性能，高并发下所有线程串行执行
			- 普通 HashMap 线程不安全，并发修改会触发 `ConcurrentModificationException`
		- Java7 分段锁的背景：想要突破全表锁的性能瓶颈，但当时 JVM 对 synchronized 优化不足，只能通过 “物理拆分哈希表” 来降低锁粒度。
			- 优化不足指的是没有锁升级机制
	- 解决措施：锁粒度从「分段」缩小到「单个哈希桶（数组下标对应的链表 / 红黑树）」
	- 具体机制：
		- 无竞争时用 CAS 无锁更新，竞争时对**节点头**加 synchronized 锁
			-  CAS 无锁：操作消除锁开销
			- **节点头**加 synchronized 锁：突破分段锁的并发度上限，同时降低内存开销
		- 链表长度超过 8 且数组长度 ≥ 64 时，转为红黑树

3. **解决哈希冲突的数据结构：链表→红黑树的转化**
	- 需求背景：哈希表存在哈希冲突，使用链表实现链地址法解决哈希冲突，再冲突严重的时候性能退化严重
	- 解决措施：当哈希冲突严重的时候，链表转化为红黑树，将查询 / 更新的时间复杂度控制在 O (logn)，避免长链表的性能退化。
	- 选择红黑树而非 AVL 树
		- 好处：红黑树插入 / 删除的旋转次数少，适合频繁更新的场景
		- 代价：红黑树查询效率略低于 AVL 树
	- 带来的问题：要平衡 “树化” 的触发条件，避免频繁树化 / 反树化带来的额外开销。
		- 树化阈值设为 8：
			- 权衡：阈值过低会导致频繁树化（增加开销），过高则长链表查询慢
			- 8的来源：泊松分布下，链表长度≥8 的概率仅 0.00000006，保证树化的必要性
		- 反树化阈值设为 6
			- 好处：避免树和链表的频繁切换（滞后阈值）
			- 代价：短时间内链表长度在 6-8 波动时，有少量性能损耗
		- 树化前提：数组长度≥64
			- 权衡：
				- 小数组情况下，扩容解决冲突性价比高，
				- 大数组情况下，树化解决冲突性价比高
			- 代价：小数组下长链表仍会存在，但小数组扩容快，整体更优
4. **协助扩容**：并行加速扩容
	- 需求背景：
		- 单线程扩容的痛点：HashMap 扩容由单个线程完成，数据量大时扩容耗时久，期间所有写操作阻塞，高并发下性能瓶颈明显。
		- 扩容与读写的冲突：如果扩容时阻塞所有读写，会导致并发可用性急剧下降。
	- 解决措施：协助扩容
		- 核心思想：当有线程在进行插入、删除等写操作时，如果发现当前Map正在扩容，那么它不会袖手旁观或单纯等待，而是会主动参与到扩容过程中，帮助迁移一部分数据。
	- 实现基础：
		- sizeCtl：能让一个线程意识到自己是收尾线程
			- 正常状态：sizeCtl = 扩容阈值（capacity * loadFactor）
			- 初始化中：sizeCtl = -1
			- 扩容中：sizeCtl = (rs << 16) + (2 + nThreads) 
				- 高16位：扩容戳（resizeStamp），与容量相关
				- 低16位：参与扩容的线程数+1
					- 线程参与协作则CAS加1
					- 线程完成协作则CAS减1
		- transferIndex：指向下一个待迁移的桶索引 
			- 从高到低
			- 每次减少stride个
		- `ForwardingNode`
			- 设计目的：为什么不设置为null，避免出现二义性。
				- null可能代表着该桶没有存放任何节点
				- null也可能代表该桶迁移到新的表中去了
			- 存储着新哈希表的位置
			- 标志容器正在进行哈希扩容：写进程需要进行协助
			- 标志这个桶已经完成了迁移：读进程应该去新的哈希表读取，如果读取不到则说明不存在
	- 工作流程：保证并发安全
		1. 协助迁移的入口：写操作看到ForwardingNode
			- 为什么选择先协助再操作：
		2. 迁移任务分配：
			- 分配的区间：从`[transferIndex-stride，transferIndex）`
			- 分配的方式：通过CAS竞争
				- 避免一段桶区间被两个线程领走，防止重复迁移和遗漏 
			- 出现的结果：transferIndex减小一个stride
		3. 进行迁移
			- 迁移方式：使用 `synchronized`关键字锁住该桶的头节点
				- 链表迁移：
					1. 区分低位链表”和“高位链表”
					2. **低位链表**（`hash & n`为0）放入新数组的相同索引位置
					3. **高位链表**(`hash & n`为1)放入“原索引 + 旧数组长度”的位置
				- 红黑树迁移：
					- 同样根据高位进行拆分。
					- 拆分后若节点数过少，则会将其转换为链表
			- 迁移完成：在原桶位置CAS设置 `ForwardingNode`
		4. 读操作的扩容感知：
			- 读操作遇到ForwardingNode时：
				- 通过ForwardingNode.nextTable转到新表 - 在新表中继续查找 
				- 如果新表中又遇到ForwardingNode（嵌套扩容），则继续跳转 
		5. 扩容完成：
			- 最后一个迁移线程（sizeCtl低16位为1）负责：
				- 将table引用指向新表，将nextTable置为null 
				- 更新sizeCtl为新阈值 
			- 此时所有后续操作都使用新表

----

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