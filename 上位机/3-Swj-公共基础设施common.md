# 3. 公共基础设施 common.py

> 所有协议标签页与主框架都从这里导入，保证全网视觉/交互统一。无外部依赖，只依赖 tkinter + time + random。本笔记逐函数讲解实现方式与作用。

## 3.1 配色与字体常量

```python
C_SEND    = "#1890FF"   # 发送/上行/发布 - 蓝
C_RECV    = "#52C41A"   # 接收/下行/应答 - 绿
C_ERROR   = "#F5222D"   # 异常/报错 - 红
C_DISABLE = "#86909C"   # 禁用/闲置 - 灰
C_TEXT    = "#333333"   # 正常日志文本
C_BG      = "#F6F8FA"   # 页面底色
C_CARD    = "#FFFFFF"   # 卡片/列表底色
C_BORDER  = "#E5E6EB"   # 边框分割线
C_PAUSE   = "#FAAD14"   # 暂停 - 橙

FONT      = ("Microsoft YaHei UI", 9)
FONT_B    = ("Microsoft YaHei UI", 9, "bold")
FONT_MONO = ("Consolas", 9)
```

协议名 / 状态映射：

```python
PROTO_NAME  = {"645": "DL/T645 多电表", "modbus": "Modbus 轮询", "mqtt": "MQTT JSON"}
STATE_COLOR = {"running": C_RECV, "paused": C_PAUSE, "error": C_ERROR, "stopped": C_DISABLE}
STATE_TEXT  = {"running": "运行中", "paused": "已暂停", "error": "异常", "stopped": "已停止"}
```

## 3.2 dot / set_dot（状态指示灯）

**作用**：在 ttk 容器上放一个小圆点表示运行/停止/异常状态。

**实现方式**：

```python
def dot(parent, color, size=10, bg=None):
    if bg is None:
        try:
            bg = parent.cget("bg")      # 普通 tk 容器可取到 bg
        except tk.TclError:
            bg = C_CARD                 # ttk 容器无 -bg 属性，抛 TclError 时回退
    c = tk.Canvas(parent, width=size, height=size, highlightthickness=0, bg=bg)
    c.create_oval(1, 1, size - 1, size - 1, fill=color, outline="")
    return c
```

关键点：**ttk 容器没有 `-bg` 属性**（`cget("bg")` 会抛 `TclError`），所以不能直接取背景色，需要显式传或回退默认卡片色。圆点用 `Canvas.create_oval` 画实心椭圆（`outline=""` 去边框）。

```python
def set_dot(canvas, color):
    canvas.delete("all")                            # 清掉旧圆点
    size = int(canvas.cget("width"))
    canvas.create_oval(1, 1, size - 1, size - 1, fill=color, outline="")
```

**作用**：改变已有圆点颜色。通过 `delete("all")` 再重画实现「换色」，因为 Canvas 没有直接改 ovals 填充色的简单 API。size 从 canvas 自身宽度取，避免硬编码。

## 3.3 hex_spaced

```python
def hex_spaced(b):
    return " ".join(f"{x:02X}" for x in b)
```

**作用**：字节串 → 空格分隔大写十六进制文本（`b"\x01\x06"` → `"01 06"`），是报文日志显示的标准格式。`f"{x:02X}"` 保证每个字节两位、不足补 0、大写。

## 3.4 apply_style（统一 ttk 样式）

```python
def apply_style(root):
    st = ttk.Style(root)
    try:
        st.theme_use("clam")
    except tk.TclError:
        pass                        # 某些平台 clam 不可用则忽略，用默认主题
    st.configure(".", font=FONT, background=C_BG, foreground=C_TEXT)
    st.configure("TFrame", background=C_BG)
    st.configure("Card.TFrame", background=C_CARD)
    # ... TLabel / Card.TLabel / Title.TLabel / TButton / Send.TButton /
    #     TNotebook / TCheckbutton / TRadiobutton / TLabelframe / Treeview ...
    st.map("Treeview", background=[("selected", "#E6F4FF")])
```

**作用**：全 App 唯一的样式入口。`st.configure("样式名.TFrame", ...)` 定义派生样式（`Card.TFrame` 白底卡片、`Send.TButton` 蓝色文字按钮）。`st.map` 定义状态映射（选中行高亮）。

