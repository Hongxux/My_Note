![[Pasted image 20251022145554.png]]
​**顶层设计：​**​ Java 异常体系的根是 `java.lang.Throwable`类。所有异常和错误都是它的子类。它有两个直接的子类：`java.lang.Error`和 `java.lang.Exception`。这套体系的核心目的是**将程序的控制权从错误发生点转移到合适的错误处理器**。
#### **Error vs Exception 本质区别**​

Exception是程序员可以处理的异常，Error是程序员无法处理的，我们只要关注Exception就好了。

|​**维度**​|​**Error**​|​**Exception**​|
|---|---|---|
|​**产生来源**​|JVM或底层系统故障|应用程序逻辑或外部环境问题|
|​**可恢复性**​|不可恢复（如内存耗尽）|可捕获处理（如文件未找到）|
|​**处理责任方**​|JVM自身|应用程序开发者|
|​**典型子类**​|`OutOfMemoryError`, `StackOverflowError`|`IOException`, `SQLException`|
|​**处理建议**​|不捕获，直接终止进程|必须捕获或声明|

> ​**设计哲学**​：Error代表系统级灾难，Exception反映应用级异常。Java通过此分离强制开发者关注可修复问题。
---

#### RuntimeException和其他Exception
- ​**根本不同点：​**​ 核心区别在于 ​**异常发生的原因和责任归属**。这直接导致了编译器对它们的不同处理要求。
    
- ​**​“你的错误”（RuntimeException） vs “外部错误”（Checked Exception）：​**​
    
|类别|RuntimeException（未经检查的异常）|其他 Exception（受检异常，Checked）|
|---|---|---|
|​**根本原因**​|​**编程逻辑错误**​（Bug）。是程序员的过错。|​**外部环境问题**或**意外情况**。程序本身逻辑可能正确。|
|​**作者解释**​|​**​“你的错误”​**​：因为你犯了错。|​**​“外部错误”​**​：坏事发生在你原本好的程序上。|
|​**处理理念**​|​**应通过修复代码来预防**，而不是在运行时大量捕获。|​**必须在代码中处理**，因为无法通过编码完全避免。|
|​**典型例子**​|`ClassCastException`（类型转换错误）  <br>`ArrayIndexOutOfBoundsException`（数组越界）  <br>`NullPointerException`（空指针）|`IOException`（如文件未找到）  <br>`SQLException`（数据库连接失败）  <br>`ClassNotFoundException`（类不存在）|

​**代码示例：​**​

```
// === "你的错误" (RuntimeException) - 应修复代码 ===
String str = null;
// 这是代码Bug，应确保str不为null，而不是依赖捕获
// int length = str.length(); // 抛出 NullPointerException

// === "外部错误" (Checked Exception) - 必须处理 ===
try {
    Files.readString(Path.of("nonexistent.txt")); // 可能抛出 IOException
} catch (IOException e) {
    // 必须处理，因为文件是否存在是外部因素
    System.out.println("文件未找到，进行错误处理。");
}
```

---

#### ​**受检异常与非受检异常**​

- ​**定义：​**​
    
    - ​**非受检异常：​**​ 指 `Error`及其子类，以及 `RuntimeException`及其子类。编译器**不强制**要求你在代码中处理（捕获或声明抛出）这些异常。
        
    - ​**受检异常：​**​ 指除了 `Error`和 `RuntimeException`之外的所有其他 `Exception`的子类。编译器**强制**要求你必须为它们提供异常处理器。
        
    
- ​**编译器要求的不同：​**​ 这是最关键的实践区别。
    

```
// === 非受检异常 (Unchecked) ===
// 编译器不检查，不处理也能编译通过
public void example1() {
    throw new RuntimeException(); // 编译通过
    throw new Error(); // 编译通过
}

// === 受检异常 (Checked) ===
// 编译器强制检查，不处理则编译报错
public void example2() {
    // throw new IOException(); // 编译错误：Unhandled exception: java.io.IOException
}

// 正确写法：要么捕获（try-catch）...
public void example2Fixed1() {
    try {
        throw new IOException();
    } catch (IOException e) {
        // 处理异常
    }
}

// ...要么声明抛出（throws）
public void example2Fixed2() throws IOException {
    throw new IOException();
}
```

​**总结关系图：​**​

```
java.lang.Throwable
              /      \
             /        \
     **Error**      **Exception**
     (非受检)         /       \
                    /         \
    **RuntimeException**   **其他Exception**
        (非受检)            (受检异常, Checked Exception)
```