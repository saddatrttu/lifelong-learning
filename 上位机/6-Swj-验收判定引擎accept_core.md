# 6. 验收判定引擎 accept_core.py

> 验收测试的**核心大脑**，纯 Python 无 tk，可独立单测。负责：用例模型、Modbus 值解析/帧构造、MQTT sn 匹配、判定引擎、机器可读模板解析。本笔记逐函数讲实现。

## 6.1 用例模型 new_case

```python
def new_case(case_no, proto, name, kind, **kw):
    c = {
        "id": kw.pop("id", 0),
        "case_no": case_no or "", "proto": proto,
        "group": kw.pop("group", ""), "name": name, "kind": kind,
        "enabled": True,
        # modbus
        "point": None, "addr": None, "fc": None, "cnt": 1, "val": 0,
        # mqtt
        "cmd": None, "addrs": [], "body": None,
        # 判定
        "expect": {"mode": "any"},
        "timeout_ms": 3000,
    }
    c.update(kw)
    return c
```

**实现方式**：先给一整套默认字段，再用 `kw` 覆盖（`kw.pop("id"/"group", ...)` 同时起到「默认 + 取值」作用）。`kind ∈ read/write/group_ctrl/mqtt_cmd`。所有字段都有默认值，保证后续判定代码不用判 None。

## 6.2 Modbus 值解析与解码

```python
def modbus_parse_vals(text):
    text = str(text).strip()
    if not text:
        return [0]
    parts = [p for p in text.replace("，", ",").replace(" ", ",").split(",") if p != ""]
    return [int(p, 16) if p.lower().startswith("0x") else int(p) for p in parts]
```

**实现方式**：中文逗号「，」和空格都先统一替换成英文逗号，再 `split(",")`，过滤空串。每个值判断 `0x` 前缀决定按 16 进制还是 10 进制解析。

```python
def modbus_decode_val(raw, endian="大端", dtype="整型"):
    if endian == "小端":
        raw = ((raw & 0xFF) << 8) | (raw >> 8)     # 字节交换
    if dtype == "十六进制": return f"0x{raw:04X}"
    if dtype == "浮点":     return f"{raw / 10.0:.1f}"
    return str(raw)
```

**小端实现**：`((raw & 0xFF) << 8) | (raw >> 8)` 是 16bit 高低字节交换的标准写法（低字节移到高 8 位，高字节右移到低 8 位）。

## 6.3 Modbus 线路帧构造

### modbus_crc16

```python
def modbus_crc16(data):
    crc = 0xFFFF
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = ((crc >> 1) ^ 0xA001) if (crc & 1) else (crc >> 1)
    return crc & 0xFFFF
```

**实现方式**：标准 CRC16/MODBUS 逐位算法。初值 0xFFFF，多项式 0xA001（0x8005 反转）。返回 16bit 整型，低字节在线上先发。

### modbus_request_pdu

```python
def modbus_request_pdu(fc, addr, values=None, count=1):
    fn = int(str(fc), 16)
    vals = [int(v) & 0xFFFF for v in (values or [])]
    if str(fc) in ("01","02","03","04"):
        return bytes([fn]) + struct.pack(">HH", int(addr), int(count))
    if str(fc) in ("05","06"):
        v = vals[0] if vals else 0
        return bytes([fn]) + struct.pack(">HH", int(addr), v)
    if str(fc) in ("0F","10"):
        vals = vals or [0]
        pdu = bytes([fn]) + struct.pack(">HHB", int(addr), len(vals), len(vals)*2)
        for v in vals:
            pdu += struct.pack(">H", v)
        return pdu
    raise ValueError(f"不支持的功能码 {fc}")
```

**实现方式**：按功能码分三类组包，全部用 `struct.pack(">HH")`（大端 16bit）打包地址/数量/值。写多寄存器 `10` 的 PDU 格式是 `功能码 + 起始地址 + 寄存器数 + 字节数 + 数据`，字节数 = 寄存器数×2。

### modbus_frame

