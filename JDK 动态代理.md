- 基于
- 使用：
	- 创建动态代理对象：`Proxy.newProxyInstance()`JVM 会在运行时为你“凭空”创建一个新的类（代理类）并返回它的实例。
		- 工作流程：
		    1. 在内存中动态生成代理类的字节码
			    - 该代理类会实现指定的接口。
		    2. 实例化代理对象并返回：
			    - 方式：根据生成的字节码，通过反射创建出代理对象的实例。
		- 参数
			- `ClassLoader loader` 类加载器：用于**加载**在内存中动态生成的代理类。
				- 必须使用**目标类**的类加载器，即 `target.getClass().getClassLoader()`。
			- `Class[] interfaces`接口数组：指定动态生成的代理类需要**实现哪些接口**
				- ：代理类和目标类必须实现相同的接口，这样才能实现“透明”替换，即 `target.getClass().getInterfaces()`。
			- [[InvocationHandler]]调用处理器：是一个接口，你需要在它的实现类中编写增强逻辑（如日志、事务等）
		- 示例：
			```
			// target 是已有的目标对象
			OrderService proxyObj = (OrderService) Proxy.newProxyInstance(
			        target.getClass().getClassLoader(), // 参数1：类加载器
			        target.getClass().getInterfaces(),   // 参数2：接口数组
			        new TimerInvocationHandler(target)   // 参数3：调用处理器实例
			);
			```

- 问题：
	- “自调用”失效问题：
		- 现象：在代理对象的一个方法内部调用同一个对象的另一个方法，第二个方法不会经过代理的增强逻辑。
		- 原因：这是因为调用**发生在目标对象内部**，**没有经过代理对象**
	- 只能拦截接口中声明的方法：
		- 如果目标类有不在接口中的公有方法，通过代理对象调用该方法时，增强逻辑不会生效
	- 性能开销：反射调用比直接调用慢
		- 比起CGLIB：在现代JDK版本（如JDK 8及以上）中，由于**反射机制的优化**，两者在方法调用性能上的**差距已经显著缩小**
		- 优化方式：
			- 缓存代理对象（避免重复创建）
			- 缓存 `Method`对象（避免每次调用都进行反射查找）
	- 接口依赖：
		- 因为 Java 是单继承的，JDK 动态代理生成的代理类已经继承了 `java.lang.reflect.Proxy`类，因此无法再继承其他类，只能通过实现接口的方式