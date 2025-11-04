## 系统变量

- ​**定义**​：由 ​**MySQL 服务器**提供的配置参数，​**非用户定义**，属于服务器层面。
    
- ​**作用级别**​：分为两种：
    
    - ​**全局变量（GLOBAL）​**​：影响整个 MySQL 服务器实例的全局设置，对所有新建立的会话有效。
        
    - ​**会话变量（SESSION）​**​：仅影响**当前连接的会话(当前命令行)**，会话结束时失效。每个客户端连接都可以有自己独立的会话变量设置。
        
    

### 一、查看系统变量

图片列出了三种查看方法，适用于不同场景。

#### 1. 查看所有变量列表

```
SHOW [SESSION | GLOBAL] VARIABLES;
```

- ​**作用**​：列出所有系统变量及其当前值。
    
- ​**示例**​：
    
    ```
    SHOW GLOBAL VARIABLES;      -- 查看所有全局变量
    SHOW SESSION VARIABLES;     -- 查看所有会话变量（SESSION可省略）
    SHOW VARIABLES;             -- 默认查看当前会话变量
    ```
    

#### 2. 模糊查找特定变量

```
SHOW [SESSION | GLOBAL] VARIABLES LIKE '模糊匹配模式';
```

- ​**作用**​：当你不记得变量全称时，使用通配符 `%`进行筛选。
    
- ​**示例**​：
    
    ```
    SHOW GLOBAL VARIABLES LIKE 'wait_timeout';      -- 精确查找
    SHOW VARIABLES LIKE 'character_set_%';          -- 查找所有字符集相关变量
    SHOW VARIABLES LIKE '%timeout%';                -- 查找包含"timeout"的变量
    ```
    

#### 3. 精确查看指定变量值

```
SELECT @@[SESSION | GLOBAL] 系统变量名;
```

- ​**作用**​：最常用的方法，直接返回单个变量的值，便于在SQL中使用。
    
- ​**示例**​：
    
    ```
    SELECT @@GLOBAL.wait_timeout;        -- 查看全局wait_timeout值
    SELECT @@SESSION.autocommit;         -- 查看当前会话的自动提交设置
    SELECT @@character_set_server;       -- 默认查看SESSION级别
    ```
    

### 二、设置系统变量

图片展示了两种设置语法，效果相同。

#### 标准设置方法

```
SET [SESSION | GLOBAL] 系统变量名 = 值;
SET @@[SESSION | GLOBAL] 系统变量名 = 值;
```

- ​**两种语法等价**，根据个人习惯选择即可。
    
- ​**示例**​：
    
    ```
    -- 设置全局变量（需要SUPER权限）
    SET GLOBAL max_connections = 1000;
    SET @@GLOBAL.max_connections = 1000;
    
    -- 设置会话变量（仅当前连接有效）
    SET SESSION autocommit = 0;
    SET @@SESSION.autocommit = 0;
    SET autocommit = 0;                 -- SESSION可省略
    ```
    

### 三、重要注意事项

1. ​**权限要求**​：
    
    - 设置 ​**GLOBAL**​ 全局变量通常需要 `SUPER`权限。
        
    - 设置 ​**SESSION**​ 会话变量一般用户均可操作。
        
    
2. ​**作用范围**​：
    
    - ​**GLOBAL**​ 设置对**新建的会话**生效，当前已存在的会话不受影响。
        
    - ​**SESSION**​ 设置立即对**当前连接**生效，且只影响当前连接。
        
    
3. ​**持久化**​：
    
    - 通过 `SET`命令修改的变量在**MySQL服务重启后会失效**。
        
    - 如需永久生效，必须修改MySQL配置文件（如 `my.cnf`或 `my.ini`）。
        
    

### 四、常用系统变量实例

