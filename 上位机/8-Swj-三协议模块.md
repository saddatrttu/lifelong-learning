# 8. 三协议模块（电表模拟 / Modbus / MQTT）

> 调试模式的三个 tab，各自是 `ttk.Frame` 子类，实现统一接口（见 [[1-Swj-项目概览与架构]]）。本笔记重点讲各模块**核心函数的实现方式**。

## 8.1 电表模拟 tab_meter.py

### 8.1.1 DL/T645 帧编解码函数

**地址 BCD 转换**：

```python
def _addr_to_bcd(addr_str):
    """12位表号(十进制字符串) -> 6字节BCD，低字节在前(帧内顺序)"""
    return bytes.fromhex(addr_str)[::-1]

def _bcd_to_addr(b6):
    """帧内6字节地址 -> 12位表号字符串"""
    return b6[::-1].hex().upper()
```

**实现关键**：DL/T645 表号用 **BCD** 编码，且**低字节在前**。`bytes.fromhex(addr_str)` 把 12 位表号当 hex 解析成 6 字节，`[::-1]` 反转实现「低字节在前」。反向同理。

**构建读请求帧**：

```python
def build_read_frame(addr_str, di):
    frame = bytearray([0x68])
    frame += _addr_to_bcd(addr_str)
    frame += bytes([0x68, 0x11, 0x04])       # 起始符 + 控制码(读) + 数据长度
    for b in di:
        frame.append((b + 0x33) & 0xFF)      # DI 每字节 +0x33
    frame.append(sum(frame) & 0xFF)          # 校验和
    frame.append(0x16)
    return bytes(frame)
```

**+0x33 的作用**：DL/T645 对数据域每个字节加 0x33 加密，避免数据字节撞上帧定界符 `0x68`/`0x16` 造成误判。

**构建正常/异常应答帧**：

```python
def build_response_frame(addr_str, di, data_bytes):
    frame = bytearray([0x68])
    frame += _addr_to_bcd(addr_str)
    frame += bytes([0x68, 0x91])             # 控制码 0x91 = 读数据正常应答
    payload = di + data_bytes
    frame.append(len(payload))
    for b in payload:
        frame.append((b + 0x33) & 0xFF)
    frame.append(sum(frame) & 0xFF)
    frame.append(0x16)
    return bytes(frame)

def build_error_frame(addr_str, err=0x01):
    frame = bytearray([0x68])
    frame += _addr_to_bcd(addr_str)
    frame += bytes([0x68, 0xD1, 0x01])       # 控制码 0xD1 = 异常应答
    frame.append((err + 0x33) & 0xFF)
    frame.append(sum(frame) & 0xFF)
    frame.append(0x16)
    return bytes(frame)
```

**解析帧 parse_frame**：

```python
def parse_frame(frame):
    i = frame.index(0x68)                    # 找起始符（跳过前导 FE）
    if i + 10 > len(frame) or frame[i + 7] != 0x68: return None
    addr = _bcd_to_addr(frame[i + 1:i + 7])
    ctrl = frame[i + 8]; ln = frame[i + 9]
    end = i + 10 + ln
    if end + 2 > len(frame) or frame[end + 1] != 0x16: return None
    if (sum(frame[i:end]) & 0xFF) != frame[end]: return None   # 校验和
    data = bytes(((b - 0x33) & 0xFF) for b in frame[i + 10:end])
    return {"addr": addr, "ctrl": ctrl, "len": ln, "data": data,
            "error": bool(ctrl & 0x40), "more": bool(ctrl & 0x20)}
```

**实现逻辑**：`index(0x68)` 跳过前导 `FE`；校验第二处 `0x68`（`frame[i+7]`）；按长度字节算数据域范围；校验 `0x16` 结束符和 CS 校验和；数据域每字节 `-0x33` 还原。`ctrl & 0x40` 判异常位，`ctrl & 0x20` 判后续帧位。

**BCD 编解码**：

```python
def _encode_bcd(value, nbytes=4):
    s = f"{int(round(value * 100)):0{nbytes * 2}d}"[-nbytes * 2:]
    return bytes.fromhex(s)[::-1]

def _decode_bcd(data):
    s = data[::-1].hex()
    return int(s) / 100.0 if s else 0.0
```

**实现方式**：DL/T645 电量数据是 BCD，末 2 位小数。编码时 `value*100` 转整数 → 定长十进制字符串 → 当 hex 解析 → 反转（低字节在前）。解码是逆过程，`int(s)/100.0` 恢复两位小数。

