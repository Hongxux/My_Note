- 需求背景：
	- 配置工作繁琐复杂：手动编写大量的 XML 配置文件或 Java 配置类来定义 Bean**繁琐**且容易出现**配置出错**
	- 版本管理困难：自行管理众多第三方库的版本及其兼容性，极易出现**版本冲突**问题，且排查难度大
	- 集成成本高：每次引入新的第三方库（如MyBatis、Redis等），都需要编写大量的"胶水代码"进行集成配置，增加了项目的复杂度和维护成本
	- 项目初始化：新项目的启动速度缓慢，需要从零开始配置各种组件，难以快速进行原型验证和迭代开发。
- 解决措施：SpringBoot 自动配置（**约定优于配置**）
	- 核心思想：通过**预先定义**一系列默认配置规则，**在满足特定条件时**自动装配组件，从而大幅减少手动配置工作
	- 实现机制：
		- **提供智能默认值**：为常见应用场景（如Web服务器、数据源）提供经过优化的默认配置参数，例如内嵌的Tomcat服务器和8080端口
		- **基于条件激活**：通过一系列 `@ConditionalOn...`注解（如 `@ConditionalOnClass`, `@ConditionalOnProperty`），根据类路径、环境、配置属性等条件动态判断是否应启用某项配置，确保按需加载
		- **确保用户配置优先（可覆盖性的关键）**：
			- **Bean覆盖**：如果开发者在自己的 `@Configuration`类中定义了某个 Bean，那么根据 `@ConditionalOnMissingBean`条件，自动配置中对应的默认 Bean 将不会被创建，从而实现用自定义 Bean 替换框架默认 Bean。
		    - **配置覆盖**：开发者在 `application.properties/yml`等配置文件中书写的任何属性值，其优先级都高于 `XxxProperties`类中的默认值。框架会自动读取并覆盖默认配置。
- 实现原理：
	1. **准备阶段：定义约定（Starter与配置类）**
		- 目标：确立“开箱即用”的基石，预先定义好技术栈的默认集成方案。
		- 关键机制：
			- **Starter启动器**：将某个功能（如`spring-boot-starter-web`）所需的所有依赖打包，解决版本管理问题。
		    - **自动配置类**：
			    - 关系：每个 Starter 通常包含一个或多个 **自动配置类**（`XxxAutoConfiguration`）
			    - 作用：这些类中通过 `@Bean`方法**定义了组件的默认创建逻辑**。
			    - 绑定配置属性类的方式：使用@EnableConfigurationProperties(CartoonProperties.class)指定属性配置类
		    - **配置属性类**：每个自动配置类关联一个 **配置属性类**（`XxxProperties`）
			    - 关联参数的方式：@ConfigurationProperties绑定配置文件前缀
			    - 作用：
				    - 参数绑定：【参数】配置文件（如 `application.yml`）中对应前缀的参数
				    - 默认值提供：
	2. **声明阶段：注册配置类**
		- 目标：让Spring Boot在启动时能够发现并定位到这些预先定义好的**自动配置类**。
		- 注册方式：在 Jar 包的特定文件中声明
		    - **Spring Boot 2.x**：在 `META-INF/spring.factories`文件中以键值对形式声明。
		    - **Spring Boot 3.x**：在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件中直接列出全限定类名（推荐方式）。
	3. **触发阶段：启用自动装配**
	    - 应用的主类使用 `@SpringBootApplication`注解，该注解包含了 **`@EnableAutoConfiguration`**，它是触发整个自动装配流程的入口开关。
	        ![[Pasted image 20251113184028.png]]![[Pasted image 20251113184050.png]]
	4. **加载阶段：收集候选配置**
		- 加载对象：所有 Jar 包中步骤2所提到的声明文件
	    - 加载方式：
		    1. **读取**并**收集**自动配置类：形成待处理的“候选自动配置类集合”。
			    1. `@EnableAutoConfiguration`注解通过`@Import`导入了`AutoConfigurationImportSelector`类
			    2. 这个类负责**扫描并读取**所有 Jar 包的对应位置的声明文件
				3. 文件中注册的所有自动配置类的全限定名被这个类**收集起来**，形成**“候选配置类集合”**
	        2. **筛选**：筛选出最终“**生效的**自动配置类集合”。
		        - 判断的条件：每个配置类上标注的 `@ConditionalOn...`​ 条件注解
			        - `@ConditionalOnClass`：类路径下存在指定的类时生效。
					- `@ConditionalOnMissingBean`：Spring 容器中不存在指定类型的 Bean 时生效。
						- 这保证了用户一旦自定义Bean，框架配置会自动后退
			        - `@ConditionalOnProperty`：配置文件中存在指定属性且为特定值时生效。
		        - 判断的依据：当前应用的**运行环境**（如类路径、已存在的 Bean、配置属性、Web 应用类型等）
	5. **生效阶段：创建Bean与注入配置**
	    - 该配置类中通过 `@Bean`注解的方法会向 Spring 容器注册组件实例。
	    - 配置类通过 `@EnableConfigurationProperties`导入对应的 `XxxProperties`类，并将配置文件中用户的配置值（或默认值）绑定到该 Bean 的属性上，完成配置注入。
	6. **覆盖阶段：用户自定义优先（重要原则）**：开发者可以轻松地定制和扩展任何默认配置，获得完全的控制权。
	    - **Bean覆盖**：如果开发者在自己的 `@Configuration`类中定义了某个 Bean，那么根据 `@ConditionalOnMissingBean`条件，自动配置中对应的默认 Bean 将不会被创建，从而实现用自定义 Bean 替换框架默认 Bean。
	    - **配置覆盖**：开发者在 `application.properties/yml`等配置文件中书写的任何属性值，其优先级都高于 `XxxProperties`类中的默认值。框架会自动读取并覆盖默认配置。