|变量名|级别|作用|示例设置|
|---|---|---|---|
|`wait_timeout`|全局/会话|非交互连接超时时间（秒）|`SET GLOBAL wait_timeout=300;`|
|`autocommit`|会话|是否自动提交事务|`SET autocommit=0;`|
|`character_set_server`|全局|服务器默认字符集|`SET GLOBAL character_set_server='utf8mb4';`|
|`max_connections`|全局|最大连接数|`SET GLOBAL max_connections=500;`|
|`sql_mode`|全局/会话|SQL语法校验模式|`SET SESSION sql_mode='STRICT_TRANS_TABLES';`|

### 总结

这张图是管理和优化MySQL服务器的重要参考资料。通过掌握系统变量的查看和设置方法，您可以：

- 优化数据库性能参数
    
- 调整连接和内存配置
    
- 设置字符集、时区等运行环境
    
- 临时修改会话行为满足特定需求
    

正确使用系统变量是MySQL数据库管理和性能调优的基础技能。如果您有具体的配置需求，我可以提供更详细的应用建议！

## 用户定义变量

用户定义变量的核心特性是：

- ​**由用户自定义**​：根据需要在 **SQL 会话**中临时创建，无需提前声明。
    
- ​**使用简单**​：直接用 `@变量名`的形式即可使用。
    
- ​**作用域**​：仅限于**当前数据库连接（会话）​**。当连接关闭后，变量自动销毁。
    

### 一、变量的赋值方法

数据类型自适应**​：变量类型由赋值表达式动态决定。

#### 1. ​**使用 SET 语句（推荐）​**​
    
    ```
    -- 方式1：使用 = 赋值
    SET @var_name = expr [, @var_name2 = expr2, ...];
    
    -- 方式2：使用 := 赋值（更明确，避免与比较运算符混淆）
    SET @var_name := expr [, @var_name2 := expr2, ...];
    ```
    
    ​**示例**​：
    
    ```
    SET @total_count = 100;
    SET @user_name := '张三', @user_age := 25;
    ```
    
#### 2. ​**使用 SELECT 语句赋值**（查询的结果作为赋的值）​
    
    ```
    -- 方式1：直接赋值
    SELECT @var_name := expr [, @var_name2 := expr2, ...];
    
    -- 方式2：从查询结果中赋值（重要用途！）
    SELECT 字段名 INTO @var_name FROM 表名 [WHERE 条件];
    ```
    
    ​**示例**​：
    
    ```
    -- 直接赋值
    SELECT @max_salary := MAX(salary) FROM employees;
    
    -- 从查询结果赋值（确保只返回一行）
    SELECT name, salary INTO @emp_name, @emp_salary 
    FROM employees WHERE id = 1001;
    ```
    

### 二、变量的使用

赋值后，通过简单的 SELECT 语句即可查看变量值：

```
SELECT @var_name;
```

### 实际应用示例

结合图片内容，以下是一些典型应用场景：

```
-- 1. 存储计算结果
SET @page_size = 20;
SET @page_num = 3;
SET @offset = (@page_num - 1) * @page_size;

SELECT * FROM products LIMIT @offset, @page_size;

-- 2. 保存聚合结果
SELECT @total_users := COUNT(*) FROM users;
SELECT @avg_salary := AVG(salary) FROM employees;

-- 3. 跨语句传递值
SELECT @last_id := MAX(id) FROM orders;
INSERT INTO order_logs (order_id, action) VALUES (@last_id, 'created');

-- 4. 查看所有会话变量（调试用）
SELECT @page_size, @page_num, @offset;
```





## 局部变量


> ​**​“局部变量是根据需要定义的在局部生效的变量，访问之前，需要DECLARE声明。可用作存储过程内的局部变量和输入参数，局部变量的范围是在其内声明的BEGIN...END块。”​**​

​**关键特性解读：​**​

1. ​**需要显式声明**​：使用前**必须**使用 `DECLARE`语句声明，不能像用户变量（@var）那样直接使用。
    
