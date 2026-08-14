# 4. 配置持久化 persist.py

> 把各协议模块上一次改动存到 `config.json`（UTF-8），下次启动自动恢复。实现极简：读全量 + 写全量 + 原子替换。

## 4.1 核心函数

```python
CONFIG_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "config.json")

def load_all() -> dict                 # 读整个 config.json，异常返回 {}
def load_section(name) -> dict         # 取某模块段，缺失返回 {}
def save_all(data)                     # 写整个 config.json（原子）
def save_section(name, section)        # 读全量 → 覆盖某段 → save_all
```

## 4.2 原子写入（防写坏）

```python
def save_all(data):
    d = os.path.dirname(CONFIG_PATH)
    fd, tmp = tempfile.mkstemp(dir=d, prefix=".cfg_", suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        os.replace(tmp, CONFIG_PATH)   # 原子替换
    except Exception:
        os.remove(tmp)                 # 失败清理临时文件
```

**为什么先写临时文件再替换**：避免写一半断电/崩溃把配置写坏。`os.replace` 在同一文件系统上是原子操作。

## 4.3 约定

1. **只存纯数据**（字符串/数字/bool/list/dict），不存 tk 变量/控件
2. **读取方负责填回控件**，任何字段缺失都要能容忍（用 `.get` 给默认值）
3. 关闭时 `main._on_close` 调各 tab 的 `save_cfg()`

## 4.4 config.json 结构（示例）

```json
{
  "modbus": {"mode": "RTU", "rtu": {...}, "tcp": {...}, "endian": "大端",
             "dtype": "整型", "times": "1", "tasks": []},
  "mqtt": {"server": {...}, "topics": {"sub": [], "pub": []},
           "poll": {...}, "templates": [...]},
  "accept": {"templates": {}, "sn": 44, "uuid": "", "sample_n": 3, "auto": true},
  "meter": {"protocol": "DL/T645", "serial": {...}, "meters": [...]}
}
```

> ⚠️ `mqtt.server` 里有明文密码。私有仓库可用；若仓库公开/加协作者需脱敏（改 `config.example.json` + gitignore 真实配置）。

## 延伸阅读

- 各 tab 的 `_load_cfg`/`save_cfg` 用法 → [[8-Swj-三协议模块]]
- 验收配置节 `accept` 的消费方 → [[9-Swj-验收主页面与并发执行]]
