# 9. 验收主页面与并发执行 tab_accept.py + conn_embed.py

> 验收测试的「前台」：用例管理 + 一键执行（Modbus+MQTT 并发）+ 实时判定 + 报告导出。本笔记重点讲**执行流程与核心函数的实现**。

## 9.1 内嵌连接配置 conn_embed.py

两个类 `ConnEmbedModbus` / `ConnEmbedMqtt` 结构对称，核心是「与调试页共享 client + 双向同步」。

### 同步实现（以 Modbus 为例）

```python
def _sync_from_tab(self):
    """从 TabModbus 同步当前配置（无真实接口时回退默认值，兼容测试替身）"""
    try:
        cfg = self.tabmb._conn_params()
    except Exception:
        cfg = None
    if not cfg:
        cfg = {"mode": "RTU", "port": "COM1", "baud": 9600, "dev": 1, ...}
    self._cfg_from(cfg); self._refresh_status()

def _sync_to_tab(self):
    """把当前配置写回 TabModbus（无真实接口时跳过）"""
    if not (hasattr(self.tabmb, "mode") and hasattr(self.tabmb, "_switch_mode")): return
    self.tabmb.mode.set(self.mode.get())
    self.tabmb._switch_mode()
    if self.mode.get() == "RTU":
        self.tabmb.rtu_cfg["COM"].set(self.cmb_com.get())
        self.tabmb.rtu_cfg["波特率"].set(self.cmb_baud.get())
        ...
```

**关键设计**：`_toggle_conn` 连接时先 `_sync_to_tab()` 把验收页填的参数写回调试页控件，再调 `tabmb._do_connect()`——**连接动作永远由调试页的 client 完成，验收页不自己建连接**，保证两个页面共享同一个 client。

```python
def _toggle_conn(self):
    if not hasattr(self.tabmb, "_do_connect"): return
    if self.tabmb.connected: self.tabmb._do_disconnect()
    else:
        self._sync_to_tab()
        self.tabmb._do_connect()
    self._refresh_status()
```

## 9.2 一键启动 _start_all / _trigger

```python
def _start_all(self):
    if self._running_protos: return   # 已有执行中
    started = False
    for proto in ("modbus", "mqtt"):
        cases = [c for c in self.cases if c.get("proto") == proto and c.get("enabled", True)]
        if not cases: continue
        tab = self.app.tabmb if proto == "modbus" else self.app.tabmq
        if not (tab.connected and getattr(tab, "client", None) is not None):
            self.logp.log(f"{proto} 未连接，跳过验收", "warn"); continue
        self._trigger(proto, cases); started = True
```

```python
def _trigger(self, proto, cases):
    self._stop_evt.clear()
    self._running_protos.add(proto)
    self._lock_ui(True)
    for c in cases: c["state"] = "stop"
    if proto == "mqtt": self._install_mqtt_hook()   # 挂接回显钩子
    t = threading.Thread(target=self._run_group, args=(proto, cases, proto), daemon=True)
    self._threads.append(t); t.start()
```

**实现要点**：每个协议一组后台线程，`_running_protos` 集合记录哪些协议在跑，`_stop_evt` 是共享停止事件。MQTT 组启动前先 `_install_mqtt_hook` 拦截应答。

## 9.3 _run_group（组内顺序执行）

```python
def _run_group(self, proto, cases, _key):
    try:
        if proto == "modbus":
            self._precheck_modbus()
            for c in cases:
                if self._stop_evt.is_set(): break
                c["state"] = "running"
                res = self._exec_modbus(c)
                self._finalize(c, res)
        else:
            self._precheck_mqtt()
            for c in cases:
                if self._stop_evt.is_set(): break
                c["state"] = "running"
                res = self._exec_mqtt(c)
                self._finalize(c, res)
    except Exception as ex:
        self._queue.put(("info", f"{proto} 组执行异常: {ex}", "fail"))
    finally:
        self._queue.put(("done", proto, None))
```

