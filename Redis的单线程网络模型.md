- AE库：高性能库![[Pasted image 20251127155607.png]]
	- 基础：IO多路复用
	- 选择：根据运行的环境选择合适的AE库![[Pasted image 20251127160300.png]]
	- 特点：暴露的对外方法统一![[Pasted image 20251127155751.png]]
-  流程![[Pasted image 20251127145922.png]]![[Pasted image 20251127162846.png]]
![[Pasted image 20251127162015.png]]


- 性能瓶颈（多线程Redis的优化点）：网络的影响
	- 命令请求处理器的读取请求数据
	- 命令回复处理器的写入返回数据
- 多线程Redis优化：![[Pasted image 20251127163357.png]]