### 8.1.2 DLT645Slave 从站线程

```python
class DLT645Slave(threading.Thread):
    def __init__(self, port, baud, databits, stopbits, parity, ui_queue,
                 get_meter, find_meter):
        # get_meter: seq -> meter dict（实时取值）
        # find_meter: addr_str -> meter dict / None
```

**run 实现**：

```python
def run(self):
    try:
        self.ser = serial.Serial(port=self.port_name, baudrate=self.baud,
                                 bytesize=self.databits, parity=self.parity,
                                 stopbits=self.stopbits, timeout=0.1, write_timeout=1.0)
        self.q.put(("opened", self.port_name))
    except Exception as ex:
        self.q.put(("open_fail", str(ex))); return
    while not self._stop_evt.is_set():
        frame = self._read_request()
        if frame: self._handle(frame)
    self.ser.close(); self.q.put(("closed", self.port_name))
```

**核心 _read_request（阻塞拼帧）**：

```python
def _read_request(self):
    buf = bytearray(); deadline = time.time() + 2.0
    while time.time() < deadline and not self._stop_evt.is_set():
        b = self.ser.read(1)
        if not b: continue
        if b[0] == 0xFE: continue              # 前导字节丢弃
        if b[0] == 0x68:
            buf.append(b[0]); break
    # 读固定头到长度字节（共10字节）
    while len(buf) < 10 and time.time() < deadline:
        chunk = self.ser.read(10 - len(buf))
        if chunk: buf += chunk
    if len(buf) < 10 or buf[7] != 0x68: return bytes(buf)
    total = 10 + buf[9] + 2                    # 头 + 数据长度 + CS + 16
    while len(buf) < total and time.time() < deadline:
        chunk = self.ser.read(total - len(buf))
        if chunk: buf += chunk
    return bytes(buf)
```

**实现逻辑**：串口是字节流，需要手动拼帧。先丢弃 `FE` 前导、等到 `0x68` 起始符，再读满 10 字节固定头，从 `buf[9]` 读数据长度算总帧长，继续读满剩余字节。2 秒 deadline 防止卡死。

**核心 _handle（解析并应答）**：

```python
def _handle(self, frame):
    p = parse_frame(frame)
    if not p: self.q.put(("rx_invalid", ...)); return
    addr = p["addr"]
    m = self.find_meter(addr)
    if m is None: self.q.put(("no_meter", ...)); return   # 表号未注册，忽略
    if (p["ctrl"] & 0x1F) != 0x11:            # 非读数据命令
        resp = build_error_frame(addr, 0x01)  # 不支持的功能
        self._send(resp, ...); return
    di = p["data"][:4]
    if di == DI_ENERGY_TOTAL: value = m["energy"]
    elif di == DI_ENERGY_COMB: value = m.get("energy_comb", m["energy"])
    elif di == DI_VOLTAGE:     value = m["v"]
    else:
        resp = build_error_frame(addr, 0x02)  # 无此数据项
        self._send(...); return
    resp = build_response_frame(addr, di, _encode_bcd(value, 4))
    self._send(resp, seq, addr, True, hex_spaced(resp), rx_hex)
```

**实现逻辑**：按 DI（数据标识）匹配该电表 dict 里的电量/电压字段，用 `_encode_bcd` 打包成 4 字节数据，组正常应答。所有结果通过 `ui_queue` 入队，由主线程 `_drain_queue` 消费更新 UI。

## 8.2 Modbus 轮询 tab_modbus.py

### 8.2.1 连接管理

```python
def _conn_params(self):
    if self.mode.get() == "RTU":
        c = self.rtu_cfg
        return {"mode": "RTU", "port": c["COM"].get(), "baud": int(c["波特率"].get() or 9600),
                "bytesize": int(c["数据位"].get() or 8), "stopbits": int(c["停止位"].get() or 1),
                "parity": c["校验"].get() or "N", "dev": int(c["从站ID"].get() or 1),
                "timeout": int(c["超时(ms)"].get() or 1000) / 1000.0}
    # TCP 同理
```

**作用**：从界面控件快照出参数字典（主线程调用），供建立连接/验收执行用。

