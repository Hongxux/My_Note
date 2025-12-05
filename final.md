---
aliases:
---

### final 字段
- **final字段的定义**​：通过final关键字声明的实例字段必须**在对象构造时初始化**（即在每个构造函数结束时必须被赋值），且**之后不能再被修改**。例如，Employee类中的name字段被声明为private final String name，确保name在对象生命周期内不变。
- final字段初始化的原理：
	- 实现的效果：当其他线程通过一个引用看到包含`final`域的对象时，那么它一定能看到这些`final`域在构造函数中被设置的值，不会看到中间状态或默认值。
	- 实现方式：在对final修饰的变量赋值后加入[[写屏障]]指令
    
- ​**适用场景**​：final字段特别适用于基本类型（如int、double）和不可变类（如String），因为它们的值或状态不会改变，从而保证了对象的不可变性。
    
	- ​**可变类的注意事项**​：对于可变类（如StringBuilder），final只保证字段引用不变（即始终指向同一对象），但对象内部状态可以被修改（如通过append方法）。这可能导致混淆，因此需要谨慎使用。
    
- ​**设计优势**​：使用final字段可以提高代码的可读性、安全性和可预测性，减少意外修改的风险，并支持不可变对象模式。

### final 类
通过`final class`声明，防止类被继承。例如，`public final class Executive extends Manager`确保`Executive`类不能有子类。所有方法在final类中自动成为final方法。

### final 方法
通过`final`修饰方法，防止子类覆盖该方法。例如，`public final String getName()`确保`getName`方法的行为在子类中不变。

在final类中，所有方法自动成为final，但字段不是自动final的。