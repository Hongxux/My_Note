JAR文件用于打包Java应用程序和库，包含类文件、资源文件（如图标、声音），并使用ZIP压缩格式以减少大小和提高效率。它提供了一种单一文件分发方式，避免了复杂的目录结构。

---
​
-  **创建JAR文件**​：使用`jar`工具（位于JDK的`bin`目录）创建JAR文件。基本命令语法为`jar options file1 file2 ...`，常见选项包括`c`（创建）、`v`（详细输出）、`f`（指定文件名）。例如，`jar cvf CalculatorClasses.jar *.class icon.gif`会创建一个包含所有类文件和图标的JAR文件。
	- **清单文件（Manifest）​**​：每个JAR文件都有一个清单文件`MANIFEST.MF`，位于`META-INF/`子目录中。清单文件描述存档的特性，如版本信息、主类（用于可执行JAR）或包密封。最小清单仅包含`Manifest-Version: 1.0`，但可以添加更多节（sections）来描述特定文件或包。节之间用空行分隔。
		 ![[Pasted image 20251018100916.png]]
		- ​**编辑清单**​：可以通过命令行选项`cfm`（创建带清单的JAR）或`ufm`（更新清单）来管理清单。例如，`jar cfm MyArchive.jar manifest.mf com/mycompany/mypkg/*.class`使用指定清单文件创建JAR。
	- **jar命令行**
		- ![[Pasted image 20251018101145.png]]
		
		
	- **[[--release]]**：编译跨版本代码
- **创建可执行JAR文件**、
	-  ​**两种指定主类的方法**​：
    
	    - 使用jar命令的`-e`选项（如`jar cvfe MyProgram.jar com.mycompany.mypkg.MainAppClass files`），直接指定入口点。
	        
	    - 在manifest文件中添加`Main-Class`头（如`Main-Class: com.mycompany.mypkg.MainAppClass`），注意不要添加`.class`扩展名。
	        
		- ​**manifest文件的要求**​：manifest文件的最后一行必须以换行符结束，否则可能导致读取失败。这是一个常见错误点。****
	- ​**操作系统集成**​：
    
	    - Windows：通过文件关联，双击JAR文件使用`javaw -jar`命令启动（无命令行窗口）。
	        
	    - Mac OS X：系统自动识别.jar扩展名并执行Java程序。
        
    
	- ​**第三方工具**​：对于更本地化的体验，可以使用工具如Launch4J或IzPack将JAR文件转换为Windows可执行文件（.exe），这些工具处理JVM定位和用户提示。