```python
def _open_client(self, p):
    if p["mode"] == "RTU":
        client = ModbusSerialClient(port=p["port"], baudrate=p["baud"],
                                    bytesize=p["bytesize"], parity=p["parity"],
                                    stopbits=p["stopbits"], timeout=p["timeout"], retries=1)
    else:
        client = ModbusTcpClient(host=p["host"], port=p["port"],
                                 timeout=p["timeout"], retries=1)
    if not client.connect():
        client.close(); raise ConnectionError("目标不可达/超时")
    return client
```

`_do_connect` 先判 `HAS_MB`（pymodbus 是否安装），未装则 `_set_connected(True, sim=True)` 回退演示；装了则 `_open_client`，成功后存 `self.client` 和 `self._conn_params_snap`（连接参数快照，供重连用）。

### 8.2.2 轮询线程 _poll_loop

```python
def _poll_loop(self, client, dev, snap):
    ids = [s["id"] for s in snap]
    limit = self._times_limit
    while not self._poll_stop.is_set():
        now = time.time(); alive = False
        for tid in ids:
            t = self._task_snap(tid)          # 从主线程任务表读调度信息（dict 读原子）
            if not t or t["state"] != "running": continue
            if limit > 0 and t.get("polls", 0) >= limit:
                t["state"] = "stopped"; continue
            alive = True
            if now < t["next"]: continue      # 未到该任务下次执行时间
            t["next"] = now + t["intv"] / 1000.0
            t0 = time.time()
            self._poll_once(client, dev, t)
            t["polls"] = t.get("polls", 0) + 1
            if time.time() - t0 > 10.0:       # 单次超 10s 判通讯中断
                self._conn_lost = True
                self.queue.put(("clost",)); break
        if limit > 0 and not alive: break     # 全部达次数 → 线程退出
        time.sleep(0.02)
```

**实现要点**：每个任务有独立的 `next`（下次执行时间戳），线程按 20ms 粒度循环检查，到点才发码，实现「按各自间隔独立调度」。只读 `t["state"]/t["next"]` 等字段（dict 读原子），结果通过 `queue.put` 入队，绝不碰 tk。

### 8.2.3 _poll_once（单次读写）

```python
def _poll_once(self, client, dev, t):
    addr = int(t["addr"]); fc = t["fc"]; cnt = t["cnt"]
    vals = parse_vals(t["val"]); val = vals[0]
    if fc == "01": r = client.read_coils(addr, count=cnt, device_id=dev); raw = r.bits if not r.isError() else None
    elif fc == "02": r = client.read_discrete_inputs(...)
    elif fc == "03": r = client.read_holding_registers(...)
    elif fc == "04": r = client.read_input_registers(...)
    elif fc == "05": r = client.write_coil(addr, bool(val), device_id=dev)
    elif fc == "06": r = client.write_register(addr, int(val) & 0xFFFF, device_id=dev)
    elif fc == "10":
        wvals = ([int(v) & 0xFFFF for v in vals] + [0] * cnt)[:cnt]
        r = client.write_registers(addr, wvals, device_id=dev)
    ...
    if raw is None: self.queue.put(("perr", t["id"], f"应答异常/超时"))
    else: self.queue.put(("pok", t["id"], rvals, ms, fc, addr))
```

**实现方式**：按功能码分发到 pymodbus 对应 API；`isError()` 判断应答异常；成功把寄存器值列表 + 耗时 + 功能码/地址入队。写多寄存器 `10` 时把 vals 补零到 cnt 长度。

### 8.2.4 自动重连 _reconnect

```python
def _reconnect(self):
    if not (self.polling and HAS_MB and self._conn_lost): return
    p = self._conn_params_snap or self._conn_params()
    try: client = self._open_client(p)
    except Exception: return                    # 本次失败，下个 tick 再试
    self.client.close(); self.client = client
    self._conn_lost = False; self.reconnects += 1
    # 重启轮询线程用新 client
    self._poll_stop.set()
    if self._poll_thread and self._poll_thread.is_alive():
        self._poll_thread.join(timeout=1.0)
    self._poll_stop.clear()
    snap = [{"id": t["id"]} for t in self.tasks]
    self._poll_thread = threading.Thread(target=self._poll_loop, args=(self.client, p["dev"], snap), daemon=True)
    self._poll_thread.start()
```

**实现方式**：`_poll_once` 超 10s 会置 `_conn_lost` 并入队 `clost`；主线程 `on_tick` 里的 `_reconnect` 检测到后尝试重连，成功则停旧轮询线程、用新 client 起新线程。

## 8.3 MQTT tab_mqtt.py