```python
def _finalize(self, case, res):
    res["case"] = case
    self._queue.put(("result", res))
    case["state"] = res.get("verdict", "FAIL").lower()
    self._queue.put(("refresh", None, None))
```

**线程模型**：执行跑在工作线程，结果通过 `self._queue.put(("result", res))` 入队，由主线程 `on_tick` 消费后更新 UI 和结果列表。

## 9.4 _exec_modbus（单条 Modbus 用例执行）

按 `kind` 三分支：

### read 分支

```python
r = client.read_holding_registers(addr, count=cnt, device_id=dev)
regs = list(r.registers) if not r.isError() else []
if cnt == 1 and regs:
    res = judge_modbus(c, {"regs": [modbus_field_value(regs, addr, 1)]})
else:
    res = judge_modbus(c, {"regs": regs})
tx = modbus_frame(mode, dev, modbus_request_pdu("03", addr, None, cnt), ...)
rx = self._resp_frame(mode, dev, r) or "(无应答)"
```

**要点**：单点读时用 `modbus_field_value` 从整块回包里对齐取目标字段；同时构造原始报文 `raw_tx/raw_rx` 供报文附录。

### write 分支（写 + 轮询回读）

```python
val = int(c.get("val", 0))
expect = c.get("expect") or {}
target = expect.get("value"); target = val if target is None else int(target)
w = client.write_register(addr, val & 0xFFFF, device_id=dev)

def _rd():
    r = client.read_holding_registers(addr, count=1, device_id=dev)
    cap["r"] = r
    if r.isError(): return []
    return [modbus_field_value(r.registers, addr, 1)]

if expect.get("mode") == "manual":
    readback = _rd()                                   # 只读一次记录实测
    res = judge_modbus(c, {"written": val, "readback": readback})
else:
    readback, matched = poll_readback(_rd, target, timeout)   # 轮询回读直到匹配
    if matched: res = judge_modbus(c, {"written": val, "readback": readback})
    else: res = {"verdict": "FAIL", "reason": f"回读未达期望({target})，轮询超时"}
```

**要点**：`target` 优先取模板期望值（设备可能钳制/换算，如写 32 回读 31），未填才比对写入值。写后用 `poll_readback` 反复回读，解决设备动作延时误判。

### group_ctrl 分支（写 + 抽样回读）

```python
v = int(c.get("val", 0)); vals = [v & 0xFFFF]
w = client.write_registers(addr, vals, device_id=dev)   # 群控写单个字段
n = int(self.sample_n.get() or 3)
field = (c.get("expect") or {}).get("field", c.get("field", "o"))
ch = c.get("channel", 1)
for i in range(1, n + 1):
    a = iu_point(ch, i, field)["addr"]     # 抽样第 i 台内机的目标字段地址
    sv, _m = poll_readback(_rd, target, timeout)
    samples.append(sv[0] if sv else None)
res = judge_modbus(c, {"written": target, "samples": samples})
```

**要点**：群控只写 1 个寄存器（开关或温度），然后用 `iu_point` 算出通道内前 n 台内机的同字段地址，逐台回读抽样。

## 9.5 _exec_mqtt（单条 MQTT 用例执行）—— 最复杂的函数

### 构造并发布请求

```python
self._sn += 1; sn = self._sn
req = {"cmd": c.get("cmd"), "sn": sn}
if uuid: req["uuid"] = uuid
body = c.get("body")
if body is None and c.get("cmd") in _DEFAULT_READ_BODY:
    body = _DEFAULT_READ_BODY[c.get("cmd")]     # 读命令缺省 body（如 general_read 需 cmd:all）
if body is not None: req["body"] = body
tx = json.dumps(req, ensure_ascii=False)
client.publish(topic_pub, tx, qos=int(tabmq.qos.get()))
```

### 按 expect.mode 分支

**① lock_readback（只发不收）**：

