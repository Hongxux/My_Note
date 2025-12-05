---
aliases:
  - 对象头
---
- 定位：JAVA的一个对象由两部分组成，一个是对象头，一个是对象的成员变量
- 组成内容：
	- Mark Word：不同的状态（锁状态）下不同的内容成分![[Pasted image 20251130211335.png]]
	- Klass Word：指针，指向对象从属的类对象
	- Array Length：数组长度，仅数组对象有