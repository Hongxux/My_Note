- 含义：多个工作进程（消费者）来共同处理队列中的消息![[Pasted image 20251209222302.png]]
- 工作模式：
	- 使用公平分发：设置每个消费者在未确认之前**最多能预取的消息数量**为1
	- 消息手动确认
	- 队列和消息本身持久化
- 配置方式
```
  @Configuration
public class RabbitMQConfig {

    public static final String WORK_QUEUE = "work.queue";

    @Bean
    public Queue workQueue() {
        // 创建持久化队列
        return new Queue(WORK_QUEUE, true);
    }

    // 重要：配置监听器容器工厂，设置并发消费者数量
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        // 设置每个监听器并发启动的消费者数量，例如2个
        factory.setConcurrentConsumers(2);
        // 设置最大并发消费者数量
        factory.setMaxConcurrentConsumers(5);
        // 设置预取数量，实现公平分发。设为1表示一次只给消费者一条消息。
        factory.setPrefetchCount(1);
        // 设置为手动确认模式
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }
}
```
```
@Service
public class TaskWorker {

    // 使用 @RabbitListener 注解指定监听的队列，并使用配置的容器工厂
    @RabbitListener(queues = RabbitMQConfig.WORK_QUEUE, containerFactory = "rabbitListenerContainerFactory")
    public void receiveTask(String task, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
        System.out.println(" [x] Received task: '" + task + "' by worker: " + Thread.currentThread().getName());
        
        try {
            // 模拟耗时任务处理
            Thread.sleep(2000);
            System.out.println(" [x] Completed task: '" + task + "'");
            
            // 业务处理成功后，手动发送ACK确认
            channel.basicAck(deliveryTag, false);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            // 处理失败，拒绝消息并重新入队，让其他消费者处理
            channel.basicNack(deliveryTag, false, true);
        } catch (Exception e) {
            // 处理失败，根据情况决定是否重新入队
            channel.basicNack(deliveryTag, false, false); // 不重新入队，可记录日志或移入死信队列
        }
    }
}
```