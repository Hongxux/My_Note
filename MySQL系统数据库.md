

这四个数据库由 MySQL 自动创建和维护，存储着数据库服务器本身的元数据、运行状态和权限信息。理解它们对于数据库管理、性能优化和问题排查至关重要。

|数据库名称|核心职责|详细解释与典型用途|
|---|---|---|
|​**`mysql`**​|​**安全与核心配置库**​|这是 MySQL 的“心脏”，存储了所有**用户账户、权限分配、主从复制信息、时区、存储过程和事件定义**等关键数据。​**严禁随意修改**，除非使用如 `CREATE USER`、`GRANT`这样的专用管理命令。|
|​**`information_schema`**​|​**元数据查询接口**​|提供了以只读的**视图**方式访问数据库的元信息，如：有哪些数据库、表、列、索引、约束等。它遵循 SQL 标准，是**跨平台**查询数据库结构的首选。|
|​**`performance_schema`**​|​**底层性能监控器**​|专注于收集数据库服务器运行时的**底层性能指标**，如文件 I/O、锁等待、内存使用、线程活动等。它像一个内置的“性能分析仪”，用于深度诊断性能瓶颈。|
|​**`sys`**​|​**性能调优仪表盘**​|建立在 `performance_schema`之上，通过一系列易用的**视图、函数和存储过程**，将复杂的底层性能数据转化为人类可读的格式。它极大地简化了性能分析工作，是 DBA 进行日常调优的“瑞士军刀”。|

### 各数据库之间的关系与使用场景

#### 针对不同角色的使用建议：

|您的角色|最常用的数据库|典型查询示例|
|---|---|---|
|​**数据库管理员（DBA）​**​|`mysql`, `sys`, `performance_schema`|​**创建用户**​：`CREATE USER 'user'@'host' IDENTIFIED BY 'password';`(操作于 `mysql`库)  <br>​**查看锁等待**​：`SELECT * FROM sys.innodb_lock_waits;`(查询 `sys`库)|
|​**开发人员**​|`information_schema`|​**查看表结构**​：`SELECT COLUMN_NAME, DATA_TYPE FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'users';`|
|​**性能优化工程师**​|`sys`, `performance_schema`|​**查看最耗资源的SQL**​：`SELECT * FROM sys.statement_analysis ORDER BY avg_latency DESC LIMIT 10;`|

### 总结

这张图是理解 MySQL 数据库自我管理机制的钥匙。简单来说：

- 想**管理用户权限**，你需要与 `mysql`库打交道。
    
- 想**查询数据库里有什么表、什么字段**，就去查 `information_schema`。
    
- 当数据库**反应慢**，需要找出瓶颈时，`sys`库是你的第一站，`performance_schema`则用于深度分析。
    

掌握这四个系统数据库的用途，是进行高效的数据库管理、开发和性能优化的基础。