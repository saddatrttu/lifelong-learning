# 11. 用例转换脚本 tools/

> 把人工写的「测试用例 Excel」自动转换成**机器可读验收模板**（11 列）。两个脚本：`convert_legacy_cases.py`（Modbus）、`convert_json_cases_v5.py`（MQTT JSON，最新）。

## 11.1 convert_legacy_cases.py（Modbus 用例）

### 关键词识别

```python
def find_cmd(text)        # 文本→命令/功能码
def parse_addrs(text)     # 文本→地址
def infer_group(text)     # 文本→分组
def infer_control(text)   # 文本→控制类型
def is_manual(text)       # 是否人工介入
```

### 核心转换

```python
def convert(ws_src)                       # 旧版 Sheet 转换
def convert_modbus_new(ws_src)            # 新版详细用例 Sheet3（无表头/功能码+地址解析）
def convert_modbus(ws_src)                # Modbus Sheet
```

内部用 `_is_legal(addr, val)`（合法值集）、`_split_sequence(d)`（多值拆独立用例）、`_extract_values(d)`、`_parse_d_frame(d)`、`_group_of(b)` 等把一行拆成多条用例。

关键分流（`is_lock_case` / `is_human_case` / `is_exec_case`）：

- **锁用例**：先 lock 再 control，拆成两条（锁定 + 改被锁项验证锁定有效）
- **人工**：物理介入 → 转「人工确认」
- **可执行**：拆成自动用例

## 11.2 convert_json_cases_v5.py（MQTT JSON 用例，最新）

### 关键数据结构

```python
READ_CMDS        # 读命令集合
WRITE_BODY       # 写命令默认 body
NO_RESP          # 只发不收命令 {"energy_read","node_statistics"}
WRITE_READBACK   # 写命令→回读命令映射
_TITLE_BODY      # 按标题覆盖 body
_GENERAL_WRITE   # general_write 字段关键词
_LOCK_CASES      # 锁用例拆解
_FULL_CTRL_BODY  # 全字段控制
_SAMPLE_BODY     # 多台/全量控制（群控抽样）
_HUMAN           # 物理介入关键词
_ABNORMAL        # 异常/边界关键词
_LEGAL           # 合法值集
_ILLEGAL_EXPECT  # 非法值：拒绝 vs 确认
```

### 命令识别 find_cmd

优先中文命令检测（批量锁定→`lock_write_batch`、锁定状态→`lock_status_read`、锁定→`lock_write`、校时/索引/总线→`general_write`），再英文命令词，最后字段名兜底（`schNo`/定时→`schedule_write`、`tempSetLo`/锁定上下限→`lock_write`、`electrical_type`→`meter_electrical_type_write`、`tempSet`/`workMode`/`fanSpeed`/`onOff`→`control_write`）。

### 拆解与边界展开

```python
def _extract_vals(text)              # 文本→候选值列表
def _control_field(b)                # body→控制字段
def expand_lock_case(b)              # 锁定用例：先 lock_write 再 control_write
def expand_boundary(b, d)            # 边界/异常：按值拆行（合法→回读比对；非法拒绝→错误码；非法确认→待确认）
def convert_json_sheet(ws)           # json Sheet → v5 模板
```

判定模式分布（最终模板）：`any` / `fields` / `lock_readback` / `write_readback` / `sample` / `expect_error` / `manual`。

### 结果统计

最终 v5 模板：218 条机器可读用例 + 134 条人工 + 21 条跳过（通讯链路组暂不关心）。

## 延伸阅读

- 转换产出的 11 列格式怎么解析 → [[6-Swj-验收判定引擎accept_core]]
- 判定模式含义 → [[6-Swj-验收判定引擎accept_core]] / [[9-Swj-验收主页面与并发执行]]
