- 需求背景：需要进行深分页
- 问题分析：
	 当执行 `SELECT * FROM table ORDER BY id LIMIT 10000, 10`时，数据库的底层操作是：
	1. **读取数据**：根据索引（或全表扫描）读取前 `10000 + 10`条数据。
	2. **排序丢弃**：对这10010条数据进行排序（如果使用 `ORDER BY`），然后丢弃前10000条。
	3. **返回结果**：最终只返回最后的10条。
- 性能瓶颈：巨大的 `OFFSET`值导致数据库需要**物理地扫描和跳过大量数据**，即使最后只返回很少的结果。这造成了CPU和I/O资源的极大浪

---

- 优化策略：
	- **书签记录法**
		- 优化思路：避免使用 `OFFSET`，用上一页最后一条记录的ID作为起点
		- 实现方式：
			1. 前端或后端记录当前页最后一条记录的ID（例如 `last_id`）
			2. 查询下一页时，将 `last_id`作为条件传递给后端，以此作为查询下一页的起点，通过where和主键索引有序性
				```
				SELECT * FROM orders WHERE id > 10000 ORDER BY id LIMIT 10;			
				```
	- 覆盖索引优化
		- 优化思路：如果查询的列都包含在某个索引中，数据库可以直接从索引中获取所需数据，而无需访问真实的数据行
		- 实现方式：
			1. 利用覆盖索引快速找出满足条件的10条数据的主键ID
			2. 再通过主键ID关联回原表取出所有列
				```
				-- 快速查询：创建覆盖索引 (status, create_time, id, name, price)
				-- 然后使用子查询方式，先利用覆盖索引快速定位到主键ID
				SELECT * FROM products AS p1
				JOIN (
				    SELECT id FROM products 
				    WHERE status = 1 
				    ORDER BY create_time DESC 
				    LIMIT 10000, 10
				) AS p2 ON p1.id = p2.id;			
				```