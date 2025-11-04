#### 1. ​**背景和目的**​

- ​**历史上下文**​：在 Java 9 之前，编译跨版本代码需要使用复杂的选项，如 `-source`指定源代码版本、`-target`指定目标字节码版本，以及 `-bootclasspath`指向相应版本的引导类库以确保 API 兼容性。这容易出错且繁琐。
    
- ​**Java 9 改进**​：`--release`选项整合了这些功能，允许开发者通过一个标志指定目标 Java 版本（如 7、8、9），编译器会自动处理版本相关的配置，包括使用正确的 API 符号表（symbol files）来验证兼容性。
    
- ​**主要用途**​：`--release`常用于创建多版本 JAR 文件（Multi-Release JAR Files），其中不同版本的类文件可以共存，以支持不同 JDK 环境。例如，为 Java 8 和 Java 9 分别编译类文件，确保程序在两者上都能运行。
    

#### 2. ​**语法和用法**​

- ​**基本语法**​：`javac --release <version> <source files>`
    
    - `<version>`：指定目标 Java 版本，如 `7`、`8`、`9`。Java 9 及更高版本支持这些值。
        
    - 选项必须与双破折号（`--`）一起使用，这是 Java 9 引入的新命令行选项格式的一部分，旨在提高可读性和一致性。
        
    
- ​**常用示例**​：
    
    - 编译代码以兼容 Java 8：`javac --release 8 MyClass.java`
        
    - 编译代码以兼容 Java 9：`javac --release 9 MyClass.java`
        
    - 结合输出目录选项：`javac -d bin/8 --release 8 MyClass.java`（将编译后的类文件输出到 `bin/8`目录）。
        
    
- ​**与多版本 JAR 结合**​：当创建多版本 JAR 时，`--release`用于编译特定版本的类文件。例如：
    
    - 先编译 Java 8 版本：`javac -d bin/8 --release 8 Application.java`
        
    - 再编译 Java 9 版本：`javac -d bin/9 --release 9 Application.java`
        
    - 然后使用 `jar`命令打包：`jar cf MyProgram.jar -C bin/8 . --release 9 -C bin/9 Application.class`