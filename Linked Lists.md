

## 1. 数组和ArrayList的局限性

- ​**主要缺点**​：从数组中间删除或插入元素时代价高昂，因为需要移动后续所有元素（见图9.6）。
    
- ​**应用场景**​：频繁在中间位置进行插入或删除操作时，数组和ArrayList效率较低。
    

## 2. 链表的基本概念和优势

- ​**基本结构**​：
    
    - 链表将每个对象存储在单独的节点中。
        
    - 每个节点包含数据元素和指向下一个节点的引用。
        
    - Java中的链表是**双向链表**，每个节点还包含指向前驱节点的引用
        
    
- ​**优势**​：
    
    - ​**高效中间操作**​：删除或插入元素时只需更新相邻节点的引用（见图9.8），无需移动大量元素。
        
    - 适合频繁修改的场景。
    - -双向链表优势：
	    - 可以双向遍历（前向和后向）
    
		- 删除操作更高效（可以直接访问前驱节点）
    
		- 支持反向迭代器(previous()和hasprevious())
        

---


## 3. Java中的LinkedList类

- Java集合库提供了现成的`LinkedList`类，无需手动实现链表操作。
    
- ​**基本用法示例**​：
    
    ```
    // 创建链表并添加元素
    var staff = new LinkedList<String>();
    staff.add("Amy");  // 添加到末尾
    staff.add("Bob");
    staff.add("Carl");
    
    // 使用迭代器删除第二个元素
    Iterator<String> iter = staff.iterator();
    iter.next();  // 访问第一个元素
    iter.next();  // 访问第二个元素
    iter.remove(); // 删除当前访问的元素
    ```
    

## 4. 迭代器和ListIterator

- 链表是**有序集合**，需通过迭代器定位位置。
    
- ​**ListIterator接口**扩展了Iterator，支持**双向遍历**和位置相关操作（增删改），有**额外功能**
```
// 额外功能
listIter.add("New");     // 添加元素
listIter.set("Modified"); // 修改元素
int index = listIter.nextIndex(); // 获取下一个元素的索引
```
```
// 反向遍历
while (listIter.hasPrevious()) {
    String item = listIter.previous();
    System.out.println("反向: " + item);
}
```
    
ListIterator有两个**获得方法**：
![[Pasted image 20251026205333.png]]
### 4.1 ListIterator的add方法规则

- ​**插入位置**​：新元素添加到迭代器当前位置之前。
    
- ​**示例**​：在第二个元素前插入"Juliet"
    
    ```
    ListIterator<String> iter = staff.listIterator();
    iter.next();        // 跳过第一个元素
    iter.add("Juliet"); // 在第二个元素前插入
    ```
    
- ​**特殊位置**​：
    
    - **迭代器在链表开头时插入**（获得遍历器，但是还没有访问），新元素成为新头节点。
	```
	// 场景1：在列表开头添加
	ListIterator<String> iter = names.listIterator(); // 指向开头
	iter.add("First"); // 成为新的头节点
	```

    - **迭代器越过末尾时插入**（hasnext()返回false），新元素成为新尾节点。
        ```
        // 场景2：在列表末尾添加
		while (iter.hasNext()) iter.next(); // 移动到末尾
		iter.add("Last"); // 成为新的尾节点
		```
    - **迭代器连续添加：**
    ```
	// 场景3：多次连续添加
	iter.add("X"); // 添加X
	iter.add("Y"); // 在X之后添加Y
	iter.add("Z"); // 在Y之后添加Z
	// 结果：X → Y → Z（都添加在迭代器当前位置之前）
	```
- ​**插入点数量**​：n个元素的链表有n+1个插入位置（包括头尾）。
    

### 4.2 remove和set方法的注意事项
**remove的规则**
- **规则一：`remove`删除的是“最后一个被返回的元素”**。
	- 调用 `next()`后，最后一个返回的元素是 `next()`返回的那个。此时 `remove()`会删除它。
	- 调用 `previous()`后，最后一个返回的元素是 `previous()`返回的那个。此时 `remove()`会删除它。
-  **规则二：`remove`方法依赖于“迭代器状态”**。 
	- `add`方法（对于 `ListIterator`）只关心迭代器的**当前位置**，总是在当前位置前面插入一个新元素。 
	- 而 `remove`方法关心的是**最近一次成功调用的方法是 `next`还是 `previous`**。这个“最近的操作历史”就是它的状态。
