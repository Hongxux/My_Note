---
aliases:
  - main
---
- ​**main方法的静态性**​：main方法必须声明为静态（`public static void main(String[] args)`），因为它作为程序的入口点，在对象创建之前执行。它不依赖于任何对象实例，直接由Java虚拟机调用。
    
- ​**单元测试用途**​：任何类都可以定义自己的main方法，用于单元测试或演示代码。例如，Employee类中的main方法测试了对象创建和方法调用，而StaticTest类中的main方法展示了多个Employee对象的使用和静态方法调用。
    
- ​**静态方法访问**​：静态方法（如main）只能访问静态字段和其他静态方法，不能直接访问实例字段。例如，Employee类的advanceId静态方法操作静态字段nextId。
    
- ​**程序启动**​：当运行Java程序时，JVM会执行指定类的main方法。如果类有多个main方法，只有通过`java ClassName`调用的那个会被执行。
    
- ​**相关工具方法**​：简要提到了java.util.Objects类的静态方法（如requireNonNull），用于null检查，这些方法在后续章节中详细解释。