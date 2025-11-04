---
aliases:
---
### `final`域初始化安全性

定义：本质上是JVM提供的一种保障，**当其他线程通过一个引用看到包含`final`域的对象时，那么它一定能看到这些`final`域在构造函数中被设置的值，不会看到中间状态或默认值。​**​ 更重要的是，它还能看到通过这些`final`引用所指向的对象的正确状态。
#### 结合您的代码进行对比分析

我们看 `OneValueCache`的构造函数：


```
public OneValueCache(BigInteger i, BigInteger[] factors) {
  lastNumber = i; // 写入 final 域
  lastFactors = Arrays.copyOf(factors, factors.length); // 写入 final 域
}
```

##### 场景1：如果没有`final`关键字（普通域）

由于指令重排序，JVM可能先为`lastFactors`分配了内存地址（不再是`null`），但`Arrays.copyOf`这个填充数组内容的操作还没完成。这时如果对象引用被发布，另一个线程可能看到`lastFactors`指向一个内容不全或错误的数组。这就是“部分构造对象”。

##### 场景2：有`final`关键字（您代码的实际情况）

JVM会禁止将对`final`域的写入（`lastNumber = i`和 `lastFactors = ...`）重排序到构造函数之外。并且，对于引用型`final`域（`lastFactors`），JMM还保证通过这个`final`引用所到达的**对象内部状态**​（即新数组的内容）也对其他线程可见。
### final 字段
- **final字段的定义**​：通过final关键字声明的实例字段必须**在对象构造时初始化**（即在每个构造函数结束时必须被赋值），且**之后不能再被修改**。例如，Employee类中的name字段被声明为private final String name，确保name在对象生命周期内不变。
    
- ​**适用场景**​：final字段特别适用于基本类型（如int、double）和不可变类（如String），因为它们的值或状态不会改变，从而保证了对象的不可变性。
    
	- ​**可变类的注意事项**​：对于可变类（如StringBuilder），final只保证字段引用不变（即始终指向同一对象），但对象内部状态可以被修改（如通过append方法）。这可能导致混淆，因此需要谨慎使用。
    
- ​**设计优势**​：使用final字段可以提高代码的可读性、安全性和可预测性，减少意外修改的风险，并支持不可变对象模式。

### final 类
通过`final class`声明，防止类被继承。例如，`public final class Executive extends Manager`确保`Executive`类不能有子类。所有方法在final类中自动成为final方法。

### final 方法
通过`final`修饰方法，防止子类覆盖该方法。例如，`public final String getName()`确保`getName`方法的行为在子类中不变。

在final类中，所有方法自动成为final，但字段不是自动final的。