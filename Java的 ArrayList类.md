​
### **动态数组需求**​：
固定大小数组在编译时确定大小，无法动态调整，导致内存浪费或不足。ArrayList提供了自动容量调整，解决这一问题。
- **与数组对比**​：ArrayList更灵活，但访问元素需使用方法而非索引运算符。与C++ vector类似，但Java不支持运算符重载，且赋值是引用共享而非值复制。
- **泛型声明**​：ArrayList是泛型类，使用类型参数指定元素类型（如`ArrayList<Employee>`）。声明时可以使用var关键字或菱形语法（`new ArrayList<>()`）简化代码。
- **容量与大小**​：容量（capacity）是内部数组的分配空间，大小（size）是实际元素数量。
	- `size()`返回元素数量
	- `ensureCapacity(int)`预分配容量
	- `trimToSize()`调整容量以匹配大小，优化内存使用。
	- 在创建的时候可以在()内指明大小
- **性能考虑**​：ArrayList的自动扩容和中间插入/删除操作可能带来性能开销，对于大型集合或频繁修改，应考虑替代数据结构。
### ArrayList的常用方法
- **元素添加、删除方法：**`add(E)`添加元素，`remove(int index)`删除元素
	- **中间操作**​：通过`add(int index, E element)`和`remove(int index)`方法在中间插入或删除元素，但效率较低，频繁操作时建议使用链表
- **元素访问语法**​：ArrayList使用`get(int index)`和`set(int index, E element)`方法访问和修改元素，而不是数组的`[]`语法。例如，`staff.get(i)`等效于数组的`a[i]`，`staff.set(i, harry)`等效于`a[i] = harry`。
	- 初始填充应使用`add`方法，使用set方法会报错。
- **转换和遍历**​：可以使用`toArray()`方法将ArrayList转换为数组；支持for-each循环遍历（如`for (Employee e : staff)`），简化代码。