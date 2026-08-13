# 4. 列表与元组

## 本章目标

掌握列表（可修改）和元组（不可修改）两种序列类型，学会增删改查、切片、遍历和推导式。

## 4.1 列表：可变的"收纳箱"

列表用 `[]` 定义，可以装任意类型的数据：

```python
fruits = ["苹果", "香蕉", "橘子"]
numbers = [1, 2, 3, 4, 5]
mixed = [1, "hello", 3.14, True, None]   # 混合类型
empty = []                                # 空列表

print(fruits)
print(mixed)
```

## 4.2 访问与修改

```python
fruits = ["苹果", "香蕉", "橘子"]

# 访问（和字符串一样用索引）
print(fruits[0])     # 苹果
print(fruits[-1])    # 橘子
print(fruits[0:2])   # ['苹果', '香蕉']  切片

# 修改
fruits[1] = "草莓"
print(fruits)        # ['苹果', '草莓', '橘子']

# 添加
fruits.append("西瓜")        # 末尾加一个
print(fruits)                # ['苹果', '草莓', '橘子', '西瓜']
fruits.insert(1, "葡萄")     # 指定位置插入
print(fruits)                # ['苹果', '葡萄', '草莓', '橘子', '西瓜']

# 删除
fruits.remove("草莓")        # 按值删除（删第一个匹配）
print(fruits)                # ['苹果', '葡萄', '橘子', '西瓜']
popped = fruits.pop()        # 弹出最后一个，并返回它
print(popped, fruits)        # 西瓜 ['苹果', '葡萄', '橘子']
del fruits[1]                # 按索引删除
print(fruits)                # ['苹果', '橘子']
```

## 4.3 常用操作

```python
nums = [3, 1, 4, 1, 5, 9, 2]

print(len(nums))        # 7  长度
print(max(nums))        # 9  最大值
print(min(nums))        # 1  最小值
print(sum(nums))        # 25 求和
print(sorted(nums))     # [1, 1, 2, 3, 4, 5, 9]  排序（返回新列表）

print(3 in nums)        # True 判断存在
print(nums.count(1))    # 2  统计出现次数
print(nums.index(5))    # 4  查找索引

nums.sort()             # 原地排序（改变原列表）
print(nums)             # [1, 1, 2, 3, 4, 5, 9]
nums.reverse()          # 原地反转
print(nums)             # [9, 5, 4, 3, 2, 1, 1]
```

> `sorted()` 返回新列表不改原列表；`sort()` 直接修改原列表。注意区分。

## 4.4 遍历列表

```python
fruits = ["苹果", "香蕉", "橘子"]

# 直接遍历元素
for fruit in fruits:
    print(f"我喜欢{fruit}")

# 同时拿索引和元素：enumerate
for index, fruit in enumerate(fruits):
    print(f"第{index}个是{fruit}")
```

**运行结果：**

```
我喜欢苹果
我喜欢香蕉
我喜欢橘子
第0个是苹果
第1个是香蕉
第2个是橘子
```

## 4.5 元组：不可修改的列表

元组用 `()` 定义，**创建后不能修改**：

```python
point = (3, 5)          # 坐标
rgb = (255, 0, 128)     # 颜色

print(point[0])     # 3
print(point[1])     # 5
# point[0] = 9     # 会报错：TypeError: 'tuple' object does not support item assignment

# 元组的解包（重要特性）
x, y = point
print(x, y)         # 3 5

# 一个元素的元组要加逗号
single = (1,)
print(type(single))  # <class 'tuple'>
```

**元组适用场景：**
- 坐标、颜色、日期等**不该被修改**的数据
- 作字典的键（列表不行，因为列表可变）
- 函数返回多个值（后面章节会用到）

## 4.6 列表推导式（重点）

用一行代码生成列表，替代 for 循环：

```python
# 传统写法
squares = []
for i in range(1, 6):
    squares.append(i ** 2)
print(squares)      # [1, 4, 9, 16, 25]

# 推导式写法（等价）
squares2 = [i ** 2 for i in range(1, 6)]
print(squares2)     # [1, 4, 9, 16, 25]

# 带条件：只保留偶数
evens = [i for i in range(10) if i % 2 == 0]
print(evens)        # [0, 2, 4, 6, 8]

# 处理字符串列表
names = ["alice", "bob", "carol"]
upper_names = [n.upper() for n in names]
print(upper_names)  # ['ALICE', 'BOB', 'CAROL']
```

**推导式结构**：`[表达式 for 变量 in 序列 if 条件]`

## 4.7 列表复制的重要坑

```python
a = [1, 2, 3]

# 错误：= 只是让 b 指向同一个列表
b = a
b.append(4)
print(a)    # [1, 2, 3, 4]  a 也被改了！

# 正确：用 [:] 或 copy() 创建副本
c = a[:]
c.append(5)
print(a)    # [1, 2, 3, 4]  不受影响
print(c)    # [1, 2, 3, 4, 5]

d = a.copy()
print(d)    # [1, 2, 3, 4]
```

**讲解**：`b = a` 只是复制了引用（两个名字指向同一个盒子），`b.append` 改的是同一个列表。要独立副本必须用 `a[:]` 或 `a.copy()`。

## 4.8 综合示例：成绩统计

```python
# 学生成绩统计
scores = [85, 92, 78, 65, 88, 96, 70]

print(f"班级人数：{len(scores)}")
print(f"最高分：{max(scores)}")
print(f"最低分：{min(scores)}")
print(f"平均分：{sum(scores) / len(scores):.1f}")

# 及格人数（>= 60）
passed = [s for s in scores if s >= 60]
print(f"及格人数：{len(passed)}")

# 90 分以上优秀名单
excellent = [s for s in scores if s >= 90]
print(f"优秀人数：{len(excellent)}，成绩：{excellent}")
```

**运行结果：**

```
班级人数：7
最高分：96
最低分：65
平均分：82.0
及格人数：7
优秀人数：2，成绩：[92, 96]
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| 修改元组报 `TypeError` | 元组不可变 | 用列表代替 |
| `b = a` 后改动影响 `a` | 复制了引用而非数据 | 用 `a[:]` 或 `a.copy()` |
| `IndexError` | 索引越界 | 先 `len()` 确认 |
| 推导式漏掉 `if` | 结果包含不想要的元素 | 检查条件 |

## 延伸阅读

- 字典与集合（另一种容器）→ [[5-Py-字典与集合]]
- for 循环详解 → [[6-Py-条件与循环]]
- range() 是什么 → [[6-Py-条件与循环]]