## 3.5 Task（并发任务看板数据源）

```python
class Task:
    _seq = 0
    def __init__(self, proto, summary):
        Task._seq += 1                 # 类级自增，全局唯一 id
        self.id = Task._seq
        self.proto = proto             # "645"/"modbus"/"mqtt"
        self.summary = summary
        self.state = "running"
        self.start = time.time()
        self.count = 0
        self.delay = random.randint(8, 40)   # 模拟通讯延迟(ms)
```

`elapsed()` 实现：

```python
def elapsed(self):
    s = int(time.time() - self.start)
    return f"{s//3600:02d}:{(s%3600)//60:02d}:{s%60:02d}"   # 秒 → HH:MM:SS
```

`tick()` 实现：

```python
def tick(self):
    if self.state == "running":
        self.count += 1
        if random.random() < 0.01:     # 1% 概率演示异常红点
            self.state = "error"
        self.delay = max(5, min(120, self.delay + random.randint(-6, 6)))  # 延迟抖动
```

**作用**：首页并发看板的模拟数据源。`delay` 是演示用的伪延迟（非真实测速），`tick()` 每 500ms 被主循环调一次，制造「任务在跑、延迟波动、偶尔异常」的动态效果。

## 3.6 LogPanel（滚动报文面板）

**作用**：带自动滚底、颜色 tag、行数限制、镜像的日志面板，是三个协议页 + 验收页共用的报文显示控件。

### __init__ 实现

```python
class LogPanel(ttk.LabelFrame):
    def __init__(self, parent, title, height=8):
        super().__init__(parent, text=title, style="TLabelframe")
        self.mirrors = []              # 额外同步写入的 LogPanel
        self.text = tk.Text(self, height=height, font=FONT_MONO, bg=C_CARD,
                            fg=C_TEXT, wrap="none", state="disabled",
                            relief="flat", highlightthickness=0)
        vsb = ttk.Scrollbar(self, orient="vertical", command=self.text.yview)
        self.text.configure(yscrollcommand=vsb.set)
        self.text.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

        for tag, col in [("send", C_SEND), ("recv", C_RECV), ("err", C_ERROR),
                         ("dis", C_DISABLE), ("info", C_TEXT)]:
            self.text.tag_config(tag, foreground=col)   # 五色 tag

        self.follow = tk.BooleanVar(value=True)
        self.text.bind("<Enter>", lambda e: self.follow.set(False))   # 鼠标进入暂停跟随
        # 底部「恢复自动跟随」复选钮
```

关键设计：`state="disabled"` 的 Text 平时禁止编辑，写日志时临时切 `normal` 再切回。`wrap="none"` 让长报文不换行（配合横向滚动）。

### log 实现（核心）

```python
def log(self, msg, tag="info"):
    self.text.configure(state="normal")
    self.text.insert("end", msg + "\n", tag)
    if int(self.text.index("end-1c").split(".")[0]) > 1000:   # 超 1000 行
        self.text.delete("1.0", "200.0")                       # 删最前 200 行
    if self.follow.get():
        self.text.see("end")                    # 自动滚底
    self.text.configure(state="disabled")
    for m in self.mirrors:                      # 同步镜像
        # ... 同样的 insert / 行数限制 / see("end")
```

**行数限制**用 `self.text.index("end-1c")` 取当前总行数，超过 1000 就 `delete("1.0", "200.0")` 删掉最旧 200 行，防止日志无限增长拖慢 UI。

**镜像机制**：`self.mirrors` 是「额外同步写入的 LogPanel」列表（如首页并发看板镜像面板），主面板写日志时同步写到每个镜像面板——各自独立 follow/滚动状态，互不影响。

### clear / get_all

```python
def clear(self):
    self.text.configure(state="normal")
    self.text.delete("1.0", "end")
    self.text.configure(state="disabled")

def get_all(self):
    return self.text.get("1.0", "end-1c")
```

## 延伸阅读

- 配置读写 → [[4-Swj-配置持久化persist]]
- 三个 tab 如何复用 LogPanel/Task → [[8-Swj-三协议模块]]