### 8.3.1 真实连接 _connect_real

```python
def _connect_real(self):
    host = self.mq["IP/域名"].get().strip()
    port = int(self.mq["端口"].get().strip())
    keepalive = int(self.mq["心跳(s)"].get().strip() or 60)
    cid = self.mq["ClientID"].get().strip() or f"client_{random.randint(1000,9999)}"
    self._render_client = cid                    # 供轮询线程取快照

    # paho 2.x 需要 CallbackAPIVersion；clean_session 仅 MQTT 3.1.1 有效
    try:
        client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2,
                             client_id=cid, clean_session=self.clean_session.get())
    except Exception:
        client = mqtt.Client(client_id=cid, clean_session=self.clean_session.get())

    user = self.mq["用户名"].get().strip()
    if user:
        client.username_pw_set(user, self.mq["密码"].get())
    will = self.mq["遗嘱消息"].get().strip()
    if will:
        client.will_set(pub0, payload=will, qos=int(self.qos.get()), retain=False)

    client.on_connect = self._on_connect
    client.on_disconnect = self._on_disconnect
    client.on_message = self._on_message

    client.connect_async(host, port, keepalive=keepalive)
    client.loop_start()                          # 后台网络线程
    self.client = client
```

**实现要点**：`connect_async` + `loop_start()` 非阻塞连接（网络循环跑后台线程），三个回调只入队不碰 UI。`CallbackAPIVersion.VERSION2` 是 paho 2.x 必需的，老版本 fallback。

### 8.3.2 paho 回调（只入队）

```python
def _on_connect(self, client, userdata, flags, rc, properties=None):
    code = rc.value if hasattr(rc, "value") else int(rc)   # 兼容 1.x/2.x 返回码
    self.queue.put(("conn", code))

def _on_disconnect(self, client, userdata, *args):
    self.queue.put(("disconn", None))

def _on_message(self, client, userdata, msg):
    self.queue.put(("msg", (msg.topic, msg.payload.decode("utf-8", "replace"), msg.qos)))
```

### 8.3.3 _drain_queue（主线程消费）

```python
def _drain_queue(self):
    while True:
        kind, payload = self.queue.get_nowait()
        if kind == "conn":
            if payload == 0:                 # 连接成功
                self._after_conn_ui(); self._subscribe_topics_real()
            else: self._log(f"连接被拒绝，返回码 {payload}", "err"); ...
        elif kind == "disconn": ...
        elif kind == "msg":                  # 订阅报文
            topic, body, q = payload
            self._log(f"[订阅] {topic} QoS{q} {body}", "recv")
        elif kind == "pub_ok": self._handle_pub(True, topic, body)
        elif kind == "pub_fail": self._handle_pub(False, topic, err)
        elif kind == "poll_done": ...        # 达到次数自动停止
```

### 8.3.4 模板变量替换 _render

```python
def _render(self, body):
    return (body.replace("${ts}", str(int(time.time())))
                .replace("${client}", self._render_client)
                .replace("${temp}", str(round(random.uniform(18, 32), 1)))
                .replace("${hum}", str(round(random.uniform(40, 70), 1)))
                .replace("${value}", str(random.randint(0, 100))))
```

**注意**：此方法在轮询线程调用，只能读启动时拍好的 `self._render_client` 快照，不能碰任何 tk 控件/变量。

### 8.3.5 模板↔验收用例同源 _sync_accept_cases

```python
def _case_from_template(self, t):
    body = t.get("body") or "{}"
    try: obj = json.loads(body)
    except: obj = {}
    cmd = t.get("cmd") or (obj.get("cmd") if isinstance(obj, dict) else None)
    if not cmd: return None
    expect = t.get("expect") or {"mode": "any"}
    return new_case(t.get("case_no") or f"MQ-{t.get('id', 0):03d}", "mqtt",
                    t.get("name", cmd), "mqtt_cmd", group="MQTT 轮询",
                    cmd=cmd, addrs=t.get("addrs") or [], body=obj,
                    expect=expect, timeout_ms=3000, enabled=True)
```

**作用**：MQTT 模板区与验收用例同源——模板 dict 里带 `cmd`/`expect`/`addrs` 时直接转成验收用例，避免两处维护。

## 延伸阅读

- 协议帧/报文细节 → [[10-Swj-协议知识]]
- 三个模块怎么被验收页复用 → [[9-Swj-验收主页面与并发执行]]