```python
if mode == "lock_readback":
    raw_msgs.append({"dir": "收", "msg": "(lock_write 只发不收，无应答)"})
    vreq = self._send_lock_status_read(client, topic_pub, uuid, c)  # 补发 lock_status_read
    statuses = self._wait_statuses(c, timeout)
    res = judge_mqtt(c, {"resp": None, "statuses": statuses})
```

**② manual（短等 1s）**：`wait = 1.0 if mode == "manual" else timeout`，只发不收命令不等满超时。

**③ write_readback（写后回读）**：

```python
if resp is not None and mode == "write_readback":
    spec = expect.get("fields") or {}
    read_cmd = spec.get("read")           # 回读命令（如 device_update_config_read）
    if read_cmd:
        self._sn += 1
        rreq = {"cmd": read_cmd, "sn": self._sn}
        rbody = spec.get("read_body")
        if isinstance(rbody, dict): rreq["body"] = rbody
        client.publish(topic_pub, json.dumps(rreq, ...), qos=0)
        readback = self._wait_sn(self._sn, timeout)
    res = judge_mqtt(c, {"resp": resp, "readback": readback, "statuses": []})
```

**④ fields / sample（控制生效回显）**：

```python
if resp is not None and mode in ("fields", "sample"):
    vreq = self._send_status_read(client, topic_pub, uuid)   # 补发 status_read
    statuses = self._wait_statuses(c, timeout)
res = judge_mqtt(c, {"resp": resp, "statuses": statuses})
```

### 补发验证命令的实现

```python
def _send_status_read(self, client, topic, uuid):
    self._sn += 1
    req = {"cmd": "status_read", "sn": self._sn, "body": {"cmd": "all"}}
    if uuid: req["uuid"] = uuid
    client.publish(topic, json.dumps(req, ensure_ascii=False), qos=0)
    return req

def _send_lock_status_read(self, client, topic, uuid, c):
    self._sn += 1
    body = {"cmd": "all"}
    addrs = c.get("addrs") or []
    if addrs: body = {"cmd": "addrs", "addrs": addrs}
    req = {"cmd": "lock_status_read", "sn": self._sn, "body": body}
    ...
```

## 9.6 等待机制 _wait_sn / _wait_statuses

```python
def _wait_sn(self, sn, timeout):
    deadline = time.time() + timeout
    while time.time() < deadline and not self._stop_evt.is_set():
        for m in self._mqtt_msgs:
            if mqtt_sn_equal(m.get("sn"), sn): return m
        time.sleep(0.03)
    return None
```

**实现方式**：`_mqtt_msgs` 是 `_mqtt_hook` 拦截到的所有应答列表，`_wait_sn` 轮询（30ms 粒度）找顶层 sn 匹配的报文。用 `mqtt_sn_equal` 容忍 sn 字符串/数值差异。

```python
def _wait_statuses(self, c, timeout):
    out = []; deadline = time.time() + timeout
    seen = set(); addrs = set(c.get("addrs") or [])
    fields = (c.get("expect") or {}).get("fields") or {}
    mode = (c.get("expect") or {}).get("mode")
    sample_n = (c.get("expect") or {}).get("sample_n", 3)
    matched_addrs = set()
    while time.time() < deadline and not self._stop_evt.is_set():
        for m in list(self._mqtt_msgs):
            key = id(m)
            if key in seen: continue          # 用 id() 去重，避免重复处理同一条
            seen.add(key)
            if not _msg_has_units(m): continue
            for it in _units(m):
                a = it.get("a")
                if addrs and a not in addrs: continue
                if all(it.get(k) == v for k, v in fields.items()):
                    msg_matched.add(a)
            if msg_matched:
                out.append(m); matched_addrs |= msg_matched
            # 提前返回条件
            if mode == "sample":
                if len(matched_addrs) >= sample_n: return out
            elif addrs:
                if matched_addrs >= addrs: return out
            elif matched_addrs:               # 空 addrs（全部）任一台匹配即返回
                return out
        time.sleep(0.03)
    return out
```

