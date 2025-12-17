---
aliases:
  - 分区
---
G1把堆化整为零，分为2048 个 region（默认 1~32MB）
- 逻辑分类：
	- Eden
	- Survivor
	- Old
	- Humongous（大对象）
- 好处：
	- 内存管理更加灵活
	- 实现部分回收的基础
- `-XX:G1HeapRegionSize=n`可指定分区大小(1MB~32MB，且必须是2的幂)，默认将为2048。