```python
def modbus_frame(mode, dev, pdu, tid=0):
    dev = int(dev) & 0xFF
    if str(mode).upper() == "RTU":
        raw = bytes([dev]) + pdu
        crc = modbus_crc16(raw)
        raw += bytes([crc & 0xFF, crc >> 8])       # CRC 低字节先发
    else:
        raw = struct.pack(">HHH", int(tid) & 0xFFFF, 0, len(pdu) + 1) + bytes([dev]) + pdu
    return " ".join(f"{b:02X}" for b in raw)
```

**实现方式**：RTU = 从站地址 + PDU + CRC（低字节在前）；TCP = MBAP 头（事务号 + 协议 0 + 长度 + 从站）+ PDU，无 CRC。

### modbus_field_value（兼容整块回包）

```python
def modbus_field_value(regs, addr, count=1):
    regs = list(regs or [])
    if not regs: return None
    if len(regs) <= count: return regs[0]
    if int(addr) >= 24000:
        off = (int(addr) - 24000) % 16
        if off < len(regs): return regs[off]
    return regs[0]
```

**作用**：解决「飞奕模拟器请求 1 个寄存器却整块回 16 个」的情况。当 `len(regs) > count` 且地址 ≥ 24000 时，按 `(addr-24000)%16` 算目标字段在整块里的偏移量取对应项。

### poll_readback（轮询回读）

```python
def poll_readback(read_fn, target, timeout=3.0, interval=0.2):
    deadline = time.time() + max(timeout, 0.05)
    last = []
    while time.time() < deadline:
        vals = read_fn()
        if vals:
            last = vals
            if vals[0] == target:
                return last, True
        time.sleep(max(interval, 0.01))
    return last, False
```

**作用**：写后设备动作有延时，瞬时回读会误判。此函数反复调 `read_fn()` 直到读到 `target` 或超时，返回 `(最后读到的值, 是否匹配)`。

## 6.4 MQTT sn 回环匹配

```python
def mqtt_sn_equal(a, b):
    if a is None or b is None: return False
    try:
        return int(a) == int(b)      # 容忍 int/float/str 类型差异
    except (ValueError, TypeError):
        return a == b
```

**实现方式**：设备通常按数值回显 sn，但部分实现回成字符串（`"123"`）。统一 `int(a)==int(b)` 按数值比较，转换失败回退 `a==b` 原样比较。

```python
def mqtt_error_code(msg):
    if not msg or not isinstance(msg.get("body"), dict): return None
    return msg["body"].get("errorCode")

def _in_unit_messages(msg):
    body = msg.get("body") if isinstance(msg.get("body"), dict) else {}
    items = body.get("inUnitMessages")
    return items if isinstance(items, list) else []

def mqtt_find_unit(msg, addr):
    for it in _in_unit_messages(msg):
        if it.get("a") == addr: return it
    return None

def mqtt_match_unit(msg, addr, fields):
    item = mqtt_find_unit(msg, addr)
    if item is None: return False, None
    for k, v in fields.items():
        if k not in item or not (item[k] == v):
            return False, item
    return True, item
```

`mqtt_match_unit` 检查目标地址项里**所有**期望字段是否全等（`item[k] == v`），返回 `(是否匹配, 找到的项)`。

## 6.5 judge_modbus（Modbus 判定）

`raw` 结构：`read → {"regs":[int]}`、`write → {"written","readback"}`、`group_ctrl → {"written","samples"}`。

```python
def judge_modbus(case, raw):
    kind = case.get("kind", "read")
    expect = case.get("expect") or {}
    mode = expect.get("mode", "any")

    # 1) manual 优先：无论读到什么，记录实测标 MANUAL
    if mode == "manual":
        ...  # 按 kind 取对应实测值，_verdict_manual

    # 2) read 分支
    if kind == "read":
        regs = raw.get("regs") or []
        if not regs: return _verdict(False, None, "无数据")
        actual = regs[0]
        if mode == "eq":    ok = actual == expect.get("value")
        if mode == "range": ok = expect["min"] <= actual <= expect["max"]
        return ...   # any: 只验通讯成功

    # 3) write 分支：回读比对，target 优先取模板期望值（设备可能钳制），未填则比对写入值
    if kind == "write":
        written = raw.get("written"); readback = raw.get("readback") or []
        target = expect.get("value"); target = written if target is None else target
        ok = actual == target

    # 4) group_ctrl 分支：抽样 sample_n 台全匹配
    if kind == "group_ctrl":
        need = expect.get("sample_n", 3)
        if len(samples) < need: return FAIL("抽样回读不足")
        ok = all(s == written for s in samples)
```

