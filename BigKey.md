- BigKey的界定：key的大小和value每个成员的大小和成员数量
	- ![[Pasted image 20251125162445.png]]
	- 估算长度：
		- 可以使用`STRLEN key`查看String类型值的长度
			- 建议小于10KB
		- 可以使用类似于List的`LLEN key`查看成员的个数
			- 建议小于1000
- BigKey的危害![[Pasted image 20251125163029.png]]
- BigKey的检测
	-  **redis-cli --bigkeys**
		- **限制**：只能返回每种数据结构的Top1大键，采样可能不全面
	- **SCAN扫描+自定义分析（推荐方案）**
		- scan扫描，逐个扫描，不会阻塞主线程![[Pasted image 20251125163707.png]]
		- 不使用MOMERTY USAGE：虽然`MEMORY USAGE key`命令能返回精确的内存占用，但它需要遍历整个数据结构，对大型集合可能造成性能抖动。
		- [[自定义分析BigKeys程序]]
	- **Redis-Rdb-Tools（第三方深度分析）**
			```
			# 安装
			pip install rdbtools
			
			# 生成内存分析报告
			rdb -c memory dump.rdb --bytes 1024 --largest 20 > memory_report.csv
			
			# 分析特定模式的大键
			rdb -c memory dump.rdb --key-pattern "user:*" > user_memory.csv
			```
		- 原理：分析RDB快照
			- 好处：离线扫描
			- 坏处：存在时效性问题
	- 网络监控
- BigKey的解决
	- 对数据拆分后重新存储，再删除BigKey
		- 删除：`unlike`异步删除，避免阻塞主线程![[Pasted image 20251125164843.png]]
	- 选择合适的数据类型，避免BigKey