# 7. 验收报告导出 accept_report.py

> 一键导出「三件套」：Excel 明细、Word 结论页、报文附录。依赖 openpyxl + python-docx。本笔记讲清核心导出函数的实现方式。

## 7.1 导出三件套

```python
def export_excel(path, info, results, manual_cases)   # 验收明细.xlsx
def export_word(path, info, results, manual_cases)    # 验收结论.docx
def export_raw(path, results)                         # 原始报文附录.txt
def export_all(folder, info, results, manual_cases) -> {"excel","word","raw"}
```

`info`（抬头）与 `results`（结果列表）结构：

```python
info = {"project","operator","tester","test_time","device","template",
        "total","pass","fail","manual","rate"}
results = [{"case": case, "verdict": "PASS/FAIL/MANUAL", "actual": ...,
            "ms": 120, "reason": "", "raw_tx": ..., "raw_rx": ...,
            "raw_msgs": [{"dir":"发/收","src":...,"msg":...}]}]
```

## 7.2 值格式化 _fmt

```python
def _fmt(v):
    if v is None: return ""
    if isinstance(v, (dict, list)):
        return json.dumps(v, ensure_ascii=False)   # dict/list 转 JSON 字符串
    return str(v)
```

**作用**：把结果值转展示字符串，dict/list（如 MQTT 的 `{"ts":26}` 或群控样本 `[1,1,1]`）序列化成 JSON，None 转空串。

## 7.3 明细行抽取 _case_fields（核心）

```python
_DETAIL_COLS = ["用例编号","协议","分组","用例名称","命令","目标地址",
                "下发参数","期望值","实测值","判定","耗时(ms)","失败原因"]

def _case_fields(case, res):
    expect = case.get("expect") or {}
    mode = expect.get("mode", "any")
    # 1) 期望值列按 mode 显示
    if mode == "fields":   exp = _fmt(expect.get("fields"))
    elif mode == "range":  exp = f"{expect.get('min')}~{expect.get('max')}"
    elif mode == "eq":     exp = str(expect.get("value"))
    elif mode == "any":    exp = "读到值即可"        # 读类：只验通讯成功
    elif mode == "manual": exp = "待确认"
    # 2) 写类/群控：优先显示模板期望值，未填则显示写入值
    if case.get("kind") in ("write", "group_ctrl"):
        ev = (case.get("expect") or {}).get("value")
        if ev is not None: exp = str(ev)
        elif case.get("val") is not None: exp = str(case.get("val"))
    # 3) 地址 / 命令 / 下发参数
    addr = case.get("addr")
    if addr is None: addr = ",".join(case.get("addrs") or []) or "(全部)"
    cmd = case.get("cmd") or case.get("fc")
    body = case.get("body") if case.get("body") is not None else case.get("val")
    return [case.get("case_no"), case.get("proto"), case.get("group"),
            case.get("name"), cmd, _fmt(addr), _fmt(body), exp,
            _fmt(res.get("actual")), res.get("verdict"), res.get("ms"),
            res.get("reason") or ""]
```

**期望值列显示规则**（重点逻辑）：

| 模式 | 期望值列显示 |
|------|-------------|
| `fields` | JSON 化期望字段 |
| `range` | `min~max` |
| `eq` | 数值 |
| `any`（读类） | "读到值即可" |
| `manual` | "待确认" |
| 写类/群控 | 优先模板期望值（如写 32 期望 31），未填则显示写入值 |

**写类为何覆盖**：读类的 `mode` 可能也是 `any`/`range`，但写类/群控的「期望值」语义不同——它表示回读比对目标，未填模板期望值时退化为「写入值」。

## 7.4 export_excel（Excel 明细）

配色常量：`FILL_PASS`(绿) / `FILL_FAIL`(红) / `FILL_MAN`(黄) / `FILL_HEAD`(蓝灰表头)。

**实现流程**：

