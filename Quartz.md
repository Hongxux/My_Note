

1. 核心组件：

|组件|作用与说明|
|---|---|
|**Job**​|你需要实现的业务逻辑接口，只需实现 `execute`方法。Quartz每次执行Job时都会创建一个新的实例。|
|**JobDetail**​|用来定义Job的静态属性（身份信息等），通过 `JobBuilder`创建。|
|**Trigger**​|定义Job的执行规则（时间、频率等）。主要有 `SimpleTrigger`（固定间隔）和 `CronTrigger`（类Cron表达式）两种。|
|**Scheduler**​|任务调度的总控制器，负责将 `JobDetail`和 `Trigger`关联起来，并调度和执行任务。|
2. 使用步骤：
	1. **添加依赖**：在 `pom.xml`中添加Quartz依赖。
		```xml
		 <dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-quartz</artifactId>
		</dependency>
		```
	2. **实现Job接口**：创建一个类（如`MyJob`），实现`org.quartz.Job`接口，并将你的业务逻辑写在`execute`方法中。
	    
	3. **定义任务与触发器**：通过`JobBuilder`和`TriggerBuilder`分别创建`JobDetail`和`Trigger`。
		```java
		@Configuration
		public class QuartzConfig {
		
			@Bean
			public JobDetail myJobDetail() {
				return JobBuilder.newJob(MyQuartzJob.class)
						.withIdentity("myJob", "group1") // 指定任务标识（名称和组）
						.storeDurably() // 即使没有触发器关联，也保留该JobDetail
						.build();
			}
		
			@Bean
			public Trigger myTrigger() {
				return TriggerBuilder.newTrigger()
						.forJob(myJobDetail()) // 关联上述 JobDetail
						.withIdentity("myTrigger", "group1")
						.withSchedule(CronScheduleBuilder.cronSchedule("0/10 * * * * ?")) // 每10秒执行一次
						.build();
			}
		}
		```
	4. **调度任务**：获取`Scheduler`实例，将`JobDetail`和`Trigger`注册到调度器并启动它。
	    

2.  主要“坑点”与解决方案
	- Quartz默认使用自己的机制实例化Job对象，不经过Spring容器，无法在里面注入自己的Bean
		- 解决措施：配置一个特殊的`JobFactory`，让Quartz使用Spring来创建Job实例。你需要定义一个继承`SpringBeanJobFactory`的类，并在配置中设置给Scheduler
	- 默认情况下，每个Quartz调度器实例是独立的，不知道其他实例的存在，会出现集群环境任务重复执行
		- 解决措施：在`quartz.properties`中开启集群模式,Quartz会通过数据库表锁来协调，确保同一时间只有一个实例执行任务
			```
			org.quartz.jobStore.isClustered = true
			org.quartz.scheduler.instanceId = AUTO
			```
	- 如果你的任务执行时间可能超过触发间隔，就需要考虑并发控制，避免操作同一个数据
		- 解决措施：给Job类加上`@DisallowConcurrentExecution`注解可以防止同一个JobDetail并发执行
		- 带来的问题：如果任务因某种原因（如网络请求没有设置超时）被无限期阻塞，后续的触发都会被“卡住”，这个任务就像“消失”了一样
		- 解决措施 ：务必在任务代码中为所有外部调用（如HTTP请求、数据库查询）设置合理的超时时间
	- Quartz默认会尝试执行所有错过的（Misfire）任务，宕机后恢复会导致瞬间高压
		- 解决措施：根据业务场景为Trigger配置合适的Misfire策略
			- 对于周期性任务，通常更合理的做法是“错过就错过”，等待下一次正常触发
				```
				Trigger trigger = TriggerBuilder.newTrigger()
					.withIdentity("safeSimpleTrigger", "group1")
					.withSchedule(SimpleScheduleBuilder.simpleSchedule()
					  .withIntervalInMinutes(10)
					  .repeatForever()
					  .withMisfireHandlingInstructionNextWithExistingCount()) // 关键设置
					.build();
				```
			- 对于CronTrigger，可以使用`withMisfireHandlingInstructionDoNothing()`策略
			```
			Trigger trigger = TriggerBuilder.newTrigger()
			    .withIdentity("safeTrigger", "group1")
			    .withSchedule(CronScheduleBuilder.cronSchedule("0 0/5 * * * ?")
			      .withMisfireHandlingInstructionDoNothing()) // 关键设置：错过触发后什么也不做
			    .build();
			```
	- Quartz有一个内部线程池来处理任务触发。如果线程池大小设置过小，而恰好有多个耗时较长的任务（比如调用外部接口，但外部接口响应缓慢或超时）占满了所有线程，那么调度器就没有空闲线程去触发其他任务了，导致所有定时任务停止
		- 解决措施：
			- 合理设置线程池大小
			- 将业务逻辑异步化：Quartz的Job应该只负责触发，尽快释放线程。具体的耗时操作可以提交给业务层的线程池或消息队列去异步执行