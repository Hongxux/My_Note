---
aliases:
  - 弱引用
  - 软引用
  - 强引用
  - 虚引用
  - 终结器引用
---
- 强引用，就是我们最常见的普通对象引用
	- 与垃圾回收的关系：
		- 根据可达性算法：只要还有强引用指向这个对象, 垃圾回收器就不会回收这种对象。
			- 即使内存溢出，都不会回收、
- 软引用
	- 需求背景
		- 数据缓存的需要：避免频繁的数据库查询、网络请求或磁盘I/O
		- 内存资源的限制：被强引用所持有的缓存对象无法被垃圾回收，从而导致OOM
	- 解决措施：软引用。内存不足时候被回收
		- 在内存充足时，保留缓存对象以获得性能提升；
			- 垃圾回收器不会回收被软引用关联的对象，缓存正常发挥作用
		- 在内存紧张时，自动释放这些非必需的缓存对象，以避免内存溢出
			- 垃圾回收器会清除这些软引用，并回收它们所指向的对象
	- 问题：
		- 缓存命中率下降：缓存可能由于内存压力而被迫回收
		- 对象存活时间不确定：存活时间受到不确定的GC影响
		- 无效软引用对象队列
	- 使用示例：
		```java
		// 1. 创建一个大对象，并用软引用包装它
		SoftReference<byte[]> softCache = new SoftReference<>(new byte[1024 * 1024 * 500]); // 缓存一个约500MB的数据
		
		// 2. 在程序需要时尝试从缓存中获取
		byte[] largeData = softCache.get();
		if (largeData != null) {
		    // 缓存命中，直接使用数据
		    System.out.println("从缓存成功获取数据。");
		} else {
		    // 缓存未命中（对象因内存不足已被回收），需要重新创建
		    System.out.println("缓存未命中，重新加载数据...");
		    largeData = new byte[1024 * 1024 * 500]; // 模拟重新加载数据
		    softCache = new SoftReference<>(largeData); // 重新放入缓存
		}
		
		// 3. 当不再需要强引用时，确保移除，让软引用成为唯一引用
		largeData = null;
		```
		- 需要实现缓存失效的自动按需重建
		- 需要使用缓存的时候，通过get方法，将其赋值给强引用对象
		- 当不需要缓存的时候，将其相关的强引用设置为null
			- 让软引用成为其唯一引用
- 弱引用
	- 需求背景：
		- 存在主对象（key）与元数据关联（value）的需求
		- 如果使用hashMap实现这种管理：当我们不再需要主对象时，需要手动删除在HashMap对应Entry
			- HashMap是强引用，会导致主队先无法被自动垃圾回收
	- 解决措施：弱引用 WeakHashMap，实现了键值对与HashMap生命周期的解绑，而与主对象的生命周期绑定
		- 它的键（Key）是弱引用的。当作为键的对象在其他地方没有强引用时，它就可以被GC回收
		- 随后，`WeakHashMap`会有机制自动清理掉对应的整个键值对
	- 适合场景：缓存更适合创建成本不高、可以容忍丢失、但占用内存较大的对象
		- 容忍丢失：如果一个对象持有文件句柄、数据库连接、网络套接字等关键资源，丢失会导致资源泄露，此时不能容忍丢失
			- 解决措施：先使用 `try-with-resources`语句和`AutoCloseable`接口，确保资源使用完毕后被确定性地关闭
		- 创建成本不高：有些缓存使用频率高，但是缓存的重建需要访问数据库或者网络IO
			- 轻微的GC都可能导致被回收，使得缓存命中率降低
			- 解决措施：使用成熟的缓存库（如Caffeine、Guava Cache）
				- 提供了更丰富的淘汰策略（如LRU、LFU），能更好地平衡内存使用和性能
	-  弱引用与垃圾回收的关系：
		- 前提：该对象只具备弱引用
		- 行为：下一次GC时必然被回收
			- 弱引用所引用的对象：不管当前内存空间足够与否，都会回收它的内存。
			- 弱引用对象：可以被放入引用队列
				- 引用队列：当需要对软弱引用进行垃圾回收的时候，需要使用这个队列 
	- 使用示例：
		```java
		import java.util.Map;
		import java.util.WeakHashMap;
		
		public class WeakHashMapDemo {
		    public static void main(String[] args) {
		        Map<Object, String> metadata = new WeakHashMap<>();
		
		        Object someObject = new Object();
		        // 将某个对象作为键，与其元数据关联
		        metadata.put(someObject, "这是这个对象的元数据");
		
		        // 只要 someObject 存在，就能通过它获取元数据
		        System.out.println("对象存在时: " + metadata.get(someObject)); // 有输出
		        System.out.println("Map大小: " + metadata.size()); // 大小为1
		
		        // 当这个对象在程序的其他部分不再被强引用时...
		        someObject = null; // 切断强引用
		
		        System.gc();
		        try { Thread.sleep(100); } catch (InterruptedException e) { e.printStackTrace(); }
		
		        // 此时，WeakHashMap 会自动清理掉这个键值对
		        System.out.println("GC后，Map大小: " + metadata.size()); // 大小很可能变为0
		    }
		}
		```
- 虚引用：
	- 虚引用（`PhantomReference`）必须与[[引用队列]]联合使用
		- 虚引用引用的对象被垃圾回收之后，虚引用引用队列会被放入引用队列
		- 有一个线程不断扫描这个队列，对新入队的虚引用对象，调用该对象的clean方法
- 终结器引用：
	- 来源：一个对象，重写了finalize方法（Object的方法），当其不被强引用引用时候，将要被垃圾回收时候，JVM会自动创建一个终结器引用
	- 作用机制：
		- 第一次垃圾回收时候，终结器引用会被放入引用队列
		-  有一个线程不断扫描这个队列，对新入队的终结器引用对象所引用的对象，调用该对象的finalized()方法
		- 调用完后，在第二次垃圾回收的时候，就可以把终结器引用对象，及其所引用的对象一起回收
	- 问题：要两次垃圾回收才能回收，垃圾回收效率低
		- 使用 `AutoCloseable`接口和 `try-with-resources`语句，来确保资源能够被及时且确定地释放