1. 表头信息区（4 行，项目/操作人/测试方/被测设备/时间/模板/总用例/通过/失败/通过率）
2. 明细列头（`_DETAIL_COLS`，加粗蓝灰底）
3. 明细数据（遍历 results，`_case_fields` 抽行，判定列填色）
4. 人工确认区块（`manual_cases` 非空时单独附一块）

**判定列填色逻辑**：

```python
verdict = res.get("verdict")
if verdict == "PASS": fill = FILL_PASS
elif verdict == "FAIL": fill = FILL_FAIL
else: fill = FILL_MAN              # MANUAL 及其它
ws.cell(r, 10).fill = fill         # 第 10 列 = 判定列
if verdict == "MANUAL":            # 待确认行整行黄底
    for col in range(1, len(_DETAIL_COLS) + 1):
        ws.cell(r, col).fill = FILL_MAN
```

**列宽/对齐**：遍历所有单元格设 `Alignment(vertical="center", wrap_text=True)`，D 列(用例名称)和 G 列(下发参数)加宽。

## 7.5 export_word（Word 结论页）

章节：抬头表 → 一、结论汇总 → 二、失败用例摘要 → 三、待确认清单 → 四、签字确认。

**结论汇总实现**：

```python
p = doc.add_paragraph(
    f"本次验收共执行 {info['total']} 条自动化用例：通过 {info['pass']} 条、"
    f"失败 {info['fail']} 条、待确认 {info['manual']} 条，通过率 {info['rate']}。")
if info["fail"] == 0:
    p.add_run("全部通过。").bold = True
else:
    p.add_run("存在失败项，请见失败摘要。").bold = True
```

**失败摘要**：`fails = [res for res in results if res.get("verdict") == "FAIL"]`，4 列表格（编号/名称/实测值/失败原因）。

**待确认清单**：`manuals = [res for res in results if res.get("verdict") == "MANUAL"]`，3 列表格（编号/名称/实测值）。

## 7.6 export_raw（报文附录）

标题 + 每条用例一个分隔块：

```
[用例编号] 用例名称  判定:PASS 耗时:120ms
  发→ {...} / 收← {...}
```

**协议差异**（重要）：

```python
for m in msgs:
    d = "发→" if m.get("dir") == "发" else "收←"
    txt = str(m.get("msg") or "").strip()
    if proto == "mqtt":
        lines.append(f"  {d} {txt}")          # MQTT 直接打印原始 JSON，不做解析
    else:
        src = m.get("src") or ""
        lines.append(f"  {d} [{src}] {txt}")  # Modbus 带来源标签
```

**无 raw_msgs 时的回退**（raw_tx/raw_rx 配对）：

```python
txs = [t for t in str(res.get("raw_tx") or "").split("\n") if t]
rxs = [r for r in str(res.get("raw_rx") or "").split("\n") if r]
n = max(len(txs), len(rxs))
if n == 0: lines.append("  (无交互报文)")
for i in range(n):
    t = txs[i] if i < len(txs) else "(无)"
    r = rxs[i] if i < len(rxs) else "(无)"
    lines.append(f"  发→ {t}")
    lines.append(f"  收← {r}")
```

**实现逻辑**：`raw_tx`/`raw_rx` 可能含多条帧（`\n` 分隔），按行号一一配对，不足补 `(无)`。

## 7.7 export_all

```python
def export_all(folder, info, results, manual_cases):
    os.makedirs(folder, exist_ok=True)
    excel = os.path.join(folder, "验收明细.xlsx")
    word = os.path.join(folder, "验收结论.docx")
    raw = os.path.join(folder, "原始报文附录.txt")
    export_excel(excel, info, results, manual_cases)
    export_word(word, info, results, manual_cases)
    export_raw(raw, results)
    return {"excel": excel, "word": word, "raw": raw}
```

## 延伸阅读

- 判定结果怎么来 → [[6-Swj-验收判定引擎accept_core]]
- 报告导出按钮在哪 → [[9-Swj-验收主页面与并发执行]]
