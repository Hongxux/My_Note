
### ​**1. 迭代器接口方法**​

```
public interface Iterator<E> {
    E next();                  // 返回下一个元素
    boolean hasNext();         // 检查是否还有元素
    void remove();             // 删除当前元素
    default void forEachRemaining(Consumer<? super E> action); // 遍历剩余元素
}
```
### 2.遍历
#### ​**1. 遍历集合的核心逻辑**​

- ​**安全遍历原则**​：
    
    调用 `next()`前必须用 `hasNext()`检查，否则可能触发 `NoSuchElementException`。
    
- ​**遍历示例**​：
    
    ```
    Collection<String> c = ...;
    Iterator<String> iter = c.iterator();
    while (iter.hasNext()) {
        String element = iter.next();
        // 处理元素
    }
    ```
    

#### ​**2. 简化遍历的替代方案**​
##### for-each 循环
 **始终从头开始完整遍历**
    
```
    for (String element : c) {
        // 处理元素
    }
 ```
 
​**适用条件**​：任何实现 `Iterable`接口的对象（如标准库所有集合）：
    
```
    public interface Iterable<E> {
        Iterator<E> iterator();
    }
 ```
##### forEachRemaining：

从迭代器当前位置开始消费剩余元素（适合部分遍历后继续操作）。
 Lambda 表达式遍历：
    
```
    iterator.forEachRemaining(element -> {
        // 处理元素
    });
 ```
    

#### ​**3. 遍历顺序的特性**​

- ​**有序集合**​（如 `ArrayList`）：
    
    按索引顺序遍历（从索引 0 开始递增）。
    
- ​**无序集合**​（如 `HashSet`）：
    
    元素顺序随机，但能保证遍历所有元素。
    并非真随机，而是由**哈希桶分布**决定。同一JVM中，相同元素插入顺序下遍历顺序固定，但开发者不应依赖此顺序（因哈希扩容可能导致顺序变化）。
    _注：顺序无关性不影响计算类操作（如求和、计数）。_
        
#### ​**4. Java 迭代器的独特设计**​

- ​**与传统迭代器的区别**​：
    
    - ​**传统模型**​（如 C++ STL）：
        
        迭代器类似数组索引，可独立查询元素和移动位置。
        
    - ​**Java 模型**​：
        
        `next()`同时完成 ​**元素访问**​ 和 ​**游标移动**，迭代器位于元素之间（见图 9.3）。
        
    
- ​**类比输入流**​：
    
    `Iterator.next()`类似 `InputStream.read()`，每次调用“消费”一个元素。
    
### 3.删除
#### ​**1. `remove()`方法的规则**​
![[Pasted image 20251026193549.png]]
- ​**调用前提**​：
    
    必须紧跟在 `next()`之后调用，删除 `next()`返回的元素。
    
    _违规将抛出 `IllegalStateException`。_
    
- ​**正确删除示例**​：
    
    ```
    Iterator<String> it = c.iterator();
    it.next();        // 跳过首元素
    it.remove();      // 删除该元素
    ```
    
- ​**删除相邻元素的正确方式**​：严格遵循`next()`→ `remove()`→ `next()`→ `remove()`的交替模式
    
    ```
    it.next()         // 跳过元素 A
    it.remove();      // 删除元素 A
    it.next();        // 跳过元素 B
    it.remove();      // 删除元素 B
    ```
    
    ​**错误示范**​（连续调用 `remove()`会抛`IllegalStateException`（第二次`remove()`未关联有效元素）。）：
    
    ```
    it.remove();
    it.remove(); // ❌ IllegalStateException
    ```
    

---

### ​**关键总结**​

|特性|说明|
|---|---|
|​**遍历安全**​|始终用 `hasNext()`检查后再调用 `next()`。|
|​**简化遍历**​|优先选 for-each 循环或 `forEachRemaining`。|
|​**顺序依赖**​|`ArrayList`有序，`HashSet`无序但完整遍历。|
|​**删除约束**​|`remove()`必须紧跟 `next()`，且不能连续调用。|
|​**设计哲学**​|Java 迭代器通过耦合“访问+移动”操作，简化接口并避免位置索引的复杂性。|
