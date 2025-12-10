---
aliases:
  - 对象包装器
  - Wrapper Classes
  - 自动装箱
  - autoboxing
  - 包装器类
---

- **包装器类的作用**​：
	- **包装器类（如Integer、Double）将基本类型（如int、double）转换为对象**，以便在需要对象的上下文中使用，例如在泛型集合中（如ArrayList< Integer>）。
		- 这些类是不可变的（immutable）和final的，不能修改或子类化。
		- **身份比较问题**​：
			- 包装器对象使用`==`操作符比较时，检查的是对象引用而非值，因此可能失败。
			- 应使用`equals`方法进行值比较。Java规范缓存了常见值（如-128到127的int），但不应依赖身份。
			- 但是如果是从`ArrayList<Integer>`中取出来的，用== 是比较值
	- **包装器类提供静态方法**（如`Integer.parseInt()`）进行字符串转换，这些方法与对象实例无关，但逻辑上放置在包装器类中。
	
- **自动装箱和拆箱**​：自动在基本数据类型（如 `int`）和对应的包装类（如 `Integer`）之间进行转换。
	- **语法糖写法**：
		```
		Integer i = 10; // 自动装箱 (int -> Integer)
		int j = i;      // 自动拆箱 (Integer -> int)
		```
	- **解糖后代码**：
		```
		Integer i = Integer.valueOf(10); // 自动装箱
		int j = i.intValue();            // 自动拆箱
		```
	- **主要注意事项**：
		- **空指针风险**：如果包装类对象为 `null`，进行拆箱操作会抛出 `NullPointerException`。
			```
			Integer nullInteger = null;
			int n = nullInteger; // 运行时抛出 NullPointerException
			```
		- **性能开销与对象缓存**：装箱操作会创建对象。注意包装类的缓存机制（如 `Integer`默认缓存 -128 至 127），超出缓存范围会创建新对象 。
			```
			Integer a = 127;
			Integer b = 127;
			System.out.println(a == b); // true, 从缓存中取，是同一个对象
			
			Integer c = 128;
			Integer d = 128;
			System.out.println(c == d); // false, 超出缓存范围，是新创建的对象
			```