- **规则三：不能连续调用 `remove`两次**。
	- 在每次调用 `remove()`之前，**必须**先调用一次 `next()`或 `previous()`来“重新定位”一个可删除的元素。调用 `remove()`后，这个“状态”就被消耗掉了。
    
```
LinkedList<String> list = new LinkedList<>();
list.addAll(Arrays.asList("A", "B", "C", "D"));
ListIterator<String> iter = list.listIterator();

iter.next(); // 返回"A"，最后一个返回元素是"A"
iter.remove(); // 删除"A" - 规则一

iter.next(); // 返回"B"
iter.previous(); // 返回"B"（又回到了B）
iter.remove(); // 删除"B" - 规则一

// 错误示例：
iter.remove(); // 抛出IllegalStateException - 规则三
// 必须再次调用next()或previous()后才能remove()

iter.next(); // 重新定位
iter.remove(); // 现在可以删除
```

- ​**set方法**​：替换最后访问的元素（由`next()`或`previous()`返回），规则同理。
```
LinkedList<String> list = new LinkedList<>();
list.addAll(Arrays.asList("A", "B", "C"));

ListIterator<String> iter = list.listIterator();
iter.next(); // 返回"A"
iter.set("X"); // 将"A"替换为"X"

iter.previous(); // 返回"X"  
iter.set("Y"); // 将"X"替换为"Y"
// 结果：["Y", "B", "C"]
```
## 5. 并发修改异常（ConcurrentModificationException）

- ​**触发条件**​：一个迭代器遍历集合时，另一个迭代器或集合本身修改了结构。
- **根本原因**​：

	- 集合维护`modCount`（修改计数器）
	    
	- 迭代器创建时记录预期的`modCount`
	    
	- 每次操作前检查是否一致，不一致则抛出异常
- ​**示例**​：
    
    ```
    ListIterator<String> iter1 = list.listIterator();
    ListIterator<String> iter2 = list.listIterator();
    iter1.next();
    iter1.remove();    // 修改结构
    iter2.next();      // 抛出ConcurrentModificationException
    ```
    
- ​**避免规则**​：
    
    - 可附加多个只读迭代器。
        
    - 或只附加一个可读写迭代器。
        
    
- ​**例外**​：`set`方法不视为结构性修改，多个迭代器可同时调用。
    

---

## 6. 链表的效率问题和最佳实践
**迭代器遍历的优势**​：
```
LinkedList<String> list = new LinkedList<>();
// 填充数据...

// 方式1：使用索引（不推荐）
for (int i = 0; i < list.size(); i++) {
    String item = list.get(i); // 每次都是O(n)操作！
}

// 方式2：使用迭代器（推荐）
Iterator<String> iter = list.iterator();
while (iter.hasNext()) {
    String item = iter.next(); // O(1)操作
}

// 方式3：增强for循环（底层使用迭代器）
for (String item : list) { // 等价于使用迭代器
    System.out.println(item);
}
```

​**优先使用迭代器**进行遍历和修改操作，**避免随机访问**​（get(i)），特别是在循环中

---

## 7. 示例程序（Listing 9.1）

- ​**功能**​：演示链表操作，包括合并列表、间隔删除和批量移除。
    
- ​**关键代码**​：
    
    ```
    // 将列表b合并到a中
    ListIterator<String> aIter = a.listIterator();
    Iterator<String> bIter = b.iterator();
    while (bIter.hasNext()) {
        if (aIter.hasNext()) aIter.next();
        aIter.add(bIter.next());
    }
    
    // 删除b中每隔一个的元素
    bIter = b.iterator();
    while (bIter.hasNext()) {
        bIter.next(); // 跳过一个元素
        if (bIter.hasNext()) {
            bIter.next(); // 跳过下一个元素
            bIter.remove(); // 删除该元素
        }
    }
    
    // 从a中批量移除b的所有元素
    a.removeAll(b);
    ```
    
- ​**调试提示**​：使用`System.out.println(list)`输出链表内容（调用`toString`方法）。
    

## 8. API摘要

![[Pasted image 20251026205033.png]]
![[Pasted image 20251026205050.png]]
![[Pasted image 20251026205058.png]]
![[Pasted image 20251026205107.png]]

## 总结

- ​**链表适用场景**​：频繁在中间位置插入或删除元素。
    
- ​**核心操作工具**​：使用`ListIterator`进行遍历和修改。
    
- ​**效率关键**​：避免随机访问，优先使用迭代器。
    
- ​**选择建议**​：根据需求选择数据结构——随机访问用数组/ArrayList，频繁修改用链表。

