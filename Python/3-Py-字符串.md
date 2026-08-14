# 3. 字符串

## 本章目标

掌握字符串的创建、索引切片、格式化输出和常用方法。

## 3.1 创建字符串

```python
# 单引号、双引号都可以
s1 = 'hello'
s2 = "world"

# 三引号可以换行（多行文本）
s3 = """第一行
第二行
第三行"""
print(s3)

# 字符串里包含引号
s4 = '他说："你好"'        # 外层单引号，内层双引号
s5 = "It's a book"         # 外层双引号，内层单引号
print(s4)
print(s5)
```

**运行结果：**

```
第一行
第二行
第三行
他说："你好"
It's a book
```

## 3.2 索引与切片

字符串是**字符的有序序列**，可以用位置访问：

```python
word = "Python"

# 索引（正数从 0 开始，负数从末尾开始）
print(word[0])    # P
print(word[2])    # t
print(word[-1])   # n
print(word[-3])   # h

# 切片 [起始:结束:步长] —— 结束位置不含
print(word[0:3])    # Pyt  （第0到第2个）
print(word[1:])     # ython （从第1个到末尾）
print(word[:3])     # Pyt  （从开头到第2个）
print(word[:])      # Python（整个）
print(word[::2])    # Pto  （隔一个取一个）
print(word[::-1])   # nohtyP（倒序）
```

**切片记忆法**：`[起始:结束]` 取"起始 ≤ 索引 < 结束"，左闭右开。

## 3.3 字符串是"不可变"的

字符串一旦创建不能修改单个字符，但可以重新赋值：

```python
s = "hello"
# s[0] = "H"   # 会报错：TypeError

# 正确做法：重新赋值
s = "H" + s[1:]
print(s)      # Hello
```

## 3.4 常用方法（重点）

```python
text = "  Hello, Python World  "

print(text.lower())              # 全部小写：  hello, python world  
print(text.upper())              # 全部大写：  HELLO, PYTHON WORLD  
print(text.strip())              # 去掉首尾空格：Hello, Python World
print(text.replace("Python", "Java"))   # 替换：  Hello, Java World  
print(text.split(","))           # 按逗号分割成列表：['  Hello', ' Python World  ']
print(len(text))                 # 长度：23
print("Python" in text)          # 判断是否包含：True
print(text.startswith("  He"))   # 是否以...开头：True
print(text.endswith("  "))       # 是否以...结尾：True
```

**运行结果：**

```
  hello, python world  
  HELLO, PYTHON WORLD  
Hello, Python World
  Hello, Java World  
['  Hello', ' Python World  ']
23
True
True
True
```

> 注意：方法不改变原字符串（`text` 还是原样），而是**返回新字符串**。想保存结果要赋值：`text = text.lower()`。

## 3.5 格式化输出（三种方式）

### 方式一：`%` 格式化（老式）

```python
name = "小明"
age = 18
print("%s 今年 %d 岁" % (name, age))
# 输出：小明 今年 18 岁
```

### 方式二：`format()` 方法

```python
print("{} 今年 {} 岁".format(name, age))
print("{0} 今年 {1} 岁".format(name, age))       # 指定位置
print("{n} 今年 {a} 岁".format(n=name, a=age))   # 指定名字
# 输出：小明 今年 18 岁
```

### 方式三：f-string（现代推荐）

```python
# f-string：在字符串前加 f，用 {} 直接放变量和表达式
print(f"{name} 今年 {age} 岁")
print(f"明年他就 {age + 1} 岁了")
print(f"价格：{9.99:.2f} 元")       # 保留两位小数：价格：9.99 元
print(f"进度：{35 / 100:.0%}")      # 百分比：进度：35%
```

**运行结果：**

```
小明 今年 18 岁
明年他就 19 岁了
价格：9.99 元
进度：35%
```

**推荐**：永远用 f-string，最简洁直观。

## 3.6 数字转字符串的特殊格式

```python
# 在 f-string 中控制格式
pi = 3.1415926
print(f"{pi:.2f}")     # 3.14  保留2位小数
print(f"{pi:10.2f}")   # '      3.14'  总宽度10，右对齐
print(f"{pi:.0f}")     # 3  取整

# 补零
print(f"{42:05d}")     # 00042  宽度5，不足补0
```

## 3.7 综合示例：格式化名片

```python
# 用所学知识生成一张名片
name = "张伟"
age = 25
job = "软件工程师"
salary = 18000.5

card = f"""
==================
姓名：{name}
年龄：{age}
职业：{job}
月薪：{salary:.2f} 元
==================
"""
print(card)
```

**运行结果：**

```
==================
姓名：张伟
年龄：25
职业：软件工程师
月薪：18000.50 元
==================
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `IndexError: string index out of range` | 索引超出字符串长度 | 用 `len()` 确认长度 |
| 忘记 `.strip()` 导致比较失败 | 字符串带首尾空格 | 比较前先 `strip()` |
| f-string 里漏了 `f` 前缀 | 变量不生效，原样输出 `{name}` | 检查 `f` 前缀 |

## 延伸阅读

- 字符串.split() 返回的是列表 → [[4-Py-列表与元组]]
- 更多字符串方法查官方文档 https://docs.python.org/zh-cn/3/library/stdtypes.html#string-methods
