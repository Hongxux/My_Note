---
aliases:
  - 静态字段
  - static
---
- **与实例字段的区别**​：实例字段（如`id`）每个对象都有自己的副本，而静态字段（如`nextId`）只有一个副本，被所有对象共享。
	- **共享性**​：静态字段被所有实例共享，这意味着任何对象对静态字段的修改都会影响其他对象。
    
	- ​**生命周期**​：静态字段在类加载时初始化，并在程序运行期间持续存在，不依赖于任何实例。
    
	- ​**访问方式**​：静态字段可以通过类名直接访问（如`Employee.nextId`），而不需要创建对象实例。
    
	- ​**设计意义**​：使用静态字段可以高效管理全局状态或共享数据，但需谨慎使用，以避免不必要的全局状态。
```java
	class Employee
	{
	 private static int nextId = 1;
	 private int id;
	}
```
```java
	id = nextId;
	nextId++;
	等同于：
	harry.id = Employee.nextId;
	Employee.nextId++;
```
- **初始化**​：静态字段可以在声明时初始化（如`private static int nextId = 1;`），并且即使没有创建对象，静态字段也存在。