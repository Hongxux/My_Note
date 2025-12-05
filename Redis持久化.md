## 两种持久化
![[Pasted image 20251122170643.png]]
### RDB持久化（fork操作和刷盘的性能消耗）
 ![[Pasted image 20251122161527.png]]
 RDB持久化的时机:
 - 命令行主动持久化![[Pasted image 20251122162420.png]]
 - 内部**触发**RDB持久化![[Pasted image 20251122162530.png]]![[Pasted image 20251122162659.png]]
 - 主动关闭Redis服务器的时候持久化![[Pasted image 20251122162515.png]]
 - 第一次主从同步进行全量同步的时候
RDB的原理
![[Pasted image 20251122165057.png]]
  异步执行
  1. 
	 - 问题:如果备份进程在进行读操作的时候，主进程写内存，那会出现脏数据
	 - 解决方案：使用copy-on-write技术
		 - 主进程写的时候：拷贝一份数据副本，进行写操作
		 - 主进程读的时候
			- 如果是被拷贝过的数据，则读取该数据副本
			- 如果是未被拷贝过的数据，则读取原数据即可
2. 
	- 问题：如果出现备份的时候，有大量主进程写内存的操作，则需要备份的数据极大，可能出现内存溢出
	- 解决方案：预留一部分内存
  
RDB的缺点
- RDB执行间隔时间长，两次RDB之间写入数据有丢失的风险
- fork子进程、压缩、写出RDB文件都比较耗时
### AOF持久化（aof和rewrite的性能消耗）

AOF全称为Append Only File(追加文件)。Redis处理的每一个写命令都会记录在AOF文件，可以看做是命令日志文件。
![[Pasted image 20251122165540.png]]

开启AOF，配置AOF重写模式 

![[Pasted image 20251122165801.png]]
![[Pasted image 20251122165923.png]]





![[Pasted image 20251122170216.png]]

![[Pasted image 20251122170345.png]]

## 持久化设置
- 用来做缓存的Redis实例尽量不要开启持久化功能
	- 对于分布式锁和库存这类对安全性要求高的开启持久化
- RDB和AOF的选择：
	- 关闭RDB持久功能，使用AOF持久化
		- 数据丢失风险小，数据安全性好
	- 利用脚本在slave节点，实现定期数据备份
		- RDB文件小，适合做备份
- AOF使用设置：
	- 设置合理的rewrite阈值，避免频繁的bgrewrite
		- AOF频繁的写内存和刷盘对cpu和IO性能要求高
	- 配置no-appendfsync-on-rewrite=yes，
		- 含义：禁止在rewrite期间做aof
		- no-appendfsync-on-rewrite=yes的好处：增加可用性![[Pasted image 20251126132535.png]]
			- 如果aof的fsync时间过长，主线程认为出问题的，于是进行主线程阻塞
			- 而做RDB全量同步和rewrite会占用大量IO资源，会导致fsync时长增加
			- 因此为了避免主线程阻塞，要禁止在rewrite期间做aof，避免fsync时间增加
	- no-appendfsync-on-rewrite=yes的坏处：数据丢失的可能
		- 在禁止aof的期间可能导致数据丢失，所以这是性能和数据安全性的权衡
- Redis部署建议：
	- ① Redis实例的物理机要预留足够内存，应对fork和rewrite
	- ②单机多实例部署，单个Redis实例内存上限不要太大，例如4G或8G。
		- 可以加快fork的速度、减少主从同步、数据迁移压力
	- ③ 不要与CPU密集型应用部署在一起
		- 对fork和rewrite的时候对cpu消耗高
	- ④不要与高硬盘负载应用一起部署。例如:数据库、消息队列
		- rewrite和aof都要高磁盘读写
