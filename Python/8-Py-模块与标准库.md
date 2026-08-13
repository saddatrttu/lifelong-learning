# 8. 模块与标准库

## 本章目标

理解模块与包的概念，掌握 import 的各种用法，认识常用标准库，学会用 pip 安装第三方库。

## 8.1 什么是模块

模块就是一个 `.py` 文件。把代码拆分成多个模块，可以让项目结构清晰、代码可复用：

```
project/
├── main.py          # 主程序（入口）
└── mymath.py        # 自定义模块
```

**mymath.py：**

```python
# mymath.py
PI = 3.14159

def circle_area(radius):
    return PI * radius ** 2

def square_area(side):
    return side * side
```

**main.py 导入并使用：**

```python
# main.py
import mymath

print(mymath.PI)                    # 3.14159
print(mymath.circle_area(2))        # 12.56636
print(mymath.square_area(5))        # 25
```

## 8.2 import 的四种写法

```python
# 方式一：import 模块名（使用时带模块名前缀）
import math
print(math.sqrt(16))      # 4.0

# 方式二：from 模块 import 名字（直接用）
from math import sqrt
print(sqrt(16))           # 4.0

# 方式三：导入多个名字
from math import sqrt, pi, floor
print(floor(3.7))         # 3

# 方式四：别名
import math as m
print(m.sqrt(16))         # 4.0
```

**import 搜索顺序：** 当前目录 → PYTHONPATH → 标准库 → site-packages（第三方库）

> 自己写的模块文件不要和标准库重名（如 `math.py`），否则会覆盖标准库。

## 8.3 `if __name__ == "__main__"` 的用法

模块被导入时，里面的代码会被执行。用这个判断区分"直接运行"和"被导入"：

```python
# mymath.py
PI = 3.14159

def circle_area(radius):
    return PI * radius ** 2

# 只有直接运行 python mymath.py 时才执行测试代码
if __name__ == "__main__":
    print("测试：圆面积(2) =", circle_area(2))
```

```python
# main.py
import mymath    # 不会打印测试代码
print(mymath.circle_area(3))
```

**解释：** 直接运行时 `__name__` 等于 `"__main__"`；被导入时等于模块名 `"mymath"`。

## 8.4 包：文件夹形式的模块集合

```
project/
├── main.py
└── utils/               # 包
    ├── __init__.py      # 标记这是包（Python 3.3+ 可以省略）
    ├── file_ops.py
    └── date_ops.py
```

```python
# main.py
from utils import file_ops
from utils.file_ops import read_config

file_ops.read_config()
read_config()
```

## 8.5 常用标准库（重点）

| 模块 | 用途 | 示例 |
|------|------|------|
| `math` | 数学运算 | `math.sqrt(16)` |
| `random` | 随机数 | `random.randint(1, 100)` |
| `datetime` | 日期时间 | `datetime.now()` |
| `os` | 操作系统接口 | `os.getcwd()` |
| `sys` | 系统参数 | `sys.argv` |
| `json` | JSON 处理 | `json.dumps(data)` |
| `re` | 正则表达式 | `re.findall(pattern, text)` |
| `collections` | 高级容器 | `Counter(list)` |
| `pathlib` | 路径处理 | `Path("a/b").exists()` |

### 实战示例

```python
import math
import random
import datetime
import os
import sys

# math：数学函数
print(math.sqrt(144))          # 12.0
print(math.floor(3.7))         # 3
print(math.ceil(3.2))          # 4

# random：随机
print(random.randint(1, 6))    # 1-6 随机整数（掷骰子）
print(random.choice(["石头", "剪刀", "布"]))
print(random.sample(range(10), 3))   # 随机取 3 个不重复

# datetime：时间
now = datetime.datetime.now()
print(now.strftime("%Y-%m-%d %H:%M:%S"))   # 2026-08-13 14:30:00

# os / sys：系统信息
print(os.getcwd())             # 当前目录
print(sys.version)             # Python 版本
```

### collections.Counter 计数神器

```python
from collections import Counter

words = ["apple", "banana", "apple", "orange", "banana", "apple"]
counter = Counter(words)
print(counter)                    # Counter({'apple': 3, 'banana': 2, 'orange': 1})
print(counter.most_common(2))     # [('apple', 3), ('banana', 2)]
```

## 8.6 pip 安装第三方库

pip 是 Python 的包管理器：

```powershell
# 安装
pip install requests

# 指定版本
pip install requests==2.32.0

# 卸载
pip uninstall requests

# 查看已安装
pip list

# 导出/导入依赖
pip freeze > requirements.txt
pip install -r requirements.txt
```

**常用第三方库：**

| 库 | 用途 |
|----|------|
| `requests` | HTTP 请求（爬虫、API 调用） |
| `flask` / `fastapi` | Web 开发 |
| `pandas` / `numpy` | 数据分析 |
| `pytest` | 自动化测试（见自动化测试指南） |
| `beautifulsoup4` | HTML 解析 |

## 8.7 综合示例：日期工具模块

创建一个自己的工具模块并测试：

```python
# date_tools.py
import datetime

def today_str():
    """返回今天的日期字符串"""
    return datetime.date.today().isoformat()

def days_until(target_date_str):
    """计算距离目标日期还有多少天"""
    target = datetime.date.fromisoformat(target_date_str)
    delta = target - datetime.date.today()
    return delta.days

if __name__ == "__main__":
    print("今天：", today_str())
    print("距离 2027-01-01 还有", days_until("2027-01-01"), "天")
```

```python
# main.py
from date_tools import today_str, days_until

print(f"今天是 {today_str()}")
print(f"距离年底还有 {days_until('2026-12-31')} 天")
```

**运行结果：**

```
今天是 2026-08-13
距离年底还有 140 天
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `ModuleNotFoundError` | 模块不存在/没安装/路径不对 | `pip install` 或检查文件位置 |
| 模块名和标准库重名 | 自定义文件覆盖标准库 | 改名 |
| 被导入时执行了不该执行的代码 | 缺少 `if __name__` 判断 | 用 `if __name__ == "__main__"` |

## 延伸阅读

- 函数 → 模块的基石 → [[7-Py-函数]]
- 读写文件（os/json 的应用）→ [[9-Py-文件操作]]
- 第三方库 requests 实战 → [[13-Py-实战项目]]
