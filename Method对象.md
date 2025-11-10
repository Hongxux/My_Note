---
aliases:
  - Method
---

### ⚠️ Method 对象的注意事项

使用 `Method`对象时，有几个关键点需要特别注意：

1. **性能开销**
    
    反射调用方法的性能通常远低于直接调用。这主要是因为 JVM 无法对反射调用进行充分的优化，并且每次调用都可能涉及访问权限检查、方法解析等额外操作。对于需要频繁调用的热点代码，应谨慎使用。
    
2. **访问权限与 `setAccessible(boolean)`**
    
    默认情况下，反射无法调用类的 `private`等非公有方法。通过 `Method`对象的 `setAccessible(true)`方法，可以绕过 Java 语言的访问控制检查，从而调用私有方法。
    
    **注意**：这是一个危险操作，因为它破坏了对象的封装性，可能导致安全问题和不稳定的行为，应仅在绝对必要时（如框架开发）使用。
    
3. **异常处理**
    
    使用 `method.invoke()`调用方法时，目标方法自身抛出的任何异常都会被包装在 `InvocationTargetException`中。要获取原始的异常，需要调用 `InvocationTargetException.getTargetException()`方法。
    
4. **类型擦除与泛型**
    
    由于 Java 的类型擦除机制，反射在运行时无法获取方法的泛型参数类型信息。`Method`对象上的一些方法（如 `getReturnType()`）返回的是擦除后的类型（`Object`），而另一些方法（如 `getGenericReturnType()`）可以返回包含泛型信息的 `Type`对象。
    
5. **方法查找的精确性**
    
    通过 `Class.getMethod(String name, Class<?>... parameterTypes)`获取方法时，必须提供**精确的方法名**和**准确的参数类型列表**（考虑重载）。如果找不到匹配的方法，会抛出 `NoSuchMethodException`。`getDeclaredMethod`能获取本类中声明的所有方法（包括私有方法），但不包括继承的方法。
    

### 🛠️ Method 对象的常用方法

下表整理了 `Method`类中最常用的一些方法及其用途：

|方法名|功能描述|返回值/说明|
|---|---|---|
|**`invoke(Object obj, Object... args)`**|调用底层方法。`obj`是调用该方法的实例（静态方法则为 `null`），`args`是参数。|`Object`(方法的返回值，void方法返回null)|
|**`getName()`**|返回方法名称。|`String`|
|**`getParameterTypes()`**|获取方法的参数类型列表（按声明顺序）。|`Class<?>[]`|
|**`getReturnType()`**|获取方法的返回值类型。|`Class<?>`|
|**`getGenericReturnType()`**|获取方法的返回值类型，包含泛型信息。|`Type`|
|**`getExceptionTypes()`**|获取方法声明抛出的异常类型列表。|`Class<?>[]`|
|**`getModifiers()`**|返回方法的修饰符（如 public, static）。需用 `Modifier`类解码。|`int`|
|**`isVarArgs()`**|判断该方法是否接受可变数量的参数（形如 `String...`）。|`boolean`|
|**`getAnnotation(Class<T> annotationClass)`**|获取该方法上指定类型的注解（如果存在）。|`<T extends Annotation> T`|
|**`getDeclaringClass()`**|返回声明该方法的类对应的 `Class`对象。|`Class<?>`|
|**`setAccessible(boolean flag)`**|启用/禁用访问检查，设置为 `true`可调用私有方法。|`void`|

### 💎 总结与建议

`Method`对象是反射中用于动态调用行为的强大工具。使用时请牢记：

- **权衡性能**：避免在性能敏感的循环或核心路径中过度使用。
    
- **尊重封装**：谨慎使用 `setAccessible(true)`，并充分知晓其风险。
    
- **清晰调用**：确保在获取 `Method`对象时提供了准确的方法签名。
    

希望这些信息能帮助你更安全有效地使用 Java 反射。