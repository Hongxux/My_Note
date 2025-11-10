`EXPLAIN`是 MySQL 提供的强大工具，用于**模拟 MySQL 优化器如何执行一条 SQL 查询语句**。通过分析它的返回结果，可以了解查询的执行细节，从而对查询语句或数据库结构进行优化。

学会解读其中关键字段（尤其是 ​**type, key, rows, Extra**）的含义，是提升数据库查询性能的关键第一步。

---

#### 核心语法与用法

使用方法极其简单：在需要分析的 `SELECT`语句前加上 `EXPLAIN`或 `DESC`关键字即可。

```
-- 两种方式等效
EXPLAIN SELECT 字段列表 FROM 表名 WHERE 条件；

-- 或者
DESC SELECT 字段列表 FROM 表名 WHERE 条件；
```

#### 执行计划字段解析（关键）

执行 `EXPLAIN`后，MySQL 会返回一个表格，包含若干重要字段。图中示例 `EXPLAIN SELECT * FROM tb_user WHERE id = 1;`的结果展示了这些字段的值，它们是分析性能的关键：

| 字段                  | 含义              | 解读与重要性                                                                                                                                 |
| ------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| ​**id**​            | 查询的序列号          | 表示查询中执行 `SELECT`子句或操作表的顺序。id 越大，优先级越高。                                                                                                 |
| ​**select_type**​   | 查询的类型           | 常见值：`SIMPLE`（简单查询，无子查询或UNION）、`PRIMARY`（最外层查询）、`SUBQUERY`（子查询）等。用于判断查询的复杂度。                                                            |
| ​**table**​         | 正在访问的表名         | 显示数据来自于哪张表。                                                                                                                            |
| ​**partitions**​    | 匹配的分区           | 如果表创建了分区，此字段显示查询命中的分区。                                                                                                                 |
| ​**type**​          | ​**访问类型**​      | ​**这是判断查询效率的极其重要的指标！​**​ 常见值（性能从优到劣）：`system`> `const`（主键访问、唯一索引查询）> `eq_ref`> `ref`> `range`> `index`> `ALL`（全表扫描，需优化）。               |
| ​**possible_keys**​ | 可能使用的索引         | 显示MySQL能使用哪些索引来查找记录。                                                                                                                   |
| ​**key**​           | ​**实际使用的索引**​   | 显示MySQL**最终决定**使用的索引。如果为 `NULL`，则表示未使用索引。                                                                                              |
| ​**key_len**​       | 索引使用的字节数        | 表示索引中使用的字节数，可用于判断索引是否被完全利用。在不损失精度的情况下越短越好                                                                                              |
| ​**ref**​           | 索引的引用           | 显示使用索引时，是与哪些列或常量进行比较。                                                                                                                  |
| ​**rows**​          | ​**预估需要读取的行数**​ | 一个非常关键的指标。MySQL 估算找到所需记录需要读取的行数。​**这个值越小越好**。                                                                                          |
| ​**filtered**​      | 按条件过滤后剩余行的百分比   | 表示存储引擎返回的数据在服务器层过滤后，剩下多少满足查询。表示返回的结果占需读取函数的百分比，值越大越好                                                                                   |
| ​**Extra**​         | ​**额外信息**​      | 包含不适合在其他列显示但十分重要的额外信息。如 `Using where`（使用WHERE过滤）、`Using index`（使用了覆盖索引，性能好）、`Using temporary`（使用临时表，需优化）、`Using filesort`（使用文件排序，需优化）。 |

#### 如何使用 EXPLAIN 进行性能优化？

1. ​**查看是否使用索引**​：检查 `key`字段。如果为 `NULL`，说明是**全表扫描**，需要为查询条件建立索引。
    
2. ​**检查索引使用效率**​：查看 `type`字段。如果值为 `ALL`或 `index`，通常意味着性能不佳，需要考虑优化索引或查询语句。
    
3. ​**评估查询成本**​：关注 `rows`字段。这个数值越大，说明查询需要扫描的数据越多，性能越差。
    
4. ​**发现潜在问题**​：查看 `Extra`字段。如果出现 `Using temporary`或 `Using filesort`，尤其是在大表查询中，这通常是性能瓶颈的信号，需要优化。
非常好！`EXPLAIN`是 MySQL 性能优化的**核心诊断工具**。它揭示了 SQL 查询的执行计划，让我们能够"看到"数据库是如何处理查询的。下面我为您构建一个系统化的 **EXPLAIN 优化框架**。

---

### 根据explain优化
#### 第一步：理解 EXPLAIN 的关键字段

首先，要关注以下几个**核心字段**：

