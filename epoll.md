- 需求背景：
	- select模式存在的三个问题:
		- 能监听的FD最大不超过1024
		- 每次select都需要把所有要监听的FD都拷贝到内核空间
		- 每次都要遍历所有FD来判断就绪状态
	- poll模式的问题:
		- poll利用链表解决了select中监听FD上限的问题，但依然要遍历所有FD，如果监听较务，性能会下降
- 解决方案：epoll
	
- 数据结构：![[Pasted image 20251127140449.png]]
	- 红黑树：
		- 增删改查是O（logn）：基于epoll实例中的红黑树保存要监听的FD，理论上无上限，而且增删改查效率都非常高，性能不会随监听的FD数量增多而下降
		- 拷贝性能优化：每个FD只需要执行一次epoll_ctl添加到红黑树，以后每次epol_wait无需传递任何参数，无需重复拷贝FD到内核空间

		
- epoll的三大方法：
	- 创建epoll：返回epoll的唯一标识--epfd![[Pasted image 20251127140529.png]]
	- 监听fd：完成的功能如下![[Pasted image 20251127140648.png]]
		- 将一个FD添加到epoll的红黑树中
		- 并设置ep_poll_callback
			- callback触发时，就把对应的FD加入到rdlist这个就绪列表中
			- callback事件通知有两种模式：[[事件通知模式]]
	- 等待fd就绪：检查rdlist列表，如何有fd就绪则**将其加入到event数组**，并且返回有几个就绪fd![[Pasted image 20251127141055.png]]![[Pasted image 20251127141508.png]]
		- 可以指定是否阻塞，以及阻塞多久
		- 可以得知就绪的fd是哪几个，不需要依次遍历（events）
		- 无序拷贝所有的监听fd回用户空间，只需要拷贝已经就绪的fd（events）
		- 使用红黑树而非链表，可以监听很多个fd且性能不会随着fd增加而有太大影响
-  基于epoll的web服务![[Pasted image 20251127145922.png]]
	- 有两类fd：
		- serverSocketFD（ssfd）：其可读代表有新的客户端想要连接，接受后返回客户端socket的FD
		- socketFD：其可读代表有请求参数
		