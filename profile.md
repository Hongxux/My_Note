
`SHOW PROFILE`是 MySQL 提供的一个强大的性能诊断工具，它能够**深入分析一条 SQL 语句在执行过程中各个阶段的详细耗时**，帮助开发者精准定位性能瓶颈。

---

#### 一、启用 Profile 功能

在使用前，需要按以下步骤确保该功能已开启：

1. ​**检查是否支持**​：首先确认你的 MySQL 版本支持此功能。
    
    ```
    SELECT @@have_profiling;
    ```
    
    如果返回结果为 `YES`，则表示支持。
    
2. ​**开启功能**​：Profile 功能默认是关闭的，需要在当前会话中开启。
    
    ```
    -- 在当前会话中开启
    SET profiling = 1;
    -- 或者在全局级别开启（需要权限）
    SET GLOBAL profiling = 1;
    ```
    

#### 二、使用 Profile 分析 SQL 耗时

启用后，遵循以下流程进行分析：

1. ​**执行你的业务 SQL**​：
    
    正常执行你想要分析的 SQL 语句。例如：
    
    ```
    SELECT * FROM orders WHERE customer_id = 123 AND create_date > '2023-01-01';
    ```
    
2. ​**查看 SQL 查询列表与总耗时**​：
    
    使用以下命令查看最近执行的所有 SQL 语句及其查询 ID (`Query_ID`) 和总耗时。
    
    ```
    SHOW PROFILES;
    ```
    
    执行结果示例：
    
|Query_ID|Duration|Query|
|---|---|---|
|1|0.00015000|SELECT * FROM user WHERE id = 1|
|2|0.00234500|SELECT * FROM orders WHERE ... (你的慢SQL)|

    从这里找到你关心的那条 SQL 对应的 `Query_ID`。
    
3. ​**详细分析指定 SQL 的执行阶段**​：
    
    使用获取到的 `Query_ID`进行深入分析。
    
    - ​**查看各阶段耗时**​：这是最常用的命令，可以查看该 SQL 在**每个执行阶段（如解析、优化、执行、锁等待、数据传送等）的耗时**。
        
        ```
        SHOW PROFILE FOR QUERY 2; -- 这里的 2 是上一步查到的 Query_ID
        ```
        ![[Pasted image 20251029212609.png]]
        结果会显示类似 `starting`, `checking permissions`, `Opening tables`, `System lock`, `Sending data`等阶段的耗时，耗时最长的阶段就是主要性能瓶颈。
        
    - ​**查看 CPU 使用情况**​：
        
        ```
        SHOW PROFILE CPU FOR QUERY 2;
        ```
        ![[Pasted image 20251029212632.png]]
        此命令会额外显示各阶段对 CPU 资源的占用情况。
        
    

#### 核心要点与使用场景总结

- ​**目的**​：精准定位慢 SQL 的性能瓶颈到底出现在哪个环节（是优化器问题？锁等待太久？还是数据传送慢？）。
    
- ​**流程**​：`开启功能 -> 执行SQL -> SHOW PROFILES 获取ID -> SHOW PROFILE ... FOR QUERY [ID] 查看详情`。
    
- ​**应用场景**​：在 SQL 优化过程中，当发现某条语句执行缓慢时，使用此工具可以替代盲目猜测，进行有依据的优化。
    

通过以上步骤，你可以像使用“性能分析仪”一样，清晰地了解 SQL 语句执行的内部细节，从而进行有效的优化。
