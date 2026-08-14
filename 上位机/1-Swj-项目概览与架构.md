# 1. 项目概览与架构

> 定位：给**技术支持工程师**现场对接用的**多协议并发测试上位机**，一个窗口里同时模拟/测试 DL/T645 电表、Modbus 空调控制器、MQTT+JSON 网关，并内置「验收测试」把用例跑完自动出报告。

## 1.1 技术栈总览

| 层 | 技术 | 说明 |
|----|------|------|
| GUI | Tkinter + ttk | Python 标准库自带，`clam` 主题 |
| 串口 | pyserial | DL/T645 电表模拟从站 |
| Modbus | pymodbus | 空调控制器 RTU/TCP 轮询 |
| MQTT | paho-mqtt | 网关 JSON 对上协议 |
| 报告 | openpyxl + python-docx | Excel 明细 / Word 结论 |
| 测试 | pytest | `tests/` 目录 170+ 用例 |

依赖详情见 [[2-Swj-第三方库依赖]]。

## 1.2 双模式架构（2026-08 重构，重要）

`main.py` 的 `App(tk.Tk)` 用 **F12 或状态栏按钮** 在两种模式间切换：

- **验收模式（默认）**：单页上下布局 —— 精简状态栏 → 设备连接区(`conn_embed`) → 验收模板栏 → 用例列表(Modbus 左 / MQTT 右) → 执行/进度 → 实时判定日志 → 报告导出
- **调试模式**：全局快捷栏 + 三 tab（电表模拟 / Modbus / MQTT）

关键点：切换只改 `pack` 布局，**连接状态/用例数据不重建不丢失**。`_build_mode_container` 先建 debug_frame（含 tabmb/tabmq），再建 accept_frame——因为验收页的连接配置依赖 tabmb/tabmq 已存在。

## 1.3 文件结构清单

| 文件 | 职责 |
|------|------|
| `main.py` | 主窗口、双模式框架、`_ticker()` 500ms 主循环 |
| `common.py` | 配色/字体/组件/Task/LogPanel |
| `persist.py` | config.json 读写 |
| `points_lib.py` | 飞奕点位库 + MQTT 命令常量 |
| `accept_core.py` | 验收判定引擎（纯逻辑） |
| `accept_report.py` | 报告导出三件套 |
| `conn_embed.py` | 验收模式内嵌连接配置组件 |
| `tab_accept.py` | 验收测试主页面 |
| `tab_meter.py` | 电表模拟模块（原 tab_dlt645 扩展） |
| `tab_modbus.py` | Modbus 轮询 tab |
| `tab_mqtt.py` | MQTT JSON tab |
| `meter_protocols/` | 电表协议插件包（`base.py` + 注册表） |
| `tools/` | 用例转换脚本 |
| `tests/` | pytest 单测 |

## 1.4 标准接口（三个 tab 供 main.py 调度）

每个 tab 都是 `ttk.Frame` 子类，构造签名 `(parent, app)`，实现：

```python
running()      # 是否有任务在跑
progress()     # 进度 0~100
on_tick()      # 主线程每次 tick 调用的消费入口
clear_log()    # 清日志
get_log_text() # 取日志文本
log_title()    # 日志标题
save_cfg()     # 关闭时持久化
```

## 1.5 配色规范（PRD §8 强制统一）

定义在 [[3-Swj-公共基础设施common]]：

| 语义 | 颜色 | 常量 |
|------|------|------|
| 发/上行/发布 | 蓝 `#1890FF` | `C_SEND` |
| 收/下行/应答 | 绿 `#52C41A` | `C_RECV` |
| 异常/报错 | 红 `#F5222D` | `C_ERROR` |
| 禁用/闲置 | 灰 `#86909C` | `C_DISABLE` |
| 暂停 | 橙 `#FAAD14` | `C_PAUSE` |

主题 `clam`，字体 `Microsoft YaHei UI`，等宽 `Consolas`。

## 1.6 线程模型（别破坏）

- 网络/串口回调跑在**工作线程**（paho / pymodbus / pyserial），**绝不直接碰 tk 控件**
- 事件统一放进 `self.queue`，由主线程 `on_tick()` → `_drain_queue()` 消费后才动 UI
- 轮询线程只读任务 dict 快照（dict 读原子）、只入队结果；发布/收发可直接在线程调用（线程安全）
- `main.App._ticker()` 每 500ms 调各 tab 的 `on_tick()`

## 1.7 持久化约定

`persist.save_section("<proto>", {...})`；只存纯数据（不存 tk 变量/控件）；读取方 `.get` 给默认值容忍缺字段。关闭时 `main._on_close` 调各 tab `save_cfg()`。详见 [[4-Swj-配置持久化persist]]。

## 延伸阅读

- 判定引擎怎么工作 → [[6-Swj-验收判定引擎accept_core]]
- 三个协议 tab 各自实现 → [[8-Swj-三协议模块]]
- 验收页执行流程 → [[9-Swj-验收主页面与并发执行]]
