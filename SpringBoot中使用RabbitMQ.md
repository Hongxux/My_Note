1. 引入依赖
	```
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-amqp</artifactId>
	</dependency>
	```
2. 配置连接信息
	```
	spring:
	  rabbitmq:
	    host: 192.168.0.10  # RabbitMQ服务器地址
	    port: 5672       # AMQP协议端口，注意不是管理界面的15672
	    username: hongxu   # 默认用户名
	    password: h3800168   # 默认密码
	    virtual-host: /   # 虚拟主机，默认为/
	```
3. 声明队列，声明交换机，完成二者的绑定
	```
	@Configuration
	public class RabbitMQConfig {
	
		// 1. 配置一个队列，名为 "hello.queue"
		@Bean
		public Queue queue() {
			return new Queue("hello.queue", true); // true表示队列持久化
		}
	
		// 2. 配置一个直连交换机（DirectExchange），名为 "hello.exchange"
		@Bean
		public DirectExchange exchange() {
			return new DirectExchange("hello.exchange");
		}
	
		// 3. 将队列与交换机绑定，并设置路由键为 "hello.key"
		@Bean
		public Binding binding(Queue queue, DirectExchange exchange) {
			return BindingBuilder.bind(queue).to(exchange).with("hello.key");
		}
	}
	```
4. 创建消息生产者:使用 `RabbitTemplate`可以轻松发送消息。它是 Spring AMQP 提供的高层抽象工具类
	```
	
	@Autowired
	private RabbitTemplate rabbitTemplate;
	
	public void sendMessage(String message) {
		// 将消息发送到交换机 "hello.exchange"，并指定路由键 "hello.key"
		rabbitTemplate.convertAndSend("hello.exchange", "hello.key", message);
		System.out.println("消息已发送: " + message);
	}
	
	```
1.  创建消息消费者:使用 `@RabbitListener`注解来标记一个方法作为消息的消费者。当有消息到达指定队列时，该方法会被自动调用
	```
	
	@Component
	public class MessageReceiver {
	
	    // 监听名为 "hello.queue" 的队列
	    @RabbitListener(queues = "hello.queue")
	    public void receiveMessage(String message) {
	        System.out.println("接收到消息: " + message);
	        // 在这里添加你的业务处理逻辑
	    }
	}
	```