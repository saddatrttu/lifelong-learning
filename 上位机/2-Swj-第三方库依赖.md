# 2. 第三方库依赖

> 项目用到的全部库：5 个第三方 + 若干标准库。`requirements.txt` 已声明，未安装关键库时程序会自动降级为演示模拟模式。

## 2.1 requirements.txt 全貌

```
# 真实串口通讯（DL/T645 模块）
pyserial>=3.5

# MQTT 真实通讯（tab_mqtt）
paho-mqtt>=1.6

# Modbus 真实通讯（tab_modbus）
pymodbus>=3.0

# 验收测试报告导出（tab_accept / accept_report）
openpyxl>=3.1
python-docx>=1.0
```

## 2.2 pyserial（串口）

**用途**：DL/T645 电表模拟从站，后台线程监听串口，收到主站读请求按表号应答。

```python
import serial
from serial.tools import list_ports
```

关键 API：
- `serial.Serial(port, baud, bytesize, parity, stopbits, timeout)` 打开串口
- `ser.read(n)` / `ser.write(bytes)` 收发
- `list_ports.comports()` 枚举系统串口

**降级模式**：`tab_meter.py` 顶部 `try: import serial ... HAS_SERIAL = True except: HAS_SERIAL = False`，未安装时自动回退演示模拟。

## 2.3 paho-mqtt（MQTT 客户端）

**用途**：网关 JSON 对上协议的收发，订阅上行主题、发布下行主题。

```python
import paho.mqtt.client as mqtt
```

关键 API：
- `mqtt.Client(client_id=...)` 创建客户端
- `client.username_pw_set(user, pwd)` 设置认证
- `client.on_connect = fn` / `client.on_message = fn` / `client.on_disconnect = fn` 回调
- `client.connect(host, port, keepalive)` / `client.subscribe(topic, qos)` / `client.publish(topic, payload, qos)` / `client.loop_start()`

**重要坑**：paho 内部 `if self.on_message:` 判断回调是否为真——若误把 `on_message` 设成 `None`，所有消息会被**静默丢弃**（本项目曾踩过这个坑，见 [[9-Swj-验收主页面与并发执行]]）。

## 2.4 pymodbus（Modbus）

**用途**：Modbus RTU/TCP 读写寄存器（空调控制器轮询）。

```python
from pymodbus.client import ModbusSerialClient, ModbusTcpClient
```

关键 API：
- `ModbusSerialClient(method="rtu", port=..., baudrate=..., parity=..., stopbits=..., timeout=...)`
- `ModbusTcpClient(host, port)`
- `client.read_holding_registers(address, count, slave)` → `response.registers`
- `client.write_register(address, value, slave)` / `client.write_registers(address, values, slave)`
- `client.connect()` / `client.close()`

## 2.5 openpyxl（Excel 读写）

**用途**：验收报告 Excel 明细、机器可读模板解析、用例转换脚本。

```python
import openpyxl
```

关键 API：
- `openpyxl.Workbook()` 新建 / `openpyxl.load_workbook(path, data_only=True)` 读取
- `ws.cell(row, col, value)` 写单元格
- `ws.iter_rows(values_only=True)` 遍历
- `openpyxl.styles.Font/PatternFill/Alignment` 样式
- `wb.save(path)`

## 2.6 python-docx（Word 生成）

**用途**：验收结论页（结论汇总 + 失败摘要 + 待确认清单 + 签字栏）。

```python
from docx import Document
from docx.shared import Pt
from docx.enum.table import WD_TABLE_ALIGNMENT
```

关键 API：
- `Document()` 新建文档
- `doc.add_heading(text, level)` / `doc.add_paragraph(text)`
- `doc.add_table(rows, cols)`，`table.style = "Light Grid Accent 1"`
- `doc.save(path)`

## 2.7 用到的标准库

| 模块 | 用途 |
|------|------|
| `tkinter` / `tkinter.ttk` | 全部 GUI |
| `json` | MQTT 报文序列化/解析、配置、模板 |
| `struct` | Modbus 帧字节打包（`>HH`、`>HHB`） |
| `re` | 期望值范围/地址/字段解析 |
| `threading` | 串口监听、轮询、MQTT 后台线程 |
| `queue` | 工作线程 → 主线程事件队列 |
| `time` | 计时、超时、轮询 deadline |
| `os` / `tempfile` | 路径、配置原子写入 |
| `random` | 演示模拟数据、Task 演示抖动 |

## 延伸阅读

- 公共组件封装 → [[3-Swj-公共基础设施common]]
- 三个库各自怎么接进 tab → [[8-Swj-三协议模块]]
