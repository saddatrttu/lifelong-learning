# 2. 变量与数据类型

## 本章目标

理解变量的本质，掌握 Python 的 5 种基本数据类型和类型转换。

## 2.1 变量：给数据起名字

变量就像一个**贴了标签的盒子**，把数据放进去，通过标签取用：

```python
# 变量赋值：名字 = 值
age = 18            # 整数
price = 9.9         # 小数
name = "小明"       # 文本
is_student = True   # 布尔
nothing = None      # 空值

print(age)      # 18
print(name)     # 小明
```

**变量命名规则：**
- 只能包含字母、数字、下划线，且**不能以数字开头**
- 不能使用 Python 关键字（如 `if`、`for`、`class`）
- 区分大小写：`Age` 和 `age` 是两个不同变量

```python
# 合法：user_name, age2, _temp, myName
# 不合法：2name, my-name, class
```

**Python 变量可以随时改类型**（动态类型）：

```python
x = 10      # 先是整数
x = "abc"   # 变成字符串
x = 3.14    # 又变成小数
print(x)    # 3.14
```

## 2.2 基本数据类型

| 类型 | 中文名 | 示例 | 说明 |
|------|--------|------|------|
| `int` | 整数 | `42`, `-7`, `0` | 无小数部分 |
| `float` | 浮点数 | `3.14`, `-0.5`, `2.0` | 带小数 |
| `str` | 字符串 | `"hello"`, `'world'` | 文本，单双引号皆可 |
| `bool` | 布尔值 | `True`, `False` | 只有两个值 |
| `NoneType` | 空值 | `None` | 表示"没有值" |

**用 `type()` 查看类型：**

```python
print(type(42))      # <class 'int'>
print(type(3.14))    # <class 'float'>
print(type("abc"))   # <class 'str'>
print(type(True))    # <class 'bool'>
print(type(None))    # <class 'NoneType'>
```

## 2.3 数字运算

```python
a = 10
b = 3

print(a + b)    # 13   加法
print(a - b)    # 7    减法
print(a * b)    # 30   乘法
print(a / b)    # 3.3333333333333335   除法（结果是小数）
print(a // b)   # 3    整除（向下取整）
print(a % b)    # 1    取余
print(a ** b)   # 1000 幂运算（10的3次方）
```

**注意 `//` 和 `/` 的区别**：`10 / 3` 得到 `3.33...`，`10 // 3` 得到 `3`（丢掉小数）。

## 2.4 字符串的简单操作

```python
greeting = "Hello"
target = "Python"

# 拼接
print(greeting + " " + target)    # Hello Python

# 重复
print("ha" * 3)                   # hahaha

# 长度
print(len("Python"))              # 6

# 索引（从 0 开始）
word = "Python"
print(word[0])    # P
print(word[1])    # y
print(word[-1])   # n  （-1 表示最后一个）
```

## 2.5 类型转换

不同类型之间可以显式转换：

```python
# str -> int / float
num_str = "123"
print(int(num_str))      # 123
print(float("3.5"))      # 3.5

# int / float -> str
print(str(42))           # "42"

# 数字 -> 布尔
print(bool(0))           # False（0 是假）
print(bool(1))           # True
print(bool(""))          # False（空字符串是假）
print(bool("abc"))       # True
```

**常见坑**：把字符串转数字时，内容必须是数字，否则报错：

```python
# 会报错：ValueError: invalid literal for int()
int("abc")
```

## 2.6 综合示例：BMI 计算器

把本章知识串起来：

```python
# BMI 计算器：体重(kg) / 身高(m) 的平方
weight = 65.5        # 体重（公斤）
height = 1.75        # 身高（米）

bmi = weight / (height ** 2)
print("你的 BMI 是：", bmi)

# 保留两位小数
print("保留两位小数：", round(bmi, 2))

# 类型转换示例
print("BMI 的字符串形式是：" + str(round(bmi, 2)))
```

**运行结果：**

```
你的 BMI 是： 21.387755102040817
保留两位小数： 21.39
BMI 的字符串形式是：21.39
```

**讲解：**
- `height ** 2` 求身高的平方
- `round(bmi, 2)` 保留两位小数
- `str(...)` 把数字转成字符串才能用 `+` 拼接

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `TypeError: can only concatenate str` | 字符串和数字直接用 `+` | 用 `str()` 转换 |
| `NameError: name 'x' is not defined` | 变量名拼写不一致 | 检查大小写 |
| `ValueError` | 字符串转数字时内容不是数字 | 确认字符串是数字 |

## 延伸阅读

- 字符串还有更多玩法 → [[3-Py-字符串]]
- 一次存多个数据 → [[4-Py-列表与元组]]
