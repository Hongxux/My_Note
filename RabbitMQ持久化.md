- 需求背景：在默认情况下，RabbitMQ会将接收到的信息保存在内存中以降低消息收发的延迟。
	- 一旦MQ宕机，内存中的消息会丢失
	- 内存空间有限，当消费者故障或处理过慢时，会导致消息积压，引发MQ阻塞
- 解决措施：RabbitMQ持久化
	- 先保存待内存中，每隔一段时间定期刷盘
- 使用方式：
	- 交换机持久化：确保Broker重启后，交换机的元数据（名称、类型、绑定关系）不丢失。
		- 声明交换机时设置 `durable=true`。
	- 队列持久化：确保Broker重启后，队列本身及其中的消息（若消息也是持久的）能够恢复。
		- 声明队列时设置 `durable=true`。
	- 消息持久化：将消息内容本身写入磁盘，而不仅存于内存
		- 发送消息时设置投递模式 `delivery_mode=2`（或使用 `PERSISTENT_TEXT_PLAIN`属性）
			```
			Message message = MessageBuilder.withBody("你的消息内容".getBytes())
					.setDeliveryMode(MessageDeliveryMode.PERSISTENT) // 设置为持久化
					.build();
			rabbitTemplate.send("business.exchange", "business.key", message);
			```
	- 即使消息被标记为持久化，但如果它被发送到一个**非持久化**的队列，那么在Broker重启后，该消息**依然会丢失**。因此，三者必须同时配置。
- 问题：
	- 持久化性能开销：持久化会带来性能开销（主要源于磁盘I/O），因此需要在**可靠性和吞吐量**之间做出权衡。对于非核心业务，可考虑使用非持久化消息以提升性能。
	- 消息堆积导致换页：
		- 诱因：息堆积速度超过消费速度，导致内存占用达到一定阈值时，RabbitMQ会触发一个名为 “Page-Out”的过程
		- 换页是阻塞操作，影响性能
			- 如果堆积常常发生，可以使用Lazy Queue，实现及时持久化