|字段|重要性|理想值/含义|
|---|---|---|
|**type**|⭐⭐⭐⭐⭐|表示**访问类型**，从好到坏：`system > const > eq_ref > ref > range > index > ALL`|
|**key**|⭐⭐⭐⭐⭐|实际使用的索引，`NULL`表示未使用索引|
|**rows**|⭐⭐⭐⭐|预估需要扫描的行数，**越小越好**|
|**Extra**|⭐⭐⭐⭐⭐|**额外信息**，包含优化的重要线索|
|**possible_keys**|⭐⭐⭐|可能使用的索引，如果为空说明缺少合适索引|

##### 第二步：针对不同问题的具体优化策略

现在，我们根据流程图中的诊断结果，给出具体的优化方案。

##### 1. 当 `type = ALL`（全表扫描）时 → **最紧急问题**

**问题分析**：MySQL 正在逐行扫描整个表，性能极差。

**优化方案**：

```
-- 诊断：未使用索引
EXPLAIN SELECT * FROM users WHERE age > 25;

-- 优化：为查询条件创建索引
ALTER TABLE users ADD INDEX idx_age (age);
-- 或创建联合索引（如果还有其他条件）
ALTER TABLE users ADD INDEX idx_age_name (age, name);
```

##### 2. 当 `type = index`（全索引扫描）时 → **需要优化**

**问题分析**：虽然使用了索引，但是扫描了整个索引树，类似于全表扫描。

**优化方案**：

```
-- 诊断：扫描了整个索引
EXPLAIN SELECT id FROM users ORDER BY id;

-- 优化：确保查询能利用索引的过滤性
-- 添加WHERE条件限制扫描范围
SELECT id FROM users WHERE status = 1 ORDER BY id;
-- 相应创建索引
ALTER TABLE users ADD INDEX idx_status_id (status, id);
```

##### 3. 当 `Extra = Using filesort`时 → **排序性能问题**

**问题分析**：MySQL 需要额外的排序操作，通常因为 `ORDER BY`或 `GROUP BY`没有使用索引。

**优化方案**：

```
-- 诊断：需要文件排序
EXPLAIN SELECT * FROM users ORDER BY create_time DESC;

-- 优化：为排序字段创建索引
ALTER TABLE users ADD INDEX idx_create_time (create_time);

-- 更优：如果还有查询条件，创建联合索引
ALTER TABLE users ADD INDEX idx_status_create_time (status, create_time);
-- 查询时确保遵循最左前缀原则
SELECT * FROM users WHERE status = 1 ORDER BY create_time DESC;
```

##### 4. 当 `Extra = Using temporary`时 → **临时表问题**

**问题分析**：MySQL 需要创建临时表来处理查询，常见于复杂的 `GROUP BY`、`DISTINCT`、`UNION`等操作。

**优化方案**：

```
-- 诊断：需要临时表
EXPLAIN SELECT DISTINCT department FROM users;

-- 优化：为分组字段创建索引
ALTER TABLE users ADD INDEX idx_department (department);

-- 对于复杂分组，创建覆盖索引
EXPLAIN SELECT department, COUNT(*) FROM users GROUP BY department;
ALTER TABLE users ADD INDEX idx_department (department);
```

##### 5. 追求 `Extra = Using index`（覆盖索引）→ **最优效果**

**目标**：让查询所需的所有数据都从索引中获取，避免回表操作。

**优化方案**：

```
-- 普通查询（需要回表）
EXPLAIN SELECT id, name, email FROM users WHERE age > 25;

-- 优化为覆盖索引查询
ALTER TABLE users ADD INDEX idx_age_covering (age, name, email);
-- 现在查询只需要扫描索引，无需回表
EXPLAIN SELECT name, email FROM users WHERE age > 25;
```

#### 第三步：高级优化技巧

##### 1. 索引选择性优化

```
-- 检查索引的选择性（区分度）
SELECT 
    COUNT(DISTINCT status) / COUNT(*) as selectivity_status,
    COUNT(DISTINCT age) / COUNT(*) as selectivity_age
FROM users;

-- 选择性高的列更适合创建索引（越接近1越好）
```

##### 2. 联合索引顺序优化

```
-- 正确的联合索引顺序：高选择性列在前，等值查询在前
-- 推荐：(status, create_time) 而不是 (create_time, status)
ALTER TABLE orders ADD INDEX idx_status_created (status, create_time);

-- 这样能有效利用索引进行范围查询
SELECT * FROM orders WHERE status = 'paid' AND create_time > '2023-01-01';
```

##### 3. 避免索引失效的常见陷阱

```
-- 1. 不要在索引列上使用函数
-- 错误：索引失效
SELECT * FROM users WHERE YEAR(create_time) = 2023;
-- 正确：使用范围查询
SELECT * FROM users WHERE create_time BETWEEN '2023-01-01' AND '2023-12-31';

-- 2. 避免类型转换
-- 错误：phone是varchar，但用了数字比较
SELECT * FROM users WHERE phone = 13800138000;
-- 正确：保持类型一致
SELECT * FROM users WHERE phone = '13800138000';
```