`_verdict` / `_verdict_manual` 返回统一结构：

```python
def _verdict(pass_, actual, reason):
    return {"verdict": "PASS" if pass_ else "FAIL", "actual": actual, "reason": reason}

def _verdict_manual(actual, reason):
    return {"verdict": "MANUAL", "actual": actual, "reason": reason}
```

## 6.6 judge_mqtt（MQTT 判定）—— 模式优先级是关键

`raw` 结构：`{"resp": 应答dict|None, "statuses":[状态报文], "readback": 回读应答|None}`。

```python
def judge_mqtt(case, raw):
    expect = case.get("expect") or {}
    mode = expect.get("mode", "any")

    # ① lock_readback：lock_write 只发不收，直接比对 lock_status_read 回显
    if mode == "lock_readback":
        return _judge_mqtt_fields(case, raw)

    # ② manual：发码+记录实测，标 MANUAL（只发不收命令也适用）
    if mode == "manual":
        return _verdict_manual(raw.get("resp"), "无自动判据，待人工确认")

    resp = raw.get("resp")

    # ③ expect_error：期望非法值被拒（errorCode != 0 = PASS）
    if mode == "expect_error":
        if resp is None: return FAIL("超时")
        ec = mqtt_error_code(resp)
        if ec is not None and ec != 0: return PASS(f"正确拒绝，错误码 {ec}")
        return FAIL("未拒绝")

    # ④ 通用：无应答 / 错误码检查
    if resp is None: return FAIL("超时（未收到应答）")
    ec = mqtt_error_code(resp)
    if ec is not None and ec != 0: return FAIL(f"应答错误码 {ec}")

    # ⑤ 其余模式
    if mode == "write_readback": return _judge_write_readback(case, raw)
    if mode == "sample":         return _judge_mqtt_sample(case, raw)
    if mode != "fields":         return PASS   # any：应答存在即 PASS
    return _judge_mqtt_fields(case, raw)
```

**关键设计**：`lock_readback` 和 `manual` 必须在「取 resp / 判超时」**之前**处理——因为这两类命令本身可能没有应答，若先判 `resp is None` 会误判超时。`expect_error` 也要先于通用错误码检查，因为此时期望的就是非 0 错误码。

### 各辅助判定实现

```python
def _judge_write_readback(case, raw):
    spec = (case.get("expect") or {}).get("fields") or {}
    readback = raw.get("readback")
    if readback is None: return FAIL("回读超时")
    field = spec.get("field")
    want = spec.get("expect")
    if want is None and field:
        body = case.get("body")
        if isinstance(body, dict): want = body.get(field)   # 期望值缺省取写 body 同名字段
    rbody = readback.get("body") if isinstance(readback.get("body"), dict) else {}
    if field == "items":   # field=="items" 表示回读 items 数组非空
        items = rbody.get("items")
        return PASS if isinstance(items, list) and items else FAIL("回读 items 为空")
    if field and field in rbody and rbody[field] == want:
        return PASS
    return FAIL(f"回读不一致: 期望{field}={want}")
```

```python
def _judge_mqtt_fields(case, raw):
    addrs = case.get("addrs") or []
    fields = expect.get("fields") or {}
    statuses = raw.get("statuses") or []
    if addrs:
        missing = [a for a in addrs
                   if not any(mqtt_match_unit(s, a, fields)[0] for s in statuses)]
        return FAIL(f"地址 {missing} 未达到期望") if missing else PASS
    else:   # addrs 空 = 全部空调：任一台匹配即生效
        for s in statuses:
            for it in _in_unit_messages(s):
                if mqtt_match_unit(s, it.get("a"), fields)[0]:
                    return PASS
        return FAIL("未找到期望字段")
```

