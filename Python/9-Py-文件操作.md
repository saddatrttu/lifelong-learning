# 9. 文件操作

## 本章目标

掌握文件的读写、with 语句、路径处理（pathlib）和 JSON 数据处理。

## 9.1 读取文件

```python
# 方式一：open + read（记得关闭）
f = open("data.txt", "r", encoding="utf-8")
content = f.read()
print(content)
f.close()      # 必须手动关闭，忘记会浪费资源

# 方式二：with 语句（推荐，自动关闭）
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
print(content)
```

**with 的好处**：离开代码块自动关闭文件，即使出错也会关闭。永远用 with。

## 9.2 写入文件

```python
# "w" 覆盖写入（文件不存在会创建）
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("第一行\n")
    f.write("第二行\n")

# "a" 追加写入
with open("output.txt", "a", encoding="utf-8") as f:
    f.write("追加的第三行\n")

# 读取验证
with open("output.txt", "r", encoding="utf-8") as f:
    print(f.read())
```

**运行结果：**

```
第一行
第二行
追加的第三行
```

**模式对照：**

| 模式 | 含义 | 文件不存在 |
|------|------|-----------|
| `r` | 只读（默认） | 报错 |
| `w` | 写入（覆盖） | 创建 |
| `a` | 追加 | 创建 |
| `r+` | 读写 | 报错 |
| `x` | 新建写入（已存在报错） | 创建 |

> 中文文件必须指定 `encoding="utf-8"`，否则可能乱码或报错。

## 9.3 逐行读取

大文件不要一次 `read()` 全部读入内存，逐行处理：

```python
# 逐行读取
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())    # strip 去掉行尾换行符

# 或 readlines() 得到行列表
with open("data.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()
print(lines)
```

**统计文件行数：**

```python
count = 0
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        count += 1
print(f"共 {count} 行")
```

## 9.4 pathlib：现代路径处理

pathlib 把路径当对象处理，比字符串拼接更安全：

```python
from pathlib import Path

# 创建路径对象
p = Path("project/data/report.txt")
print(p.name)          # report.txt      文件名
print(p.stem)          # report          文件名（无扩展名）
print(p.suffix)        # .txt            扩展名
print(p.parent)        # project\data    父目录

# 拼接路径（用 / 运算符，自动处理分隔符）
base = Path("project")
file = base / "data" / "report.txt"
print(file)            # project\data\report.txt

# 检查
print(file.exists())   # False  是否存在
print(file.is_file())  # 是否是文件

# 创建目录
dirs = Path("new/dir/structure")
dirs.mkdir(parents=True, exist_ok=True)   # parents=True 创建所有层级

# 遍历目录
for item in Path(".").iterdir():
    print(item.name)
```

**glob 查找文件：**

```python
from pathlib import Path

# 找当前目录所有 .py 文件
for f in Path(".").glob("*.py"):
    print(f.name)

# 递归查找所有 .md 文件
for f in Path(".").rglob("*.md"):
    print(f.name)
```

## 9.5 JSON 数据处理

JSON 是程序间交换数据的标准格式，Python 内置支持：

```python
import json

# Python 字典 -> JSON 字符串
data = {
    "name": "小明",
    "age": 18,
    "scores": [92, 88, 95],
    "is_student": True
}

json_str = json.dumps(data, ensure_ascii=False, indent=2)
print(json_str)
```

**输出：**

```json
{
  "name": "小明",
  "age": 18,
  "scores": [
    92,
    88,
    95
  ],
  "is_student": true
}
```

```python
# JSON 字符串 -> Python 字典
text = '{"name": "小红", "age": 17}'
obj = json.loads(text)
print(obj["name"])       # 小红
print(obj["age"])        # 17

# 读写 JSON 文件
with open("student.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

with open("student.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
print(loaded["name"])    # 小明
```

> `ensure_ascii=False` 让中文正常显示（否则会变成 `\u5c0f` 转义）。

## 9.6 综合示例：配置管理器

一个简单的 JSON 配置文件读写工具：

```python
import json
from pathlib import Path

CONFIG_FILE = Path("config.json")

# 默认配置
DEFAULT_CONFIG = {
    "app_name": "My App",
    "version": "1.0",
    "debug": True,
    "max_retries": 3
}

def load_config():
    """读取配置，不存在则用默认配置创建"""
    if CONFIG_FILE.exists():
        with open(CONFIG_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    save_config(DEFAULT_CONFIG)
    return DEFAULT_CONFIG

def save_config(config):
    """保存配置"""
    with open(CONFIG_FILE, "w", encoding="utf-8") as f:
        json.dump(config, f, ensure_ascii=False, indent=2)

# 使用
config = load_config()
print(f"应用名：{config['app_name']}，版本：{config['version']}")

# 修改并保存
config["debug"] = False
config["max_retries"] = 5
save_config(config)
print("配置已更新")

# 再次读取验证
print(load_config())
```

**运行结果：**

```
应用名：My App，版本：1.0
配置已更新
{'app_name': 'My App', 'version': '1.0', 'debug': False, 'max_retries': 5}
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `FileNotFoundError` | 文件不存在 | 先 `exists()` 检查 |
| `UnicodeDecodeError` | 编码不对 | 指定 `encoding="utf-8"` |
| `PermissionError` | 文件被占用/无权限 | 关闭占用程序 |
| 忘记 close | 资源泄漏 | 用 with 语句 |

## 延伸阅读

- 模块与标准库 → [[8-Py-模块与标准库]]
- 文件操作出错的处理 → [[10-Py-异常处理]]
- JSON 配置在实战项目的应用 → [[13-Py-实战项目]]
