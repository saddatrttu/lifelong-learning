# 1. 环境搭建与运行

## 本章目标

安装 Python 开发环境，运行你的第一个 Python 程序，理解解释器与脚本两种运行方式。

## 1.1 安装 Python

### Windows

1. 访问官网 https://www.python.org/downloads/ 下载最新版本（如 3.12/3.13）
2. 运行安装包，**务必勾选 "Add Python to PATH"**（这是新手最常见的坑）
3. 验证安装：

```powershell
python --version
# 输出示例：Python 3.13.0
```

> 如果提示 `python 不是内部或外部命令`，说明安装时没勾选 PATH，重装或手动把 Python 安装目录加入系统环境变量。

### macOS / Linux

- macOS：`brew install python`
- Ubuntu/Debian：`sudo apt install python3`
- 验证：`python3 --version`

## 1.2 两种运行方式

### 方式一：交互式解释器（适合练习）

```powershell
python
```

进入后出现 `>>>` 提示符，直接输入代码回车即执行：

```
>>> 1 + 1
2
>>> print("hello")
hello
>>> exit()
```

### 方式二：脚本文件（适合正式程序）

用任何文本编辑器（推荐 VS Code）写一个 `hello.py`：

```python
# hello.py
print("Hello, Python!")
```

运行：

```powershell
python hello.py
# 输出：Hello, Python!
```

## 1.3 第一个程序：逐步讲解

```python
# 1. print() 是 Python 自带的"打印"函数，把内容输出到屏幕
print("Hello, World!")

# 2. 可以一次打印多个内容，逗号分隔会自动加空格
print("我的名字是", "小明", "今年", 18, "岁")

# 3. 字符串可以拼接（后面章节详细讲）
name = "小明"
print("你好，" + name + "！")
```

**运行结果：**

```
Hello, World!
我的名字是 小明 今年 18 岁
你好，小明！
```

**逐行讲解：**
- `print(...)` — 函数调用，括号里的内容会被打印
- `"Hello, World!"` — 字符串字面量（带引号的文本）
- `name = "小明"` — 变量赋值：把 `"小明"` 存进名为 `name` 的变量
- `+` — 字符串拼接操作符

## 1.4 注释

注释是写给**人看**的说明，Python 执行时忽略它们：

```python
# 这是单行注释，以 # 开头
print("代码")  # 行尾注释

"""
这是多行注释：
用三个引号包裹，
可以写多行说明。
"""
```

**为什么要写注释？** 代码是给人读的，几个月后你会忘了自己写的是什么。养成写注释的习惯。

## 1.5 常见错误与解决

| 错误 | 原因 | 解决 |
|------|------|------|
| `SyntaxError: invalid syntax` | 语法错误，漏了括号/引号/冒号 | 检查代码拼写 |
| `NameError: name 'xx' is not defined` | 变量名写错或未定义 | 检查拼写是否一致 |
| `python 不是内部或外部命令` | 未加入 PATH | 重装时勾选 Add to PATH |
| 中文乱码 | 文件编码问题 | 确认保存为 UTF-8 编码 |

## 1.6 推荐学习工具

| 工具 | 用途 |
|------|------|
| VS Code + Python 插件 | 写代码的主力编辑器，免费 |
| IDLE | Python 自带，最简单的入门工具 |
| Jupyter Notebook | 适合学习笔记与数据分析 |

## 延伸阅读

- 变量到底怎么存数据 → [[2-Py-变量与数据类型]]
- 学会输入输出交互 → [[3-Py-字符串]]
