### 标准toString()
- ​**标准格式**​：大多数`toString`方法遵循类名后跟字段值在方括号内的格式，例如`ClassName[field1=value1,field2=value2]`。这提高了可读性和一致性。
    
- ​**动态获取类名**​：避免硬编码类名，使用`getClass().getName()`来支持子类：
    
    ```
    public String toString() {
        return getClass().getName() + "[name=" + name + ",salary=" + salary + ",hireDay=" + hireDay + "]";
    }
    ```
    
    这样，子类继承时也能正确显示类名。
    
- ​**字段格式化**​：确保字段值被正确转换为字符串，使用字符串拼接或格式化工具。对于基本类型，直接转换；对于对象，依赖其`toString`方法。
    

####  ​**在继承中的处理**​

当存在继承关系时，子类应扩展超类的`toString`方法：

- ​**调用super.toString()​**​：子类先调用超类的`toString`获取基础字符串，然后添加自己的字段。例如，在`Manager`类中：
    
    ```
    public String toString() {
        return super.toString() + "[bonus=" + bonus + "]";
    }
    ```
    
    这样，`Manager`对象的输出为`Manager[name=...,salary=...,hireDay=...][bonus=...]`，保持了层次结构。
    
- ​**兼容性**​：使用`getClass().getName()`确保类名动态变化，避免子类显示错误类名。