### 在Ubuntu上安装`rabbitmq_delayed_message_exchange`插件

如果你的RabbitMQ是直接通过`apt`安装在Ubuntu系统上的，请遵循以下步骤：

1. **确认RabbitMQ版本**
    
    插件的版本必须与RabbitMQ的版本严格匹配。首先，检查你的RabbitMQ版本。
    
    ```
    # 查看RabbitMQ安装路径，从而确定版本号
    ls /usr/lib/rabbitmq/lib/rabbitmq_server-*/
    # 或者通过以下命令查看状态，其中包含版本信息
    sudo rabbitmqctl status
    ```
    
    记下版本号，例如 `3.8.2`。
    
2. **下载对应版本的插件**
    
    前往RabbitMQ社区插件的[GitHub发布页面](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases)，找到与你的RabbitMQ版本匹配的`.ez`插件文件。例如，对于3.8.2版本，你可能需要下载 `rabbitmq_delayed_message_exchange-3.8.2.ez`。
    
    你可以使用`wget`命令直接下载到服务器：
    
    ```
    # 替换URL中的版本号为实际需要的版本
    wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.8.0/rabbitmq_delayed_message_exchange-3.8.0.ez
    ```
    
3. **复制插件文件**
    
    将下载的插件文件复制到RabbitMQ的插件目录。插件目录路径通常为 `/usr/lib/rabbitmq/lib/rabbitmq_server-<版本号>/plugins/`。
    
    ```
    # 请将 <版本号> 和 <文件名> 替换为实际名称
    sudo cp rabbitmq_delayed_message_exchange-3.8.0.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.8.2/plugins/
    ```
    
4. **启用插件并重启**
    
    ```
    # 启用插件
    sudo rabbitmq-plugins enable rabbitmq_delayed_message_exchange
    # 重启RabbitMQ服务使插件生效
    sudo service rabbitmq-server restart
    ```

5. 验证安装
	无论使用哪种方法，安装成功后，你都可以通过RabbitMQ的管理后台进行验证。
	
	1. 打开浏览器，访问 `http://<你的服务器IP>:15672`，并使用你的凭据登录。
	    
	2. 进入 **Exchanges**​ 标签页。
	    
	3. 点击 **Add a new exchange**。
	    
	4. 在 **Type**​ 下拉菜单中，如果你能看到 **`x-delayed-message`**​ 这个选项，就说明插件已经成功安装并启用了。
### 使用插件：Spring Boot 集成使用

1.  添加依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

2. 配置类

```
@Configuration
public class RabbitDelayedMessageConfig {

    /**
     * 定义延迟消息交换器
     * 注意：类型必须是 x-delayed-message
     */
    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct"); // 底层交换器类型，可以是direct/topic/fanout
        
        return new CustomExchange(
            "order.delayed.exchange",  // 交换器名称
            "x-delayed-message",       // 交换器类型，固定值
            true,                      // 是否持久化
            false,                     // 是否自动删除
            args                       // 参数
        );
    }

    /**
     * 定义订单处理队列
     */
    @Bean
    public Queue orderProcessQueue() {
        return new Queue("order.process.queue", true); // 持久化队列
    }

    /**
     * 绑定队列到延迟交换器
     */
    @Bean
    public Binding orderBinding(Queue orderProcessQueue, CustomExchange delayedExchange) {
        return BindingBuilder.bind(orderProcessQueue)
                .to(delayedExchange)
                .with("order.process.key") // 路由键
                .noargs();
    }
}
```

3. 发送延迟消息

基础发送方式

```
@Service
@Slf4j
public class OrderService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 创建订单并发送延迟检查消息
     */
    public void createOrder(OrderDTO orderDTO) {
        // 1. 保存订单到数据库
        Order order = saveOrder(orderDTO);
        
        // 2. 发送30分钟延迟消息（检查订单支付状态）
        sendDelayedMessage(order.getId(), 30 * 60 * 1000); // 30分钟
        
        log.info("订单创建成功，ID: {}, 已发送延迟检查消息", order.getId());
    }

    /**
     * 发送延迟消息
     * @param orderId 订单ID
     * @param delayMillis 延迟时间（毫秒）
     */
    private void sendDelayedMessage(String orderId, long delayMillis) {
        // 创建消息
        OrderDelayMessage message = new OrderDelayMessage(orderId, System.currentTimeMillis());
        
        // 发送延迟消息
        rabbitTemplate.convertAndSend(
            "order.delayed.exchange",    // 延迟交换器
            "order.process.key",         // 路由键
            message,                     // 消息内容
            messagePostProcessor -> {
                // 设置延迟时间（毫秒）
                messagePostProcessor.getMessageProperties()
                    .setDelay(Long.valueOf(delayMillis).intValue());
                return messagePostProcessor;
            }
        );
    }
}

/**
 * 延迟消息体
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderDelayMessage implements Serializable {
    private String orderId;
    private long createTime;
}
```


4. 消费延迟消息

```
@Component
@Slf4j
public class OrderDelayConsumer {

    @Autowired
    private OrderService orderService;

    /**
     * 监听订单延迟处理队列
     */
    @RabbitListener(queues = "order.process.queue")
    public void handleOrderDelayMessage(OrderDelayMessage message) {
        log.info("收到订单延迟检查消息: {}", message);
        
        try {
            // 检查订单支付状态
            checkOrderPaymentStatus(message.getOrderId());
        } catch (Exception e) {
            log.error("处理订单延迟消息失败, orderId: {}", message.getOrderId(), e);
            // 可以根据业务需求进行重试
        }
    }

    /**
     * 检查订单支付状态
     */
    private void checkOrderPaymentStatus(String orderId) {
        Order order = orderService.getOrderById(orderId);
        
        if (order == null) {
            log.warn("订单不存在: {}", orderId);
            return;
        }

        switch (order.getStatus()) {
            case UNPAID:
                // 30分钟未支付，自动取消订单
                orderService.cancelOrder(orderId, "超时未支付");
                log.info("订单超时未支付已取消: {}", orderId);
                break;
                
            case PAID:
                log.info("订单已支付: {}", orderId);
                break;
                
            case CANCELLED:
                log.info("订单已取消: {}", orderId);
                break;
                
            default:
                log.warn("未知订单状态: {}, orderId: {}", order.getStatus(), orderId);
        }
    }
}
```

- 问题：
	1. **延迟时间精度**：插件提供的是尽力而为的延迟，不是实时系统
	    
	2. **内存使用**：大量延迟消息会占用较多内存
	    
	3. **重启影响**：RabbitMQ重启后，未投递的延迟消息会重新计算延迟
	    
	4. **集群环境**：在集群中工作正常，但建议了解其在高可用环境下的行为
	    
	5. **延迟限制**：最大延迟时间为2^32-1毫秒（约49天）
