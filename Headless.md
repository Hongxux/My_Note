 - **是什么？**：Headless 模式是 Java 的一种运行模式，它允许在**没有物理显示设备、键盘或鼠标**的环境下（如服务器、命令行程序）运行需要图形操作的代码。
    
- **为什么需要？**：一些第三方库（如图表生成库、旧版的 AWT/Swing 组件）可能会尝试访问图形设备。在服务器环境下，这会导致 `java.awt.HeadlessException`异常。
    
- **Spring Boot 的做法**：图2中的代码确保系统属性 `java.awt.headless`被设置为 `true`。
    
    ```
    System.setProperty("java.awt.headless", 
        System.getProperty("java.awt.headless", "true")); // 默认值设为true
    ```
    
- **结论**：**Headless 模式的作用是“模拟”一个虚拟的图形环境，避免因服务器缺少真实图形设备而导致的启动失败**，确保应用在无界面环境中稳定运行。
    