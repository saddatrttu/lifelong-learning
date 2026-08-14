# 6. 条件与循环

## 本章目标

掌握 if 条件判断和 for/while 循环，理解 break/continue 以及 range() 的用法。

## 6.1 比较运算与布尔值

条件判断的核心是比较：

```python
a = 10
b = 3

print(a > b)      # True
print(a < b)      # False
print(a == b)     # False   相等（注意是两个 =）
print(a != b)     # True    不等
print(a >= 10)    # True
print(a <= 5)     # False

# 逻辑运算
print(True and False)   # False   与：都要真
print(True or False)    # True    或：一个真即可
print(not True)         # False   非：取反
```

## 6.2 if / elif / else

```python
score = 85

if score >= 90:
    print("优秀")
elif score >= 60:
    print("及格")
else:
    print("不及格")
# 输出：及格
```

**关键语法点：**
- 条件后面要**冒号 `:`**
- 代码块用**缩进**（4 个空格），不用花括号
- `elif` 可以有多个，`else` 可选

### 多条件判断示例

```python
age = 20

if age < 0:
    print("年龄不合法")
elif age < 18:
    print("未成年")
elif age < 60:
    print("成年")
else:
    print("老年")
# 输出：成年
```

### 条件表达式（三元）

```python
age = 20
status = "成年" if age >= 18 else "未成年"
print(status)    # 成年
```

## 6.3 for 循环：遍历序列

```python
# 遍历列表
fruits = ["苹果", "香蕉", "橘子"]
for fruit in fruits:
    print(f"我喜欢{fruit}")

# 遍历字符串
for ch in "ABC":
    print(ch)

# 遍历字典
scores = {"语文": 90, "数学": 85}
for subject, score in scores.items():
    print(f"{subject}: {score}")
```

**运行结果：**

```
我喜欢苹果
我喜欢香蕉
我喜欢橘子
A
B
C
语文: 90
数学: 85
```

## 6.4 range()：生成数字序列

```python
# range(结束)  0 到 结束-1
for i in range(5):
    print(i)          # 0 1 2 3 4

# range(起始, 结束)  起始 到 结束-1
for i in range(2, 5):
    print(i)          # 2 3 4

# range(起始, 结束, 步长)
for i in range(1, 10, 2):
    print(i)          # 1 3 5 7 9

# 倒序
for i in range(5, 0, -1):
    print(i)          # 5 4 3 2 1
```

**常见用法：固定次数循环**

```python
for i in range(3):
    print(f"第{i + 1}次执行")
# 第1次执行 / 第2次执行 / 第3次执行
```

## 6.5 while 循环：条件控制

while 在**条件为真**时一直执行：

```python
count = 0
while count < 3:
    print(f"count = {count}")
    count += 1        # 计数器必须递增，否则死循环！
# count = 0 / count = 1 / count = 2
```

**用 while 处理用户输入**（游戏常见的循环模式）：

```python
password = "123456"
attempt = 0

while True:
    guess = input("请输入密码：")
    attempt += 1
    if guess == password:
        print(f"密码正确，共尝试{attempt}次")
        break
    print("密码错误，再试一次")
```

> `input()` 是内置函数，读取用户输入（返回字符串）。这个程序会一直循环直到密码正确。

## 6.6 break / continue / else

```python
# break：立即结束整个循环
for i in range(10):
    if i == 3:
        break          # 到 3 就停
    print(i)           # 0 1 2

# continue：跳过本次，继续下一次
for i in range(5):
    if i == 2:
        continue       # 跳过 2
    print(i)           # 0 1 3 4

# 循环的 else：循环没被 break 结束时执行
for i in range(3):
    print(i)
else:
    print("循环正常结束")
# 0 1 2 循环正常结束
```

**循环 else 的实用场景（查找）：**

```python
# 判断是否有偶数
nums = [1, 3, 5, 7]
for n in nums:
    if n % 2 == 0:
        print("发现偶数")
        break
else:
    print("没有偶数")
# 输出：没有偶数
```

## 6.7 嵌套循环

```python
# 打印乘法表
for i in range(1, 4):
    for j in range(1, 4):
        print(f"{i}×{j}={i*j}", end="  ")
    print()     # 换行
```

**运行结果：**

```
1×1=1  1×2=2  1×3=3  
2×1=2  2×2=4  2×3=6  
3×1=3  3×2=6  3×3=9  
```

> `end="  "` 让 print 不换行，改用指定内容结尾。

## 6.8 综合示例：猜数字游戏

把本章所有知识组合起来：

```python
import random

# 生成 1-100 的随机数
target = random.randint(1, 100)
attempts = 0

print("猜数字游戏：范围 1-100")

while True:
    guess = int(input("请输入你的猜测："))
    attempts += 1

    if guess > target:
        print("太大了！")
    elif guess < target:
        print("太小了！")
    else:
        print(f"恭喜！猜对了！数字是 {target}，共猜了 {attempts} 次")
        break
```

**讲解：**
- `import random` 导入随机数模块（后面章节详解）
- `random.randint(1, 100)` 生成 1-100 随机整数
- `int(input(...))` 把用户输入转成整数
- `while True` + `break` 是"一直玩直到成功"的经典结构

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `IndentationError` | 缩进不对（空格/制表符混用） | 统一 4 个空格 |
| 忘记冒号 `:` | if/for/while 后必须有冒号 | 检查行尾 |
| 死循环 | while 条件永远为真且无 break | 确保循环内条件会变化 |
| `==` 写成 `=` | 比较写成赋值 | `=` 是赋值，`==` 是比较 |

## 延伸阅读

- 循环遍历列表 → [[4-Py-列表与元组]]
- 循环遍历字典 → [[5-Py-字典与集合]]
- 把重复代码封装成函数 → [[7-Py-函数]]
