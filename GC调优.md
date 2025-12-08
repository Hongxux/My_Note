1. 确定需求场景，选择合适的垃圾回收器
	- [[低延迟]]：快，使用CMS，G1，ZGG垃圾回收器
	- [[高吞吐]]：多，使用ParallelGC垃圾回收器

2. 收集数据：
	- 启用详细的GC日志：记录每一次GC的类型、时间、内存回收前后变化、暂停时间等关键信息
		```
		 # JDK 8及之前
		-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/to/gc.log
		
		# JDK 9及之后（推荐使用统一日志框架）
		-Xlog:gc*=info:file=/path/to/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
		```
	- 配置OOM时自动堆转储：
		```
		 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dump.hprof
		```
	- 使用实时监控工具
		- jconsole/ jvisualvm
		- APM工具：如Arthas、Prometheus + Grafana等，提供更全面的应用性能监控视图
		  
3. 分析数据：与正常的标准进行对比
	- 分析工具：GCeasy、GCViewer
	- 分析指标：
		1. GC停顿时间
		    - Young GC平均/最大暂停时间：如果过长（如超过50ms），可能原因是年轻代过大，单次回收对象多。
		    - Full GC暂停时间与频率：Full GC是“性能杀手”，目标是尽可能避免或减少其发生。频繁的Full GC通常意味着老年代对象增长过快或内存泄漏。
		2. **吞吐量**
		    - 计算公式：`吞吐量 = 应用运行时间 / (应用运行时间 + GC耗时) * 100%`。
		    - 目标：对于后端服务，通常要求达到 95%**​ 以上。过低意味着GC占用了过多CPU资源。
		3. 内存分配与回收效果
		    - 年轻代增长速率：通过GC日志估算Eden区每秒占用的速度，推算出Young GC的频率。频率过高说明对象创建太快或Eden区太小。
		    - 对象晋升老年代速率：观察每次Young GC后有多少对象进入老年代。**过早晋升**（即年轻代对象存活年龄很小就进入老年代）是导致Full GC的常见原因，通常因为Survivor区不足或`-XX:MaxTenuringThreshold`设置不合理。
4. 常见问题与针对性参数调整：每次只调整1-2个参数，然后重新测试观察效果
	- 核心思想：最快的GC是不发生GC
	- 优化[[Full GC优化策略]]

5. [[GC调优案例]]