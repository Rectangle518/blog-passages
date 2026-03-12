这里是对Python的一些零散的补充学习笔记，主要是一些需要注意的点。

- [函数的参数传递](#函数的参数传递)
  - [核心机制：传对象引用](#核心机制传对象引用)
  - [情况一：不可变对象（表现像“传值”）](#情况一不可变对象表现像传值)
  - [情况二：可变对象（表现像“传引用”）](#情况二可变对象表现像传引用)
- [Python的推导式](#python的推导式)
  - [1. 列表推导式 (List Comprehension)](#1-列表推导式-list-comprehension)
  - [2. 字典推导式 (Dictionary Comprehension)](#2-字典推导式-dictionary-comprehension)
  - [3. 集合推导式 (Set Comprehension)](#3-集合推导式-set-comprehension)
  - [4. 生成器表达式 (Generator Expression)](#4-生成器表达式-generator-expression)
  - [总结与最佳实践](#总结与最佳实践)
- [NO ENDING...](#no-ending)

## 函数的参数传递

在 Python 中，关于参数传递是“传值”还是“传引用”的问题，最准确的描述既不是纯粹的“传值”（Pass by Value），也不是纯粹的“传引用”（Pass by Reference），而是 **“传对象引用”**（Pass by Object Reference）或者更通俗地称为 **“传赋值”**（Pass by Assignment）。

这意味着：当你调用函数时，你传递的是**对象的引用（内存地址）的副本**。

理解这一机制的关键在于区分 **可变对象（Mutable）** 和 **不可变对象（Immutable）**。

### 核心机制：传对象引用

1.  **变量即标签**：在 Python 中，变量名只是指向内存中对象的“标签”或“引用”。
2.  **传递的是引用的副本**：当把变量传给函数时，函数内部的参数名成为了该对象的另一个“标签”。此时，函数内的参数和函数外的变量都指向**同一个对象**。

### 情况一：不可变对象（表现像“传值”）

如果传递的是**不可变对象**（如：整数 `int`、浮点数 `float`、字符串 `str`、元组 `tuple`），在函数内部试图修改这个参数时，Python 会创建一个新的对象，并将函数内的局部变量指向这个新对象。**原变量指向的对象不会改变。**

这给人的感觉像是“传值”，因为外部变量的值没变。

**代码示例：**

```python
def modify_immutable(x):
    print(f"函数内修改前 id: {id(x)}")
    x = x + 10  # 对于整数，这会创建一个新的整数对象
    print(f"函数内修改后 id: {id(x)}")
    print(f"函数内值: {x}")

a = 100
print(f"函数外初始 id: {id(a)}")
modify_immutable(a)
print(f"函数外最终值: {a}") 
# 输出: 100 (外部变量 a 没有改变)
```

*   **解释**：`x = x + 10` 并没有修改原来的整数 `100`（因为整数不可改），而是计算出了 `110`，生成了一个新对象，然后把函数内的标签 `x` 贴到了 `110` 上。外面的标签 `a` 依然贴在 `100` 上。

### 情况二：可变对象（表现像“传引用”）

如果传递的是**可变对象**（如：列表 `list`、字典 `dict`、集合 `set`、自定义类的实例），在函数内部通过该引用**直接修改对象的内容**（而不是重新赋值给变量），那么**外部的变量也会看到这些变化**。

这给人的感觉像是“传引用”，因为外部对象被改变了。

**代码示例：**

```python
def modify_mutable(lst):
    print(f"函数内修改前 id: {id(lst)}")
    lst.append(4)  # 直接在原对象上添加元素，不创建新对象
    print(f"函数内修改后 id: {id(lst)}")
    print(f"函数内值: {lst}")

my_list = [1, 2, 3]
print(f"函数外初始 id: {id(my_list)}")
modify_mutable(my_list)
print(f"函数外最终值: {my_list}") 
# 输出: [1, 2, 3, 4] (外部变量 my_list 指向的对象内容被改变了)
```

*   **解释**：`lst.append(4)` 是直接操作内存中的那个列表对象。因为函数内的 `lst` 和函数外的 `my_list` 指向同一个内存地址，所以修改是同步的。

**注意陷阱：如果在函数内对可变变量进行“重新赋值”**

如果你在函数内对可变变量使用了 `=` 赋值操作（例如 `lst = [5, 6]`），这就变成了创建新对象并改变局部标签的指向，**不会**影响外部变量。

```python
def reassign_mutable(lst):
    lst = [5, 6]  # 创建了新列表，将局部变量 lst 指向新列表
    print(f"函数内值: {lst}")

my_list = [1, 2, 3]
reassign_mutable(my_list)
print(f"函数外最终值: {my_list}") 
# 输出: [1, 2, 3] (外部变量未受影响)
```


## Python的推导式

Python 的**推导式**（Comprehensions）是一种简洁、高效且符合 Python 风格（Pythonic）的语法结构，用于从现有的可迭代对象（如列表、元组、字符串、集合、字典等）构建新的数据结构。

推导式的核心优势在于：**代码更短、可读性更强（熟练后）、执行速度通常比普通的 `for` 循环更快**。

Python 主要支持四种推导式：**列表推导式**、**字典推导式**、**集合推导式**和**生成器表达式**。

### 1. 列表推导式 (List Comprehension)
这是最常用的一种。它用于创建一个新的列表。

**基本语法：**
```python
[expression for item in iterable if condition]
```
*   `expression`: 对每个元素进行的操作或计算。
*   `item`: 迭代变量。
*   `iterable`: 原始的可迭代对象。
*   `if condition`: (可选) 过滤条件，只保留满足条件的元素。

**示例 1：基础用法（平方数）**
将列表中每个数字平方。
```python
numbers = [1, 2, 3, 4, 5]

# 传统写法
squares_traditional = []
for n in numbers:
    squares_traditional.append(n ** 2)

# 推导式写法
squares = [n ** 2 for n in numbers]

print(squares)  # 输出: [1, 4, 9, 16, 25]
```

**示例 2：带条件过滤（偶数平方）**
只计算偶数的平方。
```python
numbers = [1, 2, 3, 4, 5, 6]

# 推导式写法：先判断 n % 2 == 0，满足则计算 n**2
even_squares = [n ** 2 for n in numbers if n % 2 == 0]

print(even_squares)  # 输出: [4, 16, 36]
```

**示例 3：嵌套循环（矩阵展平）**
将一个二维列表（矩阵）展平为一维列表。
```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

# 推导式写法：外层循环在前，内层循环在后
flattened = [num for row in matrix for num in row]

print(flattened)  # 输出: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### 2. 字典推导式 (Dictionary Comprehension)
用于创建一个新的字典。语法与列表推导式类似，但使用花括号 `{}`，并且需要指定键和值。

**基本语法：**
```python
{key_expression: value_expression for item in iterable if condition}
```

**示例1：转换键值对**
将一个列表转换为字典，键为元素，值为元素的平方。
```python
numbers = [1, 2, 3, 4]

# 推导式写法
squares_dict = {x: x ** 2 for x in numbers}

print(squares_dict)  # 输出: {1: 1, 2: 4, 3: 9, 4: 16}
```

**示例2：交换键值对**
反转现有字典的键和值。
```python
original = {'a': 1, 'b': 2, 'c': 3}

# 推导式写法
swapped = {value: key for key, value in original.items()}

print(swapped)  # 输出: {1: 'a', 2: 'b', 3: 'c'}
```

### 3. 集合推导式 (Set Comprehension)
用于创建一个新的集合（自动去重）。语法使用花括号 `{}`，但只包含单个表达式（没有键值对）。

**基本语法：**
```python
{expression for item in iterable if condition}
```

**示例：去重并处理**
计算字符串中每个字符的 ASCII 码，并自动去重。
```python
text = "hello"

# 推导式写法：集合会自动去除重复的 'l'
ascii_codes = {ord(char) for char in text}

print(ascii_codes)  
# 输出: {104, 101, 108, 111} (顺序可能不同，因为集合无序，且 'l' 只出现一次)
```

### 4. 生成器表达式 (Generator Expression)
虽然严格来说不叫“推导式”，但语法极其相似。它使用圆括号 `()`。
**关键区别**：列表/字典/集合推导式会**立即**生成整个数据结构并占用内存；而生成器表达式返回一个**迭代器**，它是**惰性求值**的（用到时才计算），非常节省内存。

**基本语法：**
```python
(expression for item in iterable if condition)
```

**示例：处理大量数据**
假设我们要处理一百万个数字，只需要它们的平方和，不需要保存所有平方数。
```python
# 列表推导式：立即创建包含100万个整数的列表，占用大量内存
# big_list = [x ** 2 for x in range(1000000)] 

# 生成器表达式：只创建一个对象，内存占用极小
gen = (x ** 2 for x in range(1000000))

# 消耗生成器
total_sum = sum(gen)
print(total_sum)
```

### 总结与最佳实践

| 类型 | 符号 | 结果类型 | 特点 |
| :--- | :---: | :--- | :--- |
| **列表推导式** | `[]` | `list` | 最常用，立即生成所有数据 |
| **字典推导式** | `{k:v}` | `dict` | 快速构建映射关系 |
| **集合推导式** | `{}` | `set` | 快速去重 |
| **生成器表达式** | `()` | `generator` | **省内存**，适合大数据流处理 |

**何时使用？**
1.  **逻辑简单时**：如果逻辑可以用一行清晰表达，优先使用推导式。
2.  **需要性能时**：推导式通常比 `for` 循环 append 快。
3.  **数据量大时**：如果不需要一次性拿到所有结果，请使用**生成器表达式** `()` 而不是列表推导式 `[]`。

**何时避免？**
*   **逻辑复杂时**：如果包含多层嵌套循环、复杂的 `if-else` 逻辑，或者一行代码超过 80-100 个字符，请改用普通的 `for` 循环。**可读性永远优于简洁性。**

**复杂逻辑的反例（不推荐）：**
```python
# 难以阅读，建议拆分为普通循环
result = [x * y for x in range(10) for y in range(10) if x > 5 if y < 5 if x % 2 == 0]
```

## NO ENDING...