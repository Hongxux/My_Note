---
aliases:
  - 海量数据插入
---
- 需求背景：当需要插入数十万甚至上百万条数据时，使用多条 `INSERT`语句性能会非常低下。
	- 数据迁移或初始化
	- 定期从外部系统（如数据仓库）批量同步数据
- 解决措施： `LOAD DATA`命令。(优化多个数量级)
	- ​**减少开销**​：它将所有数据打包在一次操作中，极大减少了 SQL 解析、事务处理、日志写入等开销。数据库服务器只需对一条 `LOAD DATA`语句进行语法解析、优化和执行
	- ​**直接加载**​：相比执行数万条 SQL 语句，它更接近于直接向数据文件加载数据。
  
---
- 使用方式：**开启功能 -> 指定文件 -> 匹配格式**
	1. ​**客户端连接时启用本地文件加载能力**​
	    在通过命令行连接 MySQL 服务器时，必须加上 `--local-infile`参数。
	    ```
	    mysql --local-infile -u root -p
	    ```
	2. ​**服务器端开启全局开关**​
	    成功连接后，需要在 MySQL 中设置一个全局变量，开启从本地加载文件的功能。
	    ```
	    SET GLOBAL local_infile = 1;
	    ```
	3. ​**执行 LOAD DATA 指令**​
	    这是最核心的一步，指令格式如下：
	    ```
	    LOAD DATA LOCAL INFILE '本地数据文件路径'
	    INTO TABLE 目标表名
	    FIELDS TERMINATED BY '字段分隔符'  -- 图中为逗号 ','
	    LINES TERMINATED BY '行分隔符';   -- 图中为换行符 '\n'
	    ```
	    - ​`LOCAL INFILE '/root/sql.log'`：指定数据文件在客户端机器上的路径。
	    - ​`INTO TABLE 'tb_user'`*：指定数据要插入的目标表。
	    - ​`FIELDS TERMINATED BY ','`​：指明数据文件中每个字段是用什么符号分隔的（图中是逗号分隔的 CSV 格式）。
	    - ​`LINES TERMINATED BY '\n'`：指明每一行记录是用什么符号分隔的（图中是换行符 `\n`）。
- 充分发挥性能的最佳实践
	- 预处理索引
		- 在导入前**禁用非唯一索引**：`ALTER TABLE table_name DISABLE KEYS;`
		- 在导入后再开启：`ALTER TABLE table_name ENABLE KEYS;`
		- 对于 InnoDB 表，还可以在导入前设置 `SET unique_checks=0;`来临时关闭唯一性校验，导入后再恢复为 1
	- 调整系统参数
		- 增大 `bulk_insert_buffer_size`以提升批量插入的内存缓冲区大小
		- 将 `innodb_flush_log_at_trx_commit`设置为 0 或 2，可以显著减少日志刷盘的频率