---
aliases:
  - 双重检查锁定
  - DCL
---
```java
public final class singleton{  
    private singleton(){ }  
    private static singleton INSTANCE = nul1;  
    public static singleton getInstance() {  
        synchronized (singleton.class) {  
            if (INSTANCE == nu11) {  
                INSTANCE = new Singleton();  
                  
            }  
            return INSTANCE;  
        }  
    }  
}
```
- 需求背景：
	- 在单例模式中，想要避免多次初始化，于是使用锁，进行判断检查和初始化
	- 但是存在问题，这样每个线程都需要进入锁才能判断实例有无初始化，开销大
- 解决方法：两次判断INSTANCE有无初始化
	- 一次在同步块外
	- 一次在同步块外

```java
public class Singleton {
    private static Singleton instance; // 在JDK5之前，这里通常没有volatile关键字
    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查：避免不必要的同步
            synchronized (Singleton.class) { // 加锁
                if (instance == null) { // 第二次检查：确认实例尚未创建
                    instance = new Singleton(); // 问题根源所在
                }
            }
        }
        return instance;
    }
}
```
- 好处：仅仅首次初始化存在锁竞争，后续使用不会争夺锁，不进入同步块
- 存在问题：有序性问题
	- 在初始化的时候`instance = new Singleton()`，不是个原子操作，有两步：
		- 调用构造函数
		- 赋值给instance
	- 异常情况：先赋值给instance，再调用构造函数，![[Pasted image 20251202092844.png]]
		- 在这个窗口期，别的线程判断instance不为null，但是取到和使用的是未完成初始化的instance
- 解决方法：对instance使用volatile关键词修饰，确保与instance有关的操作不发生指令重排
```java
public class Singleton {
    private static volatile Singleton instance; // 在JDK5之前，这里通常没有volatile关键字
    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查：避免不必要的同步
            synchronized (Singleton.class) { // 加锁
                if (instance == null) { // 第二次检查：确认实例尚未创建
                    instance = new Singleton(); // 问题根源所在
                }
            }
        }
        return instance;
    }
}
```