**实现方式**：收集「含 inUnitMessages 且目标地址字段全匹配」的状态报文。用 `id(m)` 去重；`matched_addrs` 记录已匹配的不同地址。三种提前返回条件：sample 达到 sample_n 台 / fields 达到全部目标地址 / 空 addrs 时第一台匹配即返回。

## 9.7 MQTT 回显钩子（曾踩的大坑）

```python
def _install_mqtt_hook(self):
    tabmq = self.app.tabmq
    c = tabmq.client
    self._mqtt_msgs = []; self._mqtt_raw = {}
    if c is not None and c.on_message is not self._mqtt_hook:
        self._saved_on_message = c.on_message
        c.on_message = self._mqtt_hook
```

```python
def _mqtt_hook(self, client, userdata, msg):
    raw = msg.payload.decode("utf-8", "replace")
    try: payload = json.loads(raw)
    except (ValueError, json.JSONDecodeError): payload = raw
    self._mqtt_msgs.append(payload)          # 供 _wait_sn/_wait_statuses 匹配
    self._mqtt_raw[id(payload)] = raw        # 保存原始文本供报文附录
    tabmq = self.app.tabmq
    tabmq.queue.put(("msg", (msg.topic, raw, msg.qos)))   # 镜像到订阅日志
```

```python
def _restore_mqtt_hook(self):
    c = tabmq.client
    if c is not None and hasattr(self, "_saved_on_message"):
        c.on_message = self._saved_on_message
```

**关键 bug 教训**：`__init__` 里若写 `self._mqtt_hook = None`，会**遮蔽同名类方法** `_mqtt_hook`，导致 `_install_mqtt_hook` 把 `c.on_message` 设成 `None`。paho 内部 `if self.on_message:` 判断为假 → 所有消息**静默丢弃**（表现为「订阅只收到 online，收不到验收应答」）。正确做法：保存字段改名 `self._saved_on_message`，别与方法同名。

## 9.8 前置自检

```python
def _precheck_modbus(self):
    client = tabmb.client; dev = tabmb._conn_params()["dev"]
    d = client.read_holding_registers(0, count=6, device_id=dev)   # 读设备 ID/固件
    regs = list(d.registers) if not d.isError() else []
    if regs: self._dev_info["device"] = f"ID={regs[0]} FW={regs[1]}"

def _precheck_mqtt(self):
    self._sn += 1
    req = {"cmd": "general_read", "sn": self._sn, "body": {"cmd": "all"}}
    client.publish(topic, json.dumps(req, ...), qos=0)
    resp = self._wait_sn(self._sn, 2.0)
    if resp and isinstance(resp.get("body"), dict):
        b = resp["body"]
        cid = b.get("concentrator_id") or b.get("device")
        self._dev_info["device"] = f"CID={cid} ver={b.get('ver','--')}"
```

**作用**：跑用例前先读设备信息（ID/固件/集中器ID），填入报告抬头的「被测设备」字段。

## 9.9 on_tick（主线程消费）

```python
def on_tick(self):
    self.conn_mb.refresh(); self.conn_mq.refresh()
    if not self._running_protos: return
    while True:
        item = self._queue.get_nowait()
        kind = item[0]
        if kind == "result": self._results.append(item[1]); self._log_result(item[1])
        elif kind == "info": self.logp.log(msg, tag)
        elif kind == "refresh": self.reload_trees()
        elif kind == "done": self._running_protos.discard(proto)
    if not self._running_protos:       # 全部协议完成 → 收尾
        self._finish(); self._threads = []
    self._update_pb(...)               # 更新进度条
```

```python
def _finish(self):
    self._running_protos.clear()
    self._restore_mqtt_hook()          # 恢复原 on_message
    self._lock_ui(False)
    self._collect_info()
```

## 延伸阅读

- 判定引擎细节 → [[6-Swj-验收判定引擎accept_core]]
- 报告导出 → [[7-Swj-验收报告导出accept_report]]
- 双模式主框架 → [[1-Swj-项目概览与架构]]
