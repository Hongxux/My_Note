Stream（流）本身不存储数据，而是对数据源（如集合、数组）的元素进行一系列流水线式的处理。这个过程通常分为**中间操作**和**终端操作**。

- **中间操作**（如 `filter`, `map`）是“懒”的，它们只记录操作步骤，并不会立即执行。只有当终端操作被调用时，所有这些中间操作才会一起执行。
    
- **终端操作**（如 `collect`, `forEach`）会触发实际计算，遍历流中的元素并得到结果。执行后，该流就被消耗掉了，无法再使用。

## 中间操作


中间操作允许你对数据流进行筛选、转换、排序等处理。

### 1. **`filter`- 过滤**
    
根据条件过滤出需要的元素。

```
List<String> languages = Arrays.asList("Java", "Python", "C++", "JavaScript");
// 过滤出以 "J" 开头的语言
List<String> filteredLanguages = languages.stream()
										  .filter(lang -> lang.startsWith("J")) // 保留以"J"开头的元素
										  .collect(Collectors.toList());
// 结果: [Java, JavaScript]
```
    
### 2. **`map`- 映射/转换**
    
将流中的每个元素通过给定的函数进行转换，生成一个新的元素。

```
List<String> languages = Arrays.asList("Java", "Python", "C++", "JavaScript");
// 将每个语言字符串转换为它的长度
List<Integer> nameLengths = languages.stream()
									 .map(String::length) // 将每个字符串映射为其长度值
									 .collect(Collectors.toList());
// 结果: [4, 6, 3, 10]
```
    
### 3. **`flatMap`- 扁平化映射**
    
将多个流合并成一个流，特别适用于处理嵌套结构（如 `List<List<T>>`）。

```
List<List<String>> nestedList = Arrays.asList(
	Arrays.asList("Java", "Python"),
	Arrays.asList("C++", "JavaScript")
);
// 将嵌套的列表"拍平"为一个单一的流
List<String> flatList = nestedList.stream()
								.flatMap(Collection::stream) // 将每个内部List转换为流，然后合并
								.collect(Collectors.toList());
// 结果: [Java, Python, C++, JavaScript]
```
    
### 4. **`sorted`- 排序**
    
对流中的元素进行排序。

```
List<Integer> numbers = Arrays.asList(5, 8, 2, 6, 41, 11);
// 自然顺序排序
List<Integer> sortedNumbers = numbers.stream()
									.sorted()
									.collect(Collectors.toList());
// 结果: [2, 5, 6, 8, 11, 41]
```

### 5. **`distinct`- 去重**
    
去除流中重复的元素。

```
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);
List<Integer> distinctNumbers = numbers.stream()
									  .distinct()
									  .collect(Collectors.toList());
// 结果: [1, 2, 3, 4]
```
    
### 6. **`limit`与 `skip`- 截取与跳过**

- `limit(n)`: 截取流的前n个元素。
	
- `skip(n)`: 跳过流的前n个元素。
	

```
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> limited = numbers.stream().limit(3).collect(Collectors.toList()); // 结果: [1, 2, 3]
List<Integer> skipped = numbers.stream().skip(2).collect(Collectors.toList()); // 结果: [3, 4, 5]
```

##  终端操作

终端操作会触发流的遍历并产生最终结果。

### 1. **`forEach`- 遍历**
    
对流中的每个元素执行指定的操作。

```
List<String> languages = Arrays.asList("Java", "Python", "C++", "JavaScript");
languages.stream().forEach(System.out::println); // 逐个打印
```
    
### 2. **`collect`- 收集**
    
将流中的元素累积到不同的集合中，这是最常用的终端操作之一。

```
List<String> languages = Arrays.asList("Java", "Python", "C++", "JavaScript");
List<String> list = languages.stream().collect(Collectors.toList());
Set<String> set = languages.stream().collect(Collectors.toSet());
// 连接成字符串
String joined = languages.stream().collect(Collectors.joining(", ")); // 结果: "Java, Python, C++, JavaScript"
```

### 3. **`reduce`- 归约**
    
将流中的所有元素反复结合，得到一个单一的值，例如求和、求最大值。

```
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
// 求和
Integer sum = numbers.stream().reduce(0, Integer::sum); // 结果: 15
// 求最大值
Optional<Integer> max = numbers.stream().reduce(Integer::max);
```
    
### 4. **匹配检查**
    
- `anyMatch(Predicate)`: 判断流中是否有**至少一个**元素满足条件。
	
- `allMatch(Predicate)`: 判断流中是否**所有**元素都满足条件。
	
- `noneMatch(Predicate)`: 判断流中是否**没有**元素满足条件。
	

```
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
boolean hasEven = numbers.stream().anyMatch(n -> n % 2 == 0); // 是否存在偶数? true
boolean allPositive = numbers.stream().allMatch(n -> n > 0);   // 是否都为正数? true
boolean noneNegative = numbers.stream().noneMatch(n -> n < 0); // 是否没有负数? true
```
    
### 5. **查找**
    
- `findFirst()`: 返回流中的**第一个**元素。
	
- `findAny()`: 返回流中的**任意一个**元素（在并行流中效率更高）。
	

```
List<String> languages = Arrays.asList("Java", "Python", "C++", "JavaScript");
Optional<String> first = languages.stream().findFirst(); // 结果: "Java"
Optional<String> any = languages.parallelStream().findAny();
```
    
### 6. **`count`- 计数**

返回流中元素的个数。

```
List<String> languages = Arrays.asList("Java", "Python", "C++", "JavaScript");
long count = languages.stream().count(); // 结果: 4
```


### 💡 使用技巧与注意事项

- **链式调用**：Stream API 的魅力在于可以将多个操作流畅地链接起来，形成清晰的数据处理管道。
    
- **流不可复用**：一个流一旦被终端操作消耗，就不能再被使用。如果你需要再次处理同一组数据，需要重新创建流 。
    
- **并行流谨慎使用**：通过 `parallelStream()`可以轻松获得并行流，但并非所有情况都能提速，尤其在数据量小或操作本身不耗时时，可能因线程开销反而变慢。
    

希望这份详细的梳理能帮助你更好地掌握 Java Stream API！如果你对某个特定操作有更深入的疑问，我们可以继续探讨。