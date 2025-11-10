- `singleton`：默认。每个 IoC 容器中一个 Bean 定义只对应一个实例。
	
- `prototype`：每次获取都会创建一个新的 Bean 实例。
	
- `request`：一次 HTTP 请求一个实例（仅适用于 Web 环境）。
	
- `session`：一个 HTTP Session 一个实例（仅适用于 Web 环境）。
	
- `application`：一个 ServletContext 生命周期内一个实例。
	