```python
def _judge_mqtt_sample(case, raw):
    need = expect.get("sample_n", 3)
    addrs = set(case.get("addrs") or [])
    matched = set()
    for s in raw.get("statuses") or []:
        for it in _in_unit_messages(s):
            a = it.get("a")
            if addrs and a not in addrs: continue
            if mqtt_match_unit(s, a, fields)[0]: matched.add(a)
    if len(matched) < need: return FAIL(f"群控抽样不足: 需{need}台 实得{len(matched)}台")
    return PASS(sorted(matched))
```

**抽样判定核心**：统计「期望字段全匹配的**不同地址数**」≥ `sample_n` 即 PASS。多台/全量控制无需逐台全验，抽样确认即可。

## 6.7 机器可读模板解析

```python
def parse_case_rows(headers, rows) -> (cases, notes)
def parse_template_file(path) -> (cases, notes)   # 自动挑含"协议""命令cmd"的 Sheet
```

表头 11 列：`用例编号|协议|分组|用例名称|命令cmd|目标地址addrs|下发参数body|校验方式|期望字段|超时ms|启用`。

判定模式关键字集合：

```python
_CHECK_MODES = {"应答码","应答","any",""}
_RANGE_MODE  = {"字段范围","范围"}
_FIELDS_MODE = {"回读比对","回读","字段比对","字段"}
_SAMPLE_MODE = {"群控抽样","抽样","sample"}
_LOCK_MODE   = {"锁定回读","锁定","lock_readback","lock"}
_WRITE_READBACK_MODE = {"写后回读","write_readback","回读确认"}
_ERROR_MODE  = {"错误码","拒绝","expect_error","reject"}
_MANUAL_MODE = {"待确认","人工","manual","待定"}
```

### _parse_expect（校验方式 → mode/fields/lo/hi）

```python
def _parse_expect(check, expected):
    c = (check or "").strip()
    if c in _MANUAL_MODE:        return "manual", None, None, None
    if c in _ERROR_MODE:         return "expect_error", None, None, None
    if c in _WRITE_READBACK_MODE: return "write_readback", _parse_fields(expected), None, None
    if c in _LOCK_MODE:          return "lock_readback", _parse_fields(expected), None, None
    if c in _SAMPLE_MODE:        return "sample", _parse_fields(expected), None, None
    if c in _RANGE_MODE:         # 解析 "16~31" 范围
        m = re.match(r"\s*(\d+)\s*[~-]\s*(\d+)\s*", expected or "")
        ...
    if c in _FIELDS_MODE or (c and c not in _CHECK_MODES):
        fields = _parse_fields(expected)
        return "fields", fields, None, None
    return "any", None, None, None
```

### _parse_fields（期望字段 → dict）

```python
def _parse_fields(expected):
    s = expected.strip()
    try:
        v = json.loads(s)
        if isinstance(v, dict): return v
        if isinstance(v, (int, float)): return {"value": v}
    except (ValueError, json.JSONDecodeError): pass
    m = re.match(r"\s*(\d+)\s*[~-]\s*(\d+)\s*", s)   # "16~31" 范围
    if m: return {"min": int(m.group(1)), "max": int(m.group(2))}
    try: return {"value": int(s)}     # 单值
    except ValueError: return None
```

**实现方式**：优先按 JSON 解析（支持 `{"o":1,"ts":26}` 或 `26`），失败再匹配范围正则，最后按单值整数解析。

## 6.8 用例 ↔ 任务/模板互转

```python
def modbus_case_to_task(case, tid) -> dict      # 验收用例→Modbus 轮询任务
def mqtt_case_to_template(case) -> (name, json) # 验收用例→MQTT 模板({"cmd":..,"body":..})
```

保证「任务列表」与「验收用例」来自同一份模板，编号/名称一致。

## 延伸阅读

- 常量来源 → [[5-Swj-点位库与协议常量points_lib]]
- 报告怎么呈现判定结果 → [[7-Swj-验收报告导出accept_report]]
- 执行流程怎么调 judge → [[9-Swj-验收主页面与并发执行]]
