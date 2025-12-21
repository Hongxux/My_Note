

### SQL通用语法

![[Pasted image 20251028152010.png]]


----

### SQL分类
![[Pasted image 20251028152115.png]]
![[Pasted image 20251028152259.png]]
#### DDL 数据定义语言
##### 关于数据库
- 查询
	- 查询服务器中的所有数据库![[Pasted image 20251028152444.png]]
	  ![[Pasted image 20251028152707.png]]
	- 查询当前数据库（知道自己在使用哪个数据库）
	  ![[Pasted image 20251028152543.png]]
- 创建
  ![[Pasted image 20251028152559.png]]
  ![[Pasted image 20251028152806.png]]
	- 可选项：
		 - if not exists![[Pasted image 20251028152856.png]]
		- defualt charset![[Pasted image 20251028153046.png]]
- 删除
	- ![[Pasted image 20251028153206.png]]
	- ![[Pasted image 20251028153310.png]]
- 使用：
	- ![[Pasted image 20251028153329.png]]
#####  关于表结构
- 查询：
![[Pasted image 20251028153527.png]]

![[Pasted image 20251028154505.png]]
- 创建：
	![[Pasted image 20251028153656.png]]
	![[Pasted image 20251028154239.png]]
	- 注释用**单引号**
	- **[[SQL字段类型]]**
- 修改
	- 字段：
		- 添加字段：![[Pasted image 20251028185652.png]]
		- 修改字段：![[Pasted image 20251028185833.png]]
		- 删除字段·![[Pasted image 20251028190102.png]]
	- 修改表名：![[Pasted image 20251028190130.png]]
- 删除
	- 删除表、重新创建表（表结构清空）![[Pasted image 20251028190308.png]]
	
#### DML 数据操作语言
#####  添加数据（INSERT）
- 指定字段添加数据![[Pasted image 20251028193243.png]]
- 给全部字段添加数据![[Pasted image 20251028193259.png]]
- 批量添加数据![[Pasted image 20251028193336.png]]

- 字符串和日期型数据应该包含在单引号内
##### 修改数据（UPDATE）
![[Pasted image 20251028194111.png]]
##### 删除数据（DELETE）
![[Pasted image 20251028194154.png]]
- 没有条件就是删除这个表的所有数据
- 不可以删除某个字段的值（可以使用UPDATE设置为NULL）

#### DQL 数据查询语言
- DQL 的编写顺序
 ![[Pasted image 20251028194720.png]]
 - DQL的执行顺序
   ![[Pasted image 20251028210926.png]]
##### 基本查询
- 指定返回的字段
![[Pasted image 20251028195022.png]]
- 设置别名
  ![[Pasted image 20251028195100.png]]
- 不要重复，去重
- ![[Pasted image 20251028195129.png]]
##### 条件查询（where）
- 条件
	- 日期也能比较大小
###### 比较运算符

用于比较两个值之间的关系，返回布尔值（TRUE/FALSE）。

|比较运算符|功能说明|示例|
|---|---|---|
|​**>​**​|大于|`WHERE age > 18`|
|​**>=​**​|大于等于|`WHERE score >= 60`|
|​**​<**​|小于|`WHERE price < 100`|
|​**​<=​**​|小于等于|`WHERE quantity <= 10`|
|​**​=​**​|等于|`WHERE name = 'John'`|
|​**​<>​**​ 或 ​**​!=​**​|不等于|`WHERE status != 'inactive'`|
|​**BETWEEN ... AND ...​**​|在指定范围内（包含边界值）|`WHERE age BETWEEN 20 AND 30`|
|​**IN(...)​**​|匹配列表中的任意值|`WHERE category IN ('Electronics', 'Books')`|
|​**LIKE**​|模糊匹配（`_`匹配单个字符，`%`匹配任意字符）|`WHERE name LIKE 'J%'`（匹配J开头）|
|​**IS NULL**​|检查是否为 NULL 值|`WHERE email IS NULL`|

###### 逻辑运算符

用于连接多个条件，组合成复杂的逻辑表达式。

|逻辑运算符|功能说明|示例|
|---|---|---|
|​**AND**​ 或 ​**&&**​|逻辑与（所有条件必须同时成立）|`WHERE age > 18 AND status = 'active'`|
|​**OR**​ 或 ​**||**​|
|​**NOT**​ 或 ​**​!​**​|逻辑非（否定条件）|`WHERE NOT country = 'USA'`|

**关键注意事项**

1. ​**优先级**​：`NOT`> `AND`> `OR`，可使用括号 `()`明确优先级。
    
2. ​**NULL 处理**​：与 `NULL`比较时需使用 `IS NULL`或 `IS NOT NULL`，直接使用 `=`可能返回未知（UNKNOWN）。
    
3. ​**LIKE 通配符**​：`%`匹配零个或多个字符，`_`匹配恰好一个字符。
    
4. ​**BETWEEN 范围**​：范围是闭区间，即 `BETWEEN min AND max`包含 `min`和 `max`。

##### 分组查询（group by）
如何理解分组和聚合搭配：分组相当于根据某个字段，相同的划分成一个组，而聚合函数则是作用于一个个小组（不分组则是作用于整个表这一组），最后根据要求显示的字段列表进行显示。
[[聚合函数]]
![[Pasted image 20251028203006.png]]
1. where和having 区别（where的过滤对象是整个表，不使用聚合语句，having的过滤对象是where过滤得到的表，使用聚合语句）
	1. 执行时间：
		- where：分组前进行过滤，不满足where条件不参与分组，不会被聚合函数包含
		- having:分组之后对结果进行过滤
	2. 判断条件：
		- where：不可以将聚合函数作为条件
		- having：将聚合条件作为语法
2. 分组之后的查询字段通常为分组字段和聚合函数，其他字段无意义
##### 排序查询（order by）
![[Pasted image 20251028204738.png]]
排序方式：
- ASC：升序（默认）
- DESC：降序
支持多字段排序
##### 分页查询（limit）
![[Pasted image 20251028210436.png]]
- 起始索引：从0 开始
	- 计算方法：起始索引 = (查询页数-1)`*`每页显示记录数

#### DCL  数据控制语言
##### 用户管理
##### 权限控制
​

##### ​**1. 授权语句 (GRANT)​**​

```
-- 授予特定权限
GRANT SELECT, INSERT ON database_name.table_name TO 'username'@'host';

-- 授予所有权限
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost';

-- 允许转授权限
GRANT CREATE ON db.* TO 'user'@'%' WITH GRANT OPTION;

-- 授予角色权限
GRANT SELECT_ROLE TO 'analyst'@'10.0.%';
```

##### ​**2. 撤销权限语句 (REVOKE)​**​

```
-- 撤销特定权限
REVOKE DELETE, UPDATE ON db.orders FROM 'user'@'ip';

-- 撤销所有权限
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'old_user'@'%';

-- 撤销角色
REVOKE REPORT_ROLE FROM 'temp_user'@'vpn';
```

##### ​**3. 权限查看语句**​

```
-- 查看当前用户权限
SHOW GRANTS;

-- 查看指定用户权限
SHOW GRANTS FOR 'user'@'localhost';
```
