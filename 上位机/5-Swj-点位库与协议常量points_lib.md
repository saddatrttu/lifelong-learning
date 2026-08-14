# 5. 点位库与协议常量 points_lib.py

> **数据驱动**的核心：把飞奕中央空调的 Modbus 寄存器映射、MQTT 命令/字段/错误码全部集中成常量字典，纯 Python 无 tk，供界面与判定引擎共用。本笔记讲清常量含义 + 点位工厂函数的实现。

## 5.1 Modbus 内机参数块（表6）

```python
_BASE   = 24000          # 通道1 第1台内机参数块起点
_STRIDE = 160 * 16       # 每通道 160 台，每台 16 寄存器
_IU_SIZE = 16            # 每台内机 16 个寄存器
```

**地址公式**：通道 M 第 N 台起点 = `D24000 + 160*16*(M-1) + 16*(N-1)`。
即：每台占 16 个寄存器，每通道 160 台（160×16=2560 寄存器 = `_STRIDE`），下一个通道从上一个通道末尾接着排。

`IU_FIELD_META` 字段（offset 相对该台起点）：

| 字段 | 名称 | offset | 读写 | 默认判据 |
|------|------|--------|------|---------|
| `ou` | 外机地址 | 0 | 读 | any |
| `iu` | 内机地址 | 1 | 读 | any |
| `lock` | 锁定 | 2 | 写 | range 0~1 |
| `o` | 开关 | 3 | 写 | range 0~1 |
| `ts` | 设定温度 | 4 | 写 | range 16~31 |
| `w` | 模式 | 5 | 写 | any |
| `fs` | 风速 | 6 | 写 | any |
| `rt` | 回风温度 | 10 | 读 | range 5~40 |
| `runmin` | 当日运行分钟 | 13 | 读 | range 0~1440 |
| `err` | 故障码 | 14 | 读 | any |

## 5.2 系统信息点与群控点

```python
SYSTEM_POINTS = {
    "device_id": {"addr": 0, "fc": "03", "rw": "read", "expect": {"mode": "any"}},
    "firmware":  {"addr": 6, "fc": "03", ...},
    "channel_count": {"addr": 31, ...}, "channel_addr": {"addr": 32, ...},
    "channel_stat": {"addr": 3300, ...},
}

GROUP_CTRL_POINTS = {
    "single_channel": {"fc": "10", "rw": "write",
                       "addr_o": 23592, "addr_ts": 23593,
                       "expect": {"mode": "readback", "sample_n": 3}},
    "all_channel":    {"fc": "10", "rw": "write",
                       "addr_o": 23886, "addr_ts": 23887, ...},
}
```

> 注意：23591 是「锁定」寄存器，不是群控开关；群控开关/温度才是正确点位。

## 5.3 点位工厂函数实现

### iu_point（核心地址计算）

```python
def iu_point(channel, iu_no, field):
    meta = IU_FIELD_META[field]
    base = _BASE + _STRIDE * (channel - 1) + _IU_SIZE * (iu_no - 1)
    return {"name": meta["name"], "addr": base + meta["offset"], "fc": "03",
            "rw": meta["rw"], "expect": dict(meta["expect"])}
```

**实现逻辑**：`base` 算出「通道 M 第 N 台」的参数块起点，再加字段的 `offset` 得到具体寄存器地址。注意 `dict(meta["expect"])` 是**浅拷贝**——避免返回共享的 expect dict 引用，防止调用方修改污染全局常量。

### system_point / group_ctrl_point

```python
def system_point(key):
    p = SYSTEM_POINTS[key]
    return dict(p)                  # 同样浅拷贝，防外部修改

def group_ctrl_point(key):
    p = GROUP_CTRL_POINTS[key]
    return dict(p)
```

### group_ctrl_addr

```python
def group_ctrl_addr(key, field):
    p = GROUP_CTRL_POINTS[key]
    if field == "o":  return p["addr_o"]    # 开关
    if field == "ts": return p["addr_ts"]   # 温度
    raise ValueError(f"群控不支持字段 {field}（仅 o/ts）")
```

**作用**：群控写时按字段名解析具体寄存器地址（开关和温度是两个独立寄存器）。只支持 `o`/`ts`，其他字段直接抛异常。

## 5.4 MQTT 命令枚举 MQTT_CMDS

```python
MQTT_CMDS = {
    "online": {"desc": "上线比对/应答", "kind": "read"},
    "general_read":  {"desc": "通用信息读取", "kind": "read"},
    "general_write": {"desc": "通用信息写入(校时/重启等)", "kind": "control", "danger": True},
    "status_read":   {"desc": "空调状态读取", "kind": "read"},
    "control_write": {"desc": "空调控制下发", "kind": "control",
                      "fields": ["addrs", "onOff", "tempSet", "workMode", "fanSpeed", "fanDirect"]},
    "lock_write":    {"desc": "空调锁定/解锁", "kind": "control"},
    "lock_status_read": {"desc": "锁定状态读取", "kind": "read"},
    "property_read": {"desc": "属性读取", "kind": "read"},
    "property_write":{"desc": "属性写入", "kind": "control"},
    "energy_read":   {"desc": "能耗读取", "kind": "read"},
    "node_online":   {"desc": "节点在线状态", "kind": "read"},
    "node_reset":    {"desc": "节点复位", "kind": "control", "danger": True},
    "schedule_write": {"desc": "定时写入", "kind": "control", "danger": True},
    "schedule_read": {"desc": "定时读取", "kind": "read"},
    "schedule_delete":{"desc": "定时删除", "kind": "control"},
    "device_report_config": {"desc": "设备状态上报(触发)", "kind": "control"},
}
```

`kind` 分 read（读）/ control（控制下发）；`danger` 标记校时/重启/定时等强副作用命令。

## 5.5 MQTT 状态字段与取值范围

```python
MQTT_STATUS_FIELDS = {
    "a": "地址", "o": "开关", "ts": "设定温度", "w": "模式",
    "fs": "风速", "fd": "风向", "acs": "报警码", "rt": "回风温度",
}

MQTT_FIELD_RANGE = {
    "o": (0,1), "ts": (16,31), "w": (0,14), "fs": (0,7), "fd": (0,7), "rt": (5,40),
}
```

## 5.6 错误码 ERROR_CODES

```python
ERROR_CODES = {
    0: "正常", 1: "通用错误", 2: "JSON结构错误", 3: "body不合法", 4: "日期错误",
    5: "存储错误", 6: "硬件错误", 7: "地址域错误", 8: "模式冲突",
    9: "不支持", 10: "数据不存在", 11: "认证失败",
}
```

## 延伸阅读

- 这些常量怎么被判定引擎使用 → [[6-Swj-验收判定引擎accept_core]]
- 协议本身的帧/报文细节 → [[10-Swj-协议知识]]
