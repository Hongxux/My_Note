- ​**Object类的地位**​：Object类是Java中所有类的终极超类（cosmic superclass）。每个类都隐式扩展Object，即使没有显式使用`extends Object`。例如，`Employee`类自动继承Object，**无需手动声明**。这确保了**所有对象都共享一组基本方法，如`toString()`、`hashCode()`和`equals()**`。
	- ​**原始类型和数组**​：
		- **原始类型**：（如int、char、boolean）**不是对象**，因此不能直接赋值给Object变量。
		- **数组**：所有数组类型（包括原始类型数组，如`int[]`）**都是对象**，并继承Object。例如，`obj = new int[10];`是有效的，因为数组是对象。
- ​**Object变量的使用**​：
	- 可以使用Object类型的变量引用任何对象（如`Object obj = new Employee("Harry Hacker", 35000);`），这体现了多态性。
	- 但Object变量只能作为通用持有者；要访问对象的特定方法或字段，必须进行类型转换（如`Employee e = (Employee) obj;`）。这增加了灵活性，但也引入了类型安全风险，如果转换失败会抛出`ClassCastException`。
	- 不可以new Object（）：
		- ​**该调用哪个构造函数？​**​
		    
		    （`String`有 `String()`，`Integer`有 `Integer(int)`，构造方法不统一）
		    
		- ​**分配多少内存？​**​
		    
		    （不同对象大小不同，如 `String`包含 `char[]`，`Integer`只是一个 `int`包装）
- **实际应用**​：Object类常用于编写通用代码，如集合类（ArrayList、HashMap）存储任意对象，或作为方法参数来接受多种类型。但现代Java更推荐使用泛型来增强类型安全，减少显式转换。
### Object的方法重写
####  **Object的equals方法**
- **默认行为**​：Object类中的equals方法默认比较对象引用是否相同（即`==`操作），这对于许多类足够，但通常需要重写以基于对象状态进行比较。
	- **记录（Record）的自动实现**​：记录（record）类自动生成equals方法，比较所有字段值，无需手动重写。这体现了记录的不可变性和简化设计。
- [[重写Object的equals()方法]]
#### **Object的hashcode()方法**

- 在Java中，`hashCode`方法是`Object`类的一个重要方法，用于返回对象的哈希码（一个整数值）。哈希码在哈希表（如`HashMap`或`HashSet`）中用于快速定位对象，**因此其实现必须与`equals`方法兼容**，以确保对象在集合中的正确行为。
	- **兼容性要求**​：如果`x.equals(y)`返回`true`，则`x.hashCode()`必须等于`y.hashCode()`。例如，如果`Employee.equals`基于ID比较，那么`hashCode`也必须基于ID计算，而不是其他字段。
- ​**默认实现**​：在`Object`类中，`hashCode`方法基于对象的内存地址生成哈希码。这意味着即使两个对象状态相同，如果它们是不同的实例，默认哈希码也可能不同。
- [[重写Object的hashcode()方法]]
####  **Object的toString（）方法**
- 在Java中，`toString`方法是`Object`类的一个重要方法，用于返回对象的字符串表示形式。它广泛应用于日志记录、调试和字符串拼接场景。**​
	- `toString`方法返回一个字符串，描述对象的状态。默认实现来自`Object`类，返回类名和哈希码（如`java.lang.Object@1a46e30`），但这通常不够直观。重写`toString`可以提供更有意义的信息，例如：
```
// 默认toString输出示例
System.out.println(new Object()); // 输出类似 java.lang.Object@1a46e30
// 自定义toString输出
System.out.println(new Employee("Alice", 50000, 1987, 12, 15)); // 输出 Employee[name=Alice,salary=50000.0,hireDay=1987-12-15]
```
- **[[重写Object的toString()方法]]**
- **to​String方法的自动调用：**
	- ​**字符串拼接**​：当对象与字符串使用`+`操作符连接时，Java编译器自动调用`toString`
    ```
    Employee emp = new Employee("Bob", 60000, 1990, 10, 1);
    String msg = "Employee: " + emp; // 自动调用emp.toString()
    ```
	- ​**打印输出**​：使用`System.out.println(object)`时，内部调用`object.toString()`：
    ```
    System.out.println(emp); // 输出toString()结果
    ```
	- ​**替代写法**​：对于任何对象`x`，`"" + x`等效于`x.toString()`，甚至适用于基本类型（如`"" + 10`）。
    
- **数组的特殊处理**​

	数组继承`Object`的`toString`方法，但输出不友好（如`[I@1a46e30`表示整型数组）。应使用`Arrays.toString`或`Arrays.deepToString`：

```
int[] numbers = {2, 3, 5, 7};
System.out.println(Arrays.toString(numbers)); // 输出 [2, 3, 5, 7]
int[][] matrix = {{1, 2}, {3, 4}};
System.out.println(Arrays.deepToString(matrix)); // 输出 [[1, 2], [3, 4]]
```

- ​**记录（Record）类的自动实现**​

	记录类（record）自动生成`toString`方法，基于所有组件字段：

```
record EmployeeRecord(String name, double salary, LocalDate hireDay) {}
EmployeeRecord empRec = new EmployeeRecord("Charlie", 70000, LocalDate.of(1995, 5, 5));
System.out.println(empRec); // 输出 EmployeeRecord[name=Charlie,salary=70000.0,hireDay=1995-05-05]
```

无需手动重写，简化了代码。


​
