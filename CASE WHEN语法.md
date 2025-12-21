- 需求背景：
	- 数据解码与转换：将存储时为了效率而采用的代码（如1/0）转换为业务含义明确的值（如“男”/“女”）
	- 动态分类与打标签：根据一定的范围或规则（如分数区间、金额大小）为每行数据创建一个新的分类标签
	- 实现条件聚合：在同一查询中，根据不同条件对不同的行子集进行统计，是生成透视报表的关键
	- 辅助复杂条件过滤：在 `WHERE`、`ORDER BY`等子句中构建动态条件，实现更灵活的查询控制
- 解决措施：CASE语句
	- **数据清洗与标准化**：在数据仓库或数据分析之前，经常需要清理和标准化数据。例如，将同一含义的不同表达（如“Male”、“M”、“男”）统一为“男性”。`CASE`语句可以轻松实现这种映射和转换。
	- **生成业务报表**：这是 `CASE`语句大显身手的领域。通过**条件聚合**，可以轻松实现经典的“行转列”或透视表功能。例如，统计每个销售人员在不同产品类别上的销售额。
	- **控制数据更新逻辑**：在 `UPDATE`语句中，可以根据不同的条件更新为不同的值，避免编写多条更新语句。例如，根据不同会员等级设置不同的折扣率。
- 语法：
	- 用于数据标准化的映射：用在列中
		```sql
		SELECT 
		    employee_id,
		    name,
		    salary,
		    CASE 
		        WHEN salary < 3000 THEN '初级'
		        WHEN salary BETWEEN 3000 AND 7000 THEN '中级'
		        WHEN salary > 7000 THEN '高级'
		        ELSE '待定'
		    END AS salary_grade
		FROM employees;	
		```
	- 条件统计：结合 `GROUP BY`对数据进行条件分组和统计
		```sql
		SELECT 
		    CASE 
		        WHEN salary < 3000 THEN '初级'
		        WHEN salary BETWEEN 3000 AND 7000 THEN '中级'
		        WHEN salary > 7000 THEN '高级'
		        ELSE '待定'
		    END AS salary_grade,
		    COUNT(*) AS employee_count
		FROM employees
		GROUP BY 
		    CASE 
		        WHEN salary < 3000 THEN '初级'
		        WHEN salary BETWEEN 3000 AND 7000 THEN '中级'
		        WHEN salary > 7000 THEN '高级'
		        ELSE '待定'
		    END;	
		```
		- 列需要匹配
	- 自定义排序：在 `ORDER BY`中使用 `CASE`语句并返回数字，可以实现特定的排序优先级
		```sql
		SELECT employee_id, name, salary
		FROM employees
		ORDER BY 
		    CASE 
		        WHEN salary > 7000 THEN 1
		        WHEN salary < 3000 THEN 2
		        ELSE 3
		    END;	  
		```
	- 条件更新数据：在 `UPDATE`语句中，`CASE`可以根据不同条件设置不同的值。
		```sql
		UPDATE employees
		SET bonus = 
		    CASE 
		        WHEN salary > 7000 THEN salary * 0.1  -- 高级薪资，奖金10%
		        WHEN salary < 3000 THEN salary * 0.15 -- 初级薪资，奖金15%
		        ELSE salary * 0.08  -- 中级薪资，奖金8%
		    END;	
		```