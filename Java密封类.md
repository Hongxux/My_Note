---
aliases:
  - sealed classes
---
通过密封类建模固定域模型（如JSON或XML），提高代码的可靠性和可维护性。意图是鼓励读者在需要严格类层次时使用密封类，替代传统的final类或抽象类，以利用编译时检查的优势。

---
密封：只允许被允许（permits）的类继承。

**密封类声明**​：使用`sealed`关键字声明密封类，并通过`permits`子句指定允许的子类列表。例如，`public abstract sealed class JSONValue permits JSONArray, JSONNumber, ...`。
- **子类约束**​：允许的子类必须直接扩展密封类，并且必须是`final`、`sealed`或`non-sealed`之一。这确保了继承层次的封闭性。
	- **`final`（不可进一步扩展）**
	- **`sealed`（进一步控制子类）**
	- **`non-sealed`（开放扩展）**：`non-sealed`是Java中第一个连字符关键字。
- **模块和包访问**​：允许的子类必须可访问，且如果公共子类，通常需在同一包或模块中。