- 需求背景：
	- 有的时候由于网络波动，可能会出现客户端连接MQ失败的情况
- 配置方式
	```
	spring:
	  rabbitmq:
	    host: 127.0.0.1
	    # ... 其他基础配置
	    template:
	      retry:
	        enabled: true           # 是否启用重试机制
	        max-attempts: 3         # 最大重试次数
	        initial-interval: 1000ms # 第一次重试的间隔时间
	        multiplier: 2           # 重试间隔时间的倍数
	        max-interval: 10000ms   # 最大重试间隔时间
	```
- 问题：
	- SpringAMQP提供的是阻塞式的重试机制
		- 当前发起发送操作的线程会被挂起
		- 解决措施：使用异步线程来执行消息发送任务
	- 