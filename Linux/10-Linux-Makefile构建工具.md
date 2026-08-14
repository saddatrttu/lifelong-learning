# 10. Makefile 构建工具

> Make 是一个经典构建自动化工具，通过 Makefile 描述"目标-依赖-命令"的关系，自动判断哪些文件需要重新构建。C/C++ 项目、内核、诸多开源软件的编译都靠它。

---

## 目录

1. [为什么需要 Make](#为什么需要-make)
2. [基本规则](#基本规则)
3. [变量](#变量)
4. [自动变量与模式规则](#自动变量与模式规则)
5. [伪目标与特殊目标](#伪目标与特殊目标)
6. [条件与函数](#条件与函数)
7. [自动生成依赖](#自动生成依赖)
8. [并行构建与调试](#并行构建与调试)
9. [实战示例](#实战示例)
10. [常见问题](#常见问题)

---

## 为什么需要 Make

编译一个大项目时，只有**被修改的文件**才需要重新编译，其余可以直接复用。Make 通过比较**目标和依赖的时间戳**决定是否重跑命令：

```
目标: 依赖1 依赖2
	命令            ← 依赖比目标新时执行
```

```bash
# 最简 Makefile
app: main.o util.o
	gcc -o app main.o util.o

main.o: main.c
	gcc -c main.c

util.o: util.c
	gcc -c util.c

# 然后执行
make
```

`make` 会在当前目录找 `Makefile`/`makefile`/`GNUmakefile`，默认构建第一个目标。常用选项：

```bash
make                # 构建默认目标
make app            # 构建指定目标
make -j4            # 并行 4 个任务（大幅加速）
make clean          # 执行 clean 目标
make -n             # 只预览要执行的命令（dry-run）
make -f my.mk       # 指定 Makefile 文件名
make -B             # 强制重建（忽略时间戳）
```

---

## 基本规则

规则结构：

```makefile
目标: 依赖
	命令一
	命令二
```

要点：
- **命令前必须是 Tab 缩进**，不能用空格（最高频的坑）
- 命令是传给 shell 执行的，每个命令独立 shell
- 规则之间可以有空行

```makefile
hello: main.c
	gcc -o hello main.c
	echo "构建完成"
```

> 传统写法一条规则编译+链接，改进版见下方"自动变量与模式规则"。

---

## 变量

```makefile
# 赋值方式
CC     = gcc          # 普通赋值（递归展开）
CFLAGS = -Wall -g     # 编译选项
SRCS   = main.c util.c
OBJS   = main.o util.o
PREFIX = /usr/local

# := 立即展开（区别于 = 的递归展开，两者初学时混用即可）
TARGET := app

# ?= 仅在未定义时赋值（允许外部覆盖）
CFLAGS ?= -O2

# 变量使用
$(CC) -o app $(OBJS)
${CFLAGS}            # {} 与 () 等价

# 追加
CFLAGS += -DDEBUG

# 自动推导（隐式规则）
# 不写 gcc -c 命令，make 也知道 .o 依赖同名 .c 生成
```

**预定义变量**（GNU Make）：

| 变量 | 含义 |
|------|------|
| `CC` | C 编译器（默认 cc） |
| `CXX` | C++ 编译器（默认 g++） |
| `CFLAGS` | C 编译选项 |
| `CXXFLAGS` | C++ 编译选项 |
| `LDFLAGS` | 链接选项 |
| `LDLIBS` | 链接库，如 `-lm -lpthread` |
| `PREFIX` | 安装前缀（默认 /usr/local） |

---

## 自动变量与模式规则

### 自动变量（规则内使用）

| 变量 | 含义 |
|------|------|
| `$@` | 目标文件名 |
| `$<` | 第一个依赖 |
| `$^` | 所有依赖（去重） |
| `$?` | 比目标新的依赖 |
| `$*` | 去除后缀的目标名（模式匹配部分） |

```makefile
app: main.o util.o
	gcc -o $@ $^          # gcc -o app main.o util.o
```

### 模式规则（% 匹配任意前缀）

```makefile
# 任何 .o 依赖同名 .c
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 编译 C++ 同样一条
%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@
```

```makefile
# 完整版：一条模式规则替代所有 .c→.o 手工规则
CC     = gcc
CFLAGS = -Wall -g
SRCS   = $(wildcard *.c)     # 函数：列出当前目录所有 .c
OBJS   = $(SRCS:.c=.o)       # 替换后缀
TARGET = app

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

> `$(wildcard *.c)` 和 `$(SRCS:.c=.o)` 是两个最常用的函数式写法，能省掉大量手工罗列。

---

## 伪目标与特殊目标

**伪目标**：不代表文件的目标，只执行命令（常见 `clean`、`install`、`test`）。

```makefile
.PHONY: clean install test

clean:
	rm -f $(OBJS) $(TARGET)

install:
	install -m 755 $(TARGET) $(PREFIX)/bin

test: $(TARGET)
	./$(TARGET) --test
```

> 必须声明 `.PHONY`：否则若目录里恰好有个叫 `clean` 的文件，make 认为它已"最新"而跳过命令。

其他特殊目标：

| 目标 | 作用 |
|------|------|
| `.PHONY` | 声明伪目标 |
| `.DEFAULT_GOAL := app` | 指定默认目标 |
| `.SUFFIXES` | 后缀规则（旧式，已少用） |
| `.DELETE_ON_ERROR` | 命令失败时删除目标文件 |

---

## 条件与函数

### 条件判断

```makefile
ifeq ($(OS),Windows_NT)
	RM = del /Q
else
	RM = rm -f
endif

ifdef DEBUG
	CFLAGS += -g -O0
endif

# 常用：区分平台、调试/发布
```

### 常用函数

```makefile
# 文件名操作
$(basename src/main.c)     # src/main
$(dir src/main.c)          # src/
$(notdir src/main.c)       # main.c
$(suffix main.c)           # .c
$(addprefix obj/,$(SRCS))  # obj/main.c obj/util.c

# 列表操作
$(foreach x,$(SRCS),$(x:.c=.o))    # 对每个元素做替换
$(filter %.c,$(SRCS))              # 过滤出 .c
$(patsubst %.c,%.o,$(SRCS))        # 模式替换（同 $(SRCS:.c=.o)）

# 文本操作
$(shell date +%F)          # 执行 shell 命令
$(info msg) / $(warning msg)   # 打印信息
$(strip $(VAR))            # 去首尾空格
$(wildcard src/*.c)        # 展开匹配的文件
```

> 完整函数清单见 `info make` 的 "Functions" 章节。

---

## 自动生成依赖

C 的头文件改动时 `.o` 不会自动重编（make 只跟踪规则里写明的依赖）。用编译器生成依赖文件解决：

```makefile
CFLAGS += -MMD -MP          # 编译时自动生成 .d 依赖文件

DEPS = $(OBJS:.o=.d)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

-include $(DEPS)            # 引入依赖文件（存在才引入）

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(DEPS) $(TARGET)

.PHONY: clean
```

> `-MMD` 让编译器把 `#include` 的头文件写进 `.d` 文件，`-include` 引入后，头文件改动就能触发重编。这是真实项目的标准做法。

---

## 并行构建与调试

```bash
make -j4            # 4 个任务并行（通常 = CPU 核数，可 nproc 查看）
make -j$(nproc)     # 全核并行
```

> 注意：并行时同一目标的多个命令不能并行；依赖关系错误的 Makefile 在 `-j` 下会表现出**偶发失败**。

调试技巧：

```bash
make -n             # 预览命令，不改动
make -p             # 打印所有变量/规则（排查变量值）
make --debug=v      # 详细输出，看谁触发了重建
```

---

## 实战示例

**C 项目模板 `Makefile`：**

```makefile
# 配置
CC       = gcc
CFLAGS   = -Wall -Wextra -g -MMD -MP
LDLIBS   = -lm
TARGET   = app
SRCS     = $(wildcard src/*.c)
OBJS     = $(SRCS:src/%.c=build/%.o)
DEPS     = $(OBJS:.o=.d)

build:
	mkdir -p build

# 链接
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)

# 编译（目标目录不同，用模式规则）
build/%.o: src/%.c | build
	$(CC) $(CFLAGS) -c $< -o $@

-include $(DEPS)

.PHONY: all clean test install
all: $(TARGET)

test: $(TARGET)
	./$(TARGET) --test

install: $(TARGET)
	install -m 755 $(TARGET) $(PREFIX)/bin

clean:
	rm -rf build $(TARGET)
```

> 这个模板把源码放 `src/`、中间文件放 `build/`，带自动依赖、并行安全、测试与安装目标，可直接套用。

---

## 常见问题

**Q：`missing separator` / `recipe commences before first target`？**
A：命令行的缩进**必须是 Tab 不是空格**。`cat -A Makefile` 可检查（Tab 显示为 `^I`）。

**Q：目标总是"最新"，改了源码却不重编？**
A：依赖没写全（漏了 `.h`），用 `-MMD -MP` 自动依赖；或 `make -B` 强制重编验证。

**Q：`make: Nothing to be done`？**
A：目标已最新。想强制重建用 `make -B` 或删掉目标文件。

**Q：`file not recognized: File format not recognized`？**
A：编译/链接了不该参与的文件（如 `.d` 当源文件），检查 `OBJS`、`SRCS` 变量展开是否正确（`make -p` 查看）。

**Q：make 找不到我的 Makefile？**
A：文件名须为 `Makefile`/`makefile`，或用 `make -f 我的文件`。

---

## 下一步

- gmake 与 make 变体（BSD/GNU 区别） → [[11-Linux-gmake与Make变体]]
- Shell 脚本编程（构建脚本搭配 Make 使用） → [[8-Linux-Shell脚本编程]]
- 文本处理三剑客（处理构建产物/日志） → [[7-Linux-文本处理三剑客]]
- 进程与系统管理（并行构建资源） → [[5-Linux-进程与系统管理]]
- 完整手册导航 → [[0-Linux 使用指南]]
