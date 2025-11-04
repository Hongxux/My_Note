**多态性**​：对象变量是多态的，即**一个超类变量（如Employee e）可以引用超类对象或任何子类对象（如Manager）**。这使得代码可以通用地处理对象，而不依赖于具体类型。
- **数组赋值和类型安全**​：子类数组可以赋值给超类数组变量（如Employee[] staff = managers;），但尝试在数组中存储不兼容类型（如将Employee对象存入Manager数组）会导致ArrayStoreException。数组在创建时记住元素类型，并监控存储操作以确保类型安全。
**方法调用**​：通过超类变量调用方法时，Java使用**动态绑定（dynamic binding）** 根据实际对象类型选择正确的方法实现。例如，e.getSalary()会调用Employee或Manager的getSalary方法，取决于e引用的实际对象。[[java 方法调用机制]]

