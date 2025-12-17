- 声明式开发
- 问题：
	- 调度策略选择不当
		- fixedRate以上次任务的**开始时间**为起点计算下次执行
			- 坑点：如果任务执行时间超过其周期，会导致任务**重叠堆积**，迅速耗尽资源（前提是选择 多线程的调度器）
			- 例如：一个耗时 15 秒的任务配置为 `@Scheduled(fixedRate = 10000)`（每10秒一次），那么第10秒时第二个任务就会强行开始
		- fixedDelay以上次任务的**结束时间**为起点计算下次执行
			- 坑点：不能实现固定频率的任务
			- 好处：能保证任务不重叠
		- Cron 表达式：基于日历时间，适合每天定点这类场景。
			- 坑点：如果任务执行时间跨过了下一个触发点，则会**错过一次执行**
	- Spring 默认使用一个单线程的调度器来执行所有 `@Scheduled`任务
		- 问题：如果你有多个定时任务，它们会排队串行执行。一旦某个任务执行缓慢或阻塞，其他所有任务都会被延迟，就像堵车一样
		- 解决措施：配置一个自定义的线程池，有以下两种常见方式
			- 方式一：**实现 `SchedulingConfigurer`接口**（推荐，更灵活）
				```
				@Configuration
				@EnableScheduling
				public class SchedulerConfig implements SchedulingConfigurer {
					@Override
					public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
						taskRegistrar.setScheduler(Executors.newScheduledThreadPool(10)); // 设置线程数为10
					}
				}
				```
			- 方式二：直接注入 `TaskScheduler`Bean
				```
				@Bean
				public TaskScheduler taskScheduler() {
				    ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
				    scheduler.setPoolSize(5);
				    scheduler.setThreadNamePrefix("my-scheduler-");
				    scheduler.initialize();
				    return scheduler;
				}
				```
	- 分布式环境下的重复执行
		- **每个实例内的 `@Scheduled`注解都会独立生效，导致同一任务在多个节点上同时执行**，这可能引发数据重复处理、资源冲突等灾难性后果。
		- 解决措施：
			- **基于 Redis 的分布式锁**：这是最轻量、常见的解决方案。
				- 在任务开始执行前，尝试在 Redis 中设置一个具有过期时间的锁键。只有成功获取锁的实例才能执行业务逻辑，执行完毕后释放锁。
			- **数据库唯一约束**：
				- 对于按天/小时执行的任务，可以尝试向数据库插入一条包含任务名和执行日期的记录，利用唯一键约束来防止重复
			- **专业的分布式调度框架**：如果任务非常关键或需要复杂调度（如故障转移、动态调整），
				- 可以考虑使用 **Quartz**（配置为集群模式）或 **XXL-Job**​ 等专业框架
	- 如果 `@Scheduled`方法内部抛出了未捕获的异常，这个异常会直接终止当前任务的执行线程。更糟的是，**这个任务后续的调度也可能被静默停止，让你难以察觉**
		- 解决措施：务必在任务方法内部进行显式的异常捕获和处理，并记录日志，必要时发送告警
- 使用方式 :
	1.  开启定时任务：在 Spring Boot 主应用类或配置类上添加 `@EnableScheduling` 注解。
	2.  创建定时任务：在一个被 Spring 管理的 Bean 的方法上添加 `@Scheduled` 注解，并指定执行规则。
	    ```java
	    @Component
	    public class MyScheduledTask {
	        // 固定速率执行：从上一次任务开始时间算起，间隔固定时间执行
	        @Scheduled(fixedRate = 5000) // 单位：毫秒，此处意为每5秒执行一次
	        public void taskWithFixedRate() {
	            // 你的业务逻辑
	        }
	        // 固定延迟执行：上一次任务执行完成后，间隔固定时间再执行下一次
	        @Scheduled(fixedDelay = 3000) // 上次任务结束3秒后再执行
	        public void taskWithFixedDelay() {
	            // 你的业务逻辑
	        }
	        // 使用Cron表达式：提供最灵活的时间控制
	        @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
	        public void taskWithCron() {
	            // 你的业务逻辑
	        }
	    }
	    ```
	- [[Cron表达式]]