# 7. 函数

## 本章目标

理解函数的定义与调用，掌握参数的各种形式（默认值、关键字、可变参数），理解返回值、作用域和 lambda。

## 7.1 为什么需要函数

函数是**打包的代码块**，一次定义、反复调用，让代码可复用、好维护：

```python
# 不写函数：重复代码
print("计算两个数：")
a = 3 + 5
print(a)
b = 10 + 20
print(b)
c = 100 + 200
print(c)

# 写函数：定义一次
def add(x, y):
    return x + y

print(add(3, 5))        # 8
print(add(10, 20))      # 30
print(add(100, 200))    # 300
```

## 7.2 定义与调用

```python
def 函数名(参数1, 参数2):
    # 函数体（缩进）
    return 返回值

# 示例
def greet(name):
    message = f"你好，{name}！"
    return message

# 调用
result = greet("小明")
print(result)       # 你好，小明！
```

**要点：**
- `def` 关键字定义函数
- 参数可省略（无参函数），`return` 可省略（默认返回 `None`）

## 7.3 参数详解

### 默认参数

```python
def greet(name, greeting="你好"):
    print(f"{greeting}，{name}！")

greet("小明")             # 你好，小明！
greet("小明", "早上好")   # 早上好，小明！
```

> 默认参数必须放在非默认参数**后面**。默认参数只在函数定义时求值一次，不要用可变对象（列表/字典）当默认值！

```python
# 反例：可变默认值是个坑
def add_item(item, items=[]):   # 错误示范
    items.append(item)
    return items

print(add_item("a"))    # ['a']
print(add_item("b"))    # ['a', 'b']   ← 问题：共享同一个列表！

# 正确做法
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### 关键字参数

调用时按名字传参，顺序可以乱：

```python
def introduce(name, age, job):
    print(f"{name}，{age}岁，职业{job}")

introduce(age=25, job="工程师", name="张伟")   # 顺序随意
```

### 可变参数：`*args`

接收任意数量的位置参数（打包成元组）：

```python
def total(*nums):
    print(f"接收的参数：{nums}")
    return sum(nums)

print(total(1, 2, 3))        # 6
print(total(10, 20, 30, 40)) # 100
```

### 关键字可变参数：`**kwargs`

接收任意数量的关键字参数（打包成字典）：

```python
def show_info(**info):
    print(info)

show_info(name="小明", age=18, city="北京")
# {'name': '小明', 'age': 18, 'city': '北京'}
```

### 全部组合（顺序固定）

```python
def func(普通参数, 默认参数=1, *args, **kwargs):
    pass
```

## 7.4 返回值

```python
# 返回一个值
def square(x):
    return x * x

# 返回多个值（实际是元组）
def get_min_max(nums):
    return min(nums), max(nums)

result = get_min_max([3, 1, 5, 2])
print(result)              # (1, 5)
a, b = result              # 解包
print(a, b)                # 1 5

# 没有 return 时返回 None
def do_nothing():
    pass
print(do_nothing())        # None
```

## 7.5 作用域：变量能见范围

```python
x = 10          # 全局变量

def func():
    x = 20      # 局部变量（和全局 x 无关）
    print("函数内：", x)   # 20

func()
print("函数外：", x)       # 10

# 想修改全局变量：global 关键字（慎用）
def change():
    global x
    x = 99

change()
print(x)        # 99
```

**规则：** 函数内默认只能**读**全局变量；想**改**全局变量必须 `global`。一般建议避免修改全局变量，用参数传入、返回值传出。

## 7.6 lambda 匿名函数

一行表达式函数，适合简单的逻辑：

```python
# 普通函数
def double(x):
    return x * 2

# lambda 等价写法
double = lambda x: x * 2
print(double(5))    # 10

# 典型应用：配合 sorted 排序
students = [
    {"name": "小明", "score": 92},
    {"name": "小红", "score": 88},
    {"name": "小刚", "score": 95},
]

# 按成绩排序
by_score = sorted(students, key=lambda s: s["score"])
print([s["name"] for s in by_score])    # ['小红', '小明', '小刚']

# 按成绩降序
by_score_desc = sorted(students, key=lambda s: s["score"], reverse=True)
print([s["name"] for s in by_score_desc])  # ['小刚', '小明', '小红']
```

**讲解：** `lambda 参数: 表达式` —— 参数在冒号左边，返回值是表达式本身。`key=` 参数指定"按什么排序"。

## 7.7 综合示例：简单的计算器

```python
def add(a, b): return a + b
def subtract(a, b): return a - b
def multiply(a, b): return a * b
def divide(a, b):
    if b == 0:
        return "除数不能为 0"
    return a / b

# 用字典实现"命令分发"（比 if-elif 链更优雅）
operations = {
    "+": add,
    "-": subtract,
    "*": multiply,
    "/": divide,
}

def calculate(a, op, b):
    func = operations.get(op)
    if func is None:
        return f"未知运算符: {op}"
    return func(a, b)

print(calculate(10, "+", 5))    # 15
print(calculate(10, "-", 5))    # 5
print(calculate(10, "*", 5))    # 50
print(calculate(10, "/", 0))    # 除数不能为 0
print(calculate(10, "^", 5))    # 未知运算符: ^
```

**讲解：** 把函数本身存进字典作为值，`operations[op]` 取出对应函数再调用 —— 这是"函数是一等公民"的体现，也是命令模式的简化版。

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `NameError` | 变量作用域搞混 | 区分全局/局部变量 |
| 可变默认参数共享 | `def f(x, items=[])` | 用 `None` 代替 |
| 忘记 return | 函数返回 None | 确认有 return |
| lambda 里写多条语句 | lambda 只能单表达式 | 改用普通函数 |

## 延伸阅读

- 函数是代码复用 → 练习用 [[6-Py-条件与循环]]
- 把函数组织成模块 → [[8-Py-模块与标准库]]
- 函数进阶：装饰器、生成器 → [[12-Py-进阶特性]]
