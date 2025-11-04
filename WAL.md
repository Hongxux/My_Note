**修改操作发生时，会先生成redo log并写入内存中的redo log buffer；当事务提交时，会立即将这些redo log从内存强制写入磁盘的redo log file中(原子性操作)，确保持久性，之后才返回提交成功。**
