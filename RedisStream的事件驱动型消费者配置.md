
### 1. 构建信息消费服务
这是实现多实例负载均衡和可靠消费（不丢消息） 的核心。
[[Redis Stream的构建信息消费服务]]
配置的关键是决定是基于 `ObjectRecord`的 Redis Stream 消息队列，还是基于基于 `MapRecord`的 Redis Stream 消息队列
### 2.配置消息监听器（StreamListener）
#### 工作流程
1. `StreamMessageListenerContainer`在调用 `start()`方法后，会在**后台自动启动一个持续的监听循环**。
2. 当Redis Stream中有新消息到达，或者有待处理的Pending消息时，容器会自动将消息**推送**给你实现的 `StreamListener`的 `onMessage`方法。
3. **获取信息的不同模式**：通过配置不同的 `ReadOffset`，你可以控制消费者从哪里获取信息：
	- `ReadOffset.latest()`：只获取监听启动后**新到达**的消息。
	    
	- `ReadOffset.from(">")`：从**上一次ACK的位置之后**开始消费（最常用）。
	    
	- `ReadOffset.from("0")`：**重新处理**所有未被ACK的Pending消息。
	    
	- `ReadOffset.from("某个具体ID")`：从指定的消息ID开始消费。
##### 如何[[动态绑定消息监听器]]
### 3.自定义错误处理器 (ErrorHandler)
这个类决定了在发生错误时是否停止容器
[[Redis Stream的自定义错误处理器]]


