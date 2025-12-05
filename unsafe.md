- 需求：需要极致性能
	- Java语言的设计初衷是安全、易于使用，因此将指针等容易出错的底层操作封装了起来。但有些场景下，比如编写高性能的服务器、网络框架或并发库时，开发者需要突破这些限制，直接与操作系统和硬件交互。
- 解决方法：unsafe
	- 内存管理：直接分配、释放、读写堆外内存，绕过JVM堆内存限制。\
		```
			Unsafe unsafe = getUnsafe();
		// 分配 8 字节内存，返回内存起始地址
		long address = unsafe.allocateMemory(8L);
		
		// 向该地址写入一个 long 值
		unsafe.putLong(address, 123456789L);
		
		// 从该地址读取 long 值
		long value = unsafe.getLong(address);
		System.out.println(value); // 输出: 123456789
		
		// 切记！必须手动释放内存，否则会造成内存泄漏
		unsafe.freeMemory(address);
		```
	- CAS操作：提供硬件级别的原子性“比较并交换”操作，是无锁算法的基石。
		```
		class CASCounter {
		    private volatile long value = 0;
		    private static final long VALUE_OFFSET;
		    private static final Unsafe UNSAFE;
		
		    static {
		        try {
		            UNSAFE = getUnsafe();
		            // 获取 value 字段在 CASCounter 对象内存布局中的偏移量
		            VALUE_OFFSET = UNSAFE.objectFieldOffset(CASCounter.class.getDeclaredField("value"));
		        } catch (Exception ex) { throw new Error(ex); }
		    }
		
		    public void increment() {
		        long current;
		        do {
		            current = UNSAFE.getLongVolatile(this, VALUE_OFFSET); // 获取当前值
		            // 循环尝试：如果当前值仍是 current，则原子性地更新为 current+1
		        } while (!UNSAFE.compareAndSwapLong(this, VALUE_OFFSET, current, current + 1));
		    }
		}
		```
	- 线程调度：直接挂起（park）和恢复（unpark）线程，是Java锁机制的基础。
		-  `ark(boolean isAbsolute, long time)`: 挂起当前线程。
			- `isAbsolute`参数指示 `time`是绝对时间还是相对时间。
		- `unpark(Thread thread)`: 恢复一个被挂起的线程。
	- 类与对象操作：绕过正常构造方法和初始化过程实例化对象；获取和修改对象字段的内存偏移量、
		```
		class User {
		    private int id = 10; // 这个初始化不会被执行
		    public User() { this.id = 20; } // 这个构造器也不会被调用
		}
		
		User user = (User) UNSAFE.allocateInstance(User.class);
		System.out.println(user.id); // 输出: 0 (默认值)
		```
		```
		public class Guard {
		    private int ACCESS_ALLOWED = 1;
		    public boolean giveAccess() { return 42 == ACCESS_ALLOWED; } // 正常情况下返回 false
		}
		
		Guard guard = new Guard();
		Field f = guard.getClass().getDeclaredField("ACCESS_ALLOWED");
		long offset = UNSAFE.objectFieldOffset(f);
		UNSAFE.putInt(guard, offset, 42); // 直接修改内存中的值
		System.out.println(guard.giveAccess()); // 输出: true
		```
	- 内存屏障：精细控制指令重排序和内存可见性，实现更细粒度的并发控制。
- 获取unsafe实例：通过反射获取
	```
	public class UnsafeGetter {
	    public static Unsafe getUnsafe() throws Exception {
	        // 1. 获取Unsafe类内部的私有静态字段"theUnsafe"
	        Field theUnsafeField = Unsafe.class.getDeclaredField("theUnsafe");
	        // 2. 设置这个字段为可访问
	        theUnsafeField.setAccessible(true);
	        // 3. 因为这个字段是静态的，所以传入null即可获取它的值
	        return (Unsafe) theUnsafeField.get(null);
	    }
	
	    public static void main(String[] args) throws Exception {
	        Unsafe unsafe = getUnsafe();
	        // 现在可以使用unsafe对象了
	    }
	}
	```
