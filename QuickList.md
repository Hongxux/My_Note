- 需求背景：
	 - LinkedList问题：
		- 内存开销大，不紧凑：由于其存储的元数据导致
			- 每个节点都需要存储前后指针，在64位系统占用16字节额外开销，内存碎片多。
		- 遍历性能差：节点内存不连续，CPU缓存不友好，遍历效率低。
	- ZipList的问题：
		- 大块连续内存分配难：如果数据量大时，申请大块连续内存困难，修改时可能需整体重新分配。
			- 原来的ZipList所有元素存储在连续内存中
		- 连锁更新问题
- 解决方案：QuickList：LinkedList+ZipList取长补短
	- 一个由双向链表连接起来的多节点结构，而每个节点本身就是一个小的 ZipList。
		- 数据分片提高了存储的容量，也缩小了修改的影响范围
		- 多个ZipList由双向链表联系
- QuickList数据结构：![[Pasted image 20251126205352.png]]![[Pasted image 20251126205549.png]]
	
- 配置QuickList
	- 限制每个ZipList的entry数量:list-max-ziplist-size来限制。
		- 如果值为正，则代表ZipList的允许的entry个数的最大值
		- 如果值为负，则代表ZipList的最大内存大小，分5种情况:
				① -1:每个ZipList的内存占用不能超过4kb
				②-2:每个ZipList的内存占用不能超过8kb
				③ -3:每个ZipList的内存占用不能超过16kb
				④-4:每个ZipList的内存占用不能超过3zkb
				⑤ -5:每个ZipList的内存占用不能超过64kb
		- 其默认值为 -2:
	- 控制ZipList压缩的深度：通过配置项list-compress-depth来控制。因为链表般都是从首尾访问较多，所以首尾是不压缩的。
		- 这个参数是控制首尾不压缩的节点个数:
				0:特殊值，代表不压缩
				1:标示OuickList的首尾各有1个节点不压缩，中间节点压缩
				2:标示QuickList的首尾各有2个节点不压缩，中间节点压缩
				以此类推
		- 