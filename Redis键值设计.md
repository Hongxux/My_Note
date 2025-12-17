### Key的设置
Redis的Key虽然可以自定义，但最好遵循下面的几个最佳实践约定
- 遵循基本格式:`[业务名称]:[数据名]:[id]`![[Pasted image 20251125161157.png]]
	- 可读性强：分层的模式
	- 避免key冲突
	- 方便管理
- 长度不超过44字节
	- 更节省内存: key是string类型，底层编码包含int、embstr和raw三种。embstr在小于44字节使用，采用连续内存空间，内存占用更小
- 不包含特殊字符
  
  
### [[BigKey]]

### 数据类型选择
#### 1.储存多字段对象使用hash
![[Pasted image 20251125165744.png]]
- JSON字符串
	- 缺点：数据耦合，修改不够方便
		- 修改部分数据必须要重写整个JSON
- 字段打散
	- 缺点：
		- 占用空间大：存储key和value要额外保存元信息
		- 没有办法做到统一控制（无法一口气获取user的所有信息）
- hash结构
	- 优点：
		- 可以灵活访问对象的任意字段
		- 空间占用小：底层使用ziplist
			- 但是hash的entry超过500会使用hash表，而不是ziplist，导致内存占用多
			- 可以通过`hash-max-ziplist-entries`调整上限，但是会出现BigKey问题
	- 缺点：
		- 代码相对复杂
			- 解决方法：封装工具类
#### 2.避免大量字段的hash
![[Pasted image 20251125170536.png]]
- 存在的问题：
	- hash的entry超过500会使用hash表，而不是ziplist，导致内存占用多
	- 可以通过`hash-max-ziplist-entries`调整上限，但是会出现BigKey问题![[Pasted image 20251125170712.png]]
	- 如果拆开字段成String类型：
		- string结构底层没有太多内存优化，内存占用较多。
			- 一百万数据，hash占用60MB
			- 一百万数据，String占用70MB
		- 想要批量获取这些数据比较麻烦
- 解决方案：把大的hash表根据字段id拆成小hash，而每个字段的分配方式如下（每100元素一个hash表）：![[Pasted image 20251125171406.png]]
	- key的值：id/100
	- filed的值：id%100
- 优化效果：变成24MB