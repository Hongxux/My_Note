---
aliases:
  - 对象包装器
  - Wrapper Classes
  - 自动装箱
  - autoboxing
---

- **包装器类的作用**​：
	- **包装器类（如Integer、Double）将基本类型（如int、double）转换为对象**，以便在需要对象的上下文中使用，例如在泛型集合中（如ArrayList< Integer>）。
		- 这些类是不可变的（immutable）和final的，不能修改或子类化。
		- **身份比较问题**​：
			- 包装器对象使用`==`操作符比较时，检查的是对象引用而非值，因此可能失败。
			- 应使用`equals`方法进行值比较。Java规范缓存了常见值（如-128到127的int），但不应依赖身份。
			- 但是如果是从`ArrayList<Integer>`中取出来的，用== 是比较值
	- **包装器类提供静态方法**（如`Integer.parseInt()`）进行字符串转换，这些方法与对象实例无关，但逻辑上放置在包装器类中。
	
- **自动装箱和拆箱**​：自动装箱将基本类型自动转换为对应的包装器对象（如`list.add(3)`转换为`list.add(Integer.valueOf(3))`），而自动拆箱将包装器对象转换为基本类型（如`int n = list.get(i)`转换为`int n = list.get(i).intValue()`）。这简化了代码，但可能带来性能开销。