2. ​**作用域受限**​：变量的**生命周期和可见性**严格限定在声明它的 `BEGIN ... END`代码块内部。离开这个块，变量就失效了。
    
3. ​**主要应用场景**​：主要用于**存储过程、函数**或**触发器**的内部，作为临时容器存放中间计算结果。

4. **声明位置**：在它的 `BEGIN ... END`代码块内部的开始部位，而且游标变量声明应该在普通变量声明之后，条件处理程序的声明应该在游标之后（普通变量->游标变量->条件处理程序）
    

#### 一、 局部变量的声明

图中展示了声明语法：

```
DECLARE 变量名 变量类型 [DEFAULT 默认值];
```

- ​**变量类型**​：可以是任何标准的 MySQL 数据类型，如 `INT`, `VARCHAR(100)`, `DATE`, `DECIMAL(10,2)`等，与数据表的字段类型一致。
    
- ​**默认值**​（可选）：可以使用 `DEFAULT`关键字为变量指定一个初始值。如果省略，变量初始值为 `NULL`。
    

​**示例：​**​

```
DECLARE total_count INT DEFAULT 0; -- 声明一个整数变量，初始值为0
DECLARE customer_name VARCHAR(50); -- 声明一个字符串变量，初始为NULL
DECLARE order_date DATE; -- 声明一个日期变量
```

​**重要规则**​：在存储过程中，所有的 `DECLARE`语句必须集中放在 `BEGIN`语句之后的**最前面**，在其他任何执行语句之前。

#### 二、 局部变量的赋值

图为变量赋值提供了三种常用方法：

1. ​**使用 `SET`赋值（最常用）​**​
    
    ```
    SET 变量名 = 表达式;
    SET 变量名 := 表达式; -- 使用 := 赋值运算符也是允许的
    ```
    
    ​**示例：​**​
    
    ```
    SET total_count = 100;
    SET total_count = total_count + 1; -- 可以进行运算
    SET customer_name = '张三';
    ```
    
2. ​**使用 `SELECT ... INTO`赋值（从查询结果中获取值）​**​
    
    ```
    SELECT 字段名 INTO 变量名 FROM 表名 WHERE 条件;
    ```
    
    ​**示例：​**​
    
    ```
    -- 从查询结果中获取值，确保查询只返回一行一列
    SELECT COUNT(*) INTO total_count FROM users;
    SELECT name INTO customer_name FROM customers WHERE id = 101;
    ```
    
    ​**注意**​：`SELECT ... INTO`语句必须确保只返回**单行结果**，否则会出错。
    

### 一个完整的存储过程示例

结合图中的所有要点，一个使用局部变量的典型存储过程如下：

```
DELIMITER $$
CREATE PROCEDURE CalculateOrderStats(IN order_id INT)
BEGIN
    -- 1. 声明局部变量（必须在BEGIN后的最前面）
    DECLARE item_count INT DEFAULT 0;
    DECLARE total_amount DECIMAL(10,2) DEFAULT 0.0;
    DECLARE avg_price DECIMAL(10,2) DEFAULT 0.0;
    DECLARE customer_name VARCHAR(100);

    -- 2. 使用SET赋值
    SET item_count = 0; -- 初始化

    -- 3. 使用SELECT...INTO从查询中赋值
    SELECT COUNT(*), SUM(price * quantity)
    INTO item_count, total_amount
    FROM order_items WHERE oid = order_id;

    -- 4. 进行计算赋值
    IF item_count > 0 THEN
        SET avg_price = total_amount / item_count;
    END IF;

    -- 5. 获取客户名
    SELECT c.name INTO customer_name
    FROM orders o JOIN customers c ON o.cid = c.id
    WHERE o.id = order_id;

    -- 6. 返回结果（示例）
    SELECT item_count AS '商品数量',
           total_amount AS '总金额',
           avg_price AS '平均单价',
           customer_name AS '客户姓名';
END
$$
DELIMITER ;
```

#### [[游标变量]]