# 5. 字典与集合

## 本章目标

掌握字典（键值对）的增删改查、遍历和嵌套，理解集合的去重与集合运算。

## 5.1 字典：用名字查数据

字典用 `{}` 定义，存**键值对**（key: value），像查字典一样按"键"取值：

```python
student = {
    "name": "小明",
    "age": 18,
    "score": 92.5
}

# 按键取值
print(student["name"])      # 小明
print(student["age"])       # 18

# get() 取值（键不存在返回默认值，不报错）
print(student.get("gender"))          # None
print(student.get("gender", "未知"))  # 未知
```

**字典特点：**
- 键必须是**不可变类型**（字符串、数字、元组）
- 键**唯一**（重复赋值会覆盖）
- Python 3.7+ 字典保持插入顺序

## 5.2 增删改查

```python
student = {"name": "小明", "age": 18}

# 增：键不存在就是新增
student["score"] = 92
print(student)   # {'name': '小明', 'age': 18, 'score': 92}

# 改：键存在就是修改
student["age"] = 19
print(student)   # {'name': '小明', 'age': 19, 'score': 92}

# 删
del student["score"]
print(student)   # {'name': '小明', 'age': 19}

# 查：键是否存在
print("name" in student)    # True
print("score" in student)   # False
```

## 5.3 遍历字典

```python
student = {"name": "小明", "age": 19, "score": 92}

# 遍历键
for key in student:
    print(key)          # name / age / score

# 遍历值
for value in student.values():
    print(value)        # 小明 / 19 / 92

# 遍历键值对（推荐）
for key, value in student.items():
    print(f"{key} = {value}")
```

**运行结果：**

```
name = 小明
age = 19
score = 92
```

## 5.4 字典常用方法

```python
d = {"a": 1, "b": 2}

print(list(d.keys()))      # ['a', 'b']     所有键
print(list(d.values()))    # [1, 2]         所有值
print(list(d.items()))     # [('a', 1), ('b', 2)]  所有键值对（元组）

# update 合并字典
d.update({"c": 3, "b": 99})
print(d)                   # {'a': 1, 'b': 99, 'c': 3}

# pop 删除并返回
value = d.pop("a")
print(value, d)            # 1 {'b': 99, 'c': 3}

# 清空
d.clear()
print(d)                   # {}
```

## 5.5 字典推导式

和列表推导式类似，用 `{}` + 键值对：

```python
# 生成 1-5 的平方字典
squares = {i: i ** 2 for i in range(1, 6)}
print(squares)     # {1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# 翻转键值
d = {"a": 1, "b": 2}
flipped = {v: k for k, v in d.items()}
print(flipped)     # {1: 'a', 2: 'b'}

# 过滤：只留偶数键
nums = {1: "one", 2: "two", 3: "three", 4: "four"}
even = {k: v for k, v in nums.items() if k % 2 == 0}
print(even)        # {2: 'two', 4: 'four'}
```

## 5.6 嵌套结构（重点）

列表和字典可以互相嵌套，组成复杂数据结构：

```python
# 字典里放列表
class_room = {
    "math": [85, 92, 78],
    "english": [90, 88, 95]
}
print(class_room["math"][1])      # 92  数学第二个成绩

# 列表里放字典（最常用：数据记录）
students = [
    {"name": "小明", "age": 18, "score": 92},
    {"name": "小红", "age": 17, "score": 88},
    {"name": "小刚", "age": 19, "score": 95}
]

# 遍历打印
for s in students:
    print(f"{s['name']} 的成绩是 {s['score']}")

# 找出最高分
best = max(students, key=lambda s: s["score"])
print(f"最高分：{best['name']} ({best['score']}分)")
```

**运行结果：**

```
小明 的成绩是 92
小红 的成绩是 88
小刚 的成绩是 95
最高分：小刚 (95分)
```

## 5.7 集合：去重神器

集合用 `{}` 定义（空集合必须 `set()`），**元素唯一、无序**：

```python
# 创建
s = {1, 2, 3, 3, 2, 1}
print(s)         # {1, 2, 3}  自动去重
empty_set = set()    # 空集合（不能用 {}，那是空字典）

# 增删
s.add(4)
s.discard(1)     # 删除（不存在也不报错）
print(s)         # {2, 3, 4}
```

**去重最常见用法：**

```python
# 从列表中提取不重复元素
nums = [1, 2, 2, 3, 3, 3, 4]
unique = list(set(nums))
print(unique)    # [1, 2, 3, 4]  （顺序不保证）
```

## 5.8 集合运算

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

print(a & b)       # {3, 4}          交集
print(a | b)       # {1, 2, 3, 4, 5, 6}  并集
print(a - b)       # {1, 2}          差集（在a不在b）
print(a ^ b)       # {1, 2, 5, 6}    对称差集
```

**实际应用：找两个列表的共同元素**

```python
x = ["apple", "banana", "cherry"]
y = ["banana", "cherry", "durian"]
common = set(x) & set(y)
print(common)      # {'cherry', 'banana'}
```

## 5.9 综合示例：单词统计

```python
# 统计一段文本中每个单词出现次数
text = "apple banana apple cherry banana apple"
words = text.split()

# 用字典计数
counter = {}
for word in words:
    counter[word] = counter.get(word, 0) + 1

print(counter)   # {'apple': 3, 'banana': 2, 'cherry': 1}

# 找出出现最多的词
top_word = max(counter, key=counter.get)
print(f"出现最多的词：{top_word}，共 {counter[top_word]} 次")
```

**讲解：** `counter.get(word, 0) + 1` —— 取当前计数（没有则 0）加一，这是字典计数的经典写法。

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `KeyError: 'xxx'` | 键不存在直接 `d[key]` 取值 | 用 `d.get(key, 默认值)` |
| 用 `{}` 创建空集合 | 得到的是空字典 | 用 `set()` |
| 用列表做字典键报错 | 键必须不可变 | 改用元组或字符串 |

## 延伸阅读

- 列表与元组 → [[4-Py-列表与元组]]
- for 循环遍历 → [[6-Py-条件与循环]]
- 嵌套数据在实际项目中的用法 → [[13-Py-实战项目]]
