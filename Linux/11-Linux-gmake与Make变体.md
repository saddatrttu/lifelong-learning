# 11. gmake 与 Make 变体

> `make` 有好几个"兄弟"，命令名都叫 make，语法却不完全兼容：Linux 默认是 **GNU make**，macOS/BSD 自带的是 **BSD make**，GNU make 在 BSD 系上安装后叫 **gmake**。理解这层关系，才能看懂为什么有些构建脚本里写的是 `gmake`。

---

## 目录

1. [make 家族的变体](#make-家族的变体)
2. [gmake 是什么](#gmake-是什么)
3. [为什么要用 gmake](#为什么要用-gmake)
4. [安装 gmake](#安装-gmake)
5. [检测当前 make](#检测当前-make)
6. [GNU 扩展特性清单](#gnu-扩展特性清单)
7. [Makefile 可移植写法](#makefile-可移植写法)
8. [需要 gmake 的项目](#需要-gmake-的项目)
9. [常见问题](#常见问题)

---

## make 家族的变体

`make` 不是单一程序，而是一族工具：

| make 变体 | 常见平台 | 命令名 | 特点 |
|-----------|----------|--------|------|
| **GNU make** | Linux、macOS(可装) | `make` / `gmake` | 功能最全，Linux 默认，事实标准 |
| **BSD make** | macOS 自带、FreeBSD/NetBSD/OpenBSD | `make` | 语法古老，缺 GNU 扩展 |
| dmake | Solaris 等 | `dmake` | 旧商用系统 |
| nmake | Windows (MSVC) | `nmake` | 微软专用，语法差异大 |
| ninja | 跨平台 | `ninja` | 新一代构建工具，非 make 但同类（配合 CMake） |

> 绝大多数 Linux 发行版上，`make` 就是 GNU make。区别集中在 macOS 和 BSD 系统上。

---

## gmake 是什么

**gmake = GNU make 的安装名**。在 macOS/BSD 上，系统自带的 `make` 是 BSD make，而 GNU make 被安装为 `gmake`，二者并存、互不覆盖：

```bash
# macOS（Homebrew 安装 GNU make，不会替换系统 make）
brew install make
gmake --version        # GNU Make 4.x
make --version         # 仍是 BSD make（Apple 版 3.81）

# FreeBSD
pkg install gmake
```

> 一句话：**在 Linux 上 `make` = gmake；在 macOS/BSD 上 `make` ≠ gmake**。很多开源项目的构建脚本写 `gmake`，就是为了确保用的绝对是 GNU make。

---

## 为什么要用 gmake

因为现代 Makefile（尤其是 autotools/automake 生成的和本指南 [[10-Linux-Makefile构建工具]] 里写的）几乎都用到了 **GNU 扩展**，BSD make 不认：

```bash
# 典型报错：在 BSD make 上跑 GNU 风格 Makefile
# "make: don't know how to make %.o"
# "make: missing operator"
```

用 gmake 就能直接跑，不用改 Makefile。

---

## 安装 gmake

```bash
# Debian/Ubuntu（Linux 上 make 已是 GNU make，装 make 即得 gmake 功能）
sudo apt install make

# CentOS/RHEL
sudo yum install make

# macOS（安装后命令叫 gmake）
brew install make
# 如需让 make 也指向 GNU：把 /opt/homebrew/opt/make/libexec/gnubin 加入 PATH

# FreeBSD/OpenBSD
pkg install gmake

# 验证
gmake --version
```

> Windows 上 Windows 版 Git 自带的 Git Bash 里也有 GNU make（`make`）；MSYS2/Cygwin 可 `pacman -S make` 安装。

---

## 检测当前 make

```bash
# 看是不是 GNU make（关键看第一行是否含 "GNU Make"）
make --version
gmake --version

# 输出示例
# GNU Make 4.3
# Built for x86_64-pc-linux-gnu
```

在脚本里判断并选择：

```bash
# 优先 gmake，没有则退回 make
MAKE := $(shell command -v gmake 2>/dev/null || echo make)
$(MAKE) -j4
```

---

## GNU 扩展特性清单

BSD make 不支持（或语法不同）的 GNU make 特性，看到这些就要用 gmake：

| GNU 扩展 | 说明 | BSD make 情况 |
|----------|------|----------------|
| `%.o: %.c` 模式规则 | 通配模式规则 | 不支持（旧语法 `.c.o:`） |
| `.PHONY` 伪目标 | 声明非文件目标 | 部分支持，写法不同 |
| `$@` `$<` `$^` 自动变量 | 规则内自动变量 | 只支持 `$@` 等部分 |
| `$(wildcard ...)` | 通配展开函数 | 不支持 |
| `$(patsubst ...)` `$(foreach ...)` | 字符串/列表函数 | 不支持 |
| `ifeq` `ifdef` 条件 | 条件判断 | 语法不同（`.if`） |
| `:=` 立即展开 | 变量赋值方式 | 部分支持 |
| `$(shell ...)` | 调用 shell | 不支持 |
| `-MMD -MP` 自动依赖 | 头文件依赖 | 不支持 |
| `$(MAKE)` 递归 make | 子目录构建 | 支持但扩展少 |
| `GNUmakefile` | 优先读取的专属文件名 | 不识别 |

> 判断标准很简单：**Makefile 里用了 `%` 模式规则、`$(...)` 函数、`ifeq`、`.PHONY`，基本就是 GNU make 专属**，跨平台项目应写 `gmake` 调用。

---

## Makefile 可移植写法

如果想一份 Makefile 同时兼容 GNU make 和 BSD make：

```makefile
# 方式一：少用 GNU 扩展，用经典写法
OBJS = main.o util.o
app: $(OBJS)
	cc -o app $(OBJS)
.c.o:                    # 旧式后缀规则（两可）
	cc -c $<

# 方式二：在 Makefile 顶部做检测，用到扩展时给提示
GNUMAKE := $(shell gmake --version >/dev/null 2>&1 && echo yes)
ifeq ($(GNUMAKE),yes)
# ...GNU 专属逻辑
else
$(error 请用 gmake 构建：brew install make)
endif
```

> 现实做法：现代项目**基本都默认 GNU make**（甚至直接写 `gmake`），可移植性主要靠"构建文档里写明先装 GNU make"。

---

## 需要 gmake 的项目

| 场景 | 说明 |
|------|------|
| OpenWrt 路由器固件 | 整个构建系统明确要求 gmake |
| autotools 项目 | `./configure` 生成的 Makefile 依赖 GNU 扩展 |
| Linux 内核模块 | 依赖 GNU make 特性（`make -C` 等） |
| GNU 官方软件源码 | README 常写 "run gmake" |
| 嵌入式交叉编译工具链 | 构建脚本多用 gmake |
| CMake 生成的 Makefile | 默认生成 GNU make 风格 |

```bash
# OpenWrt 典型用法
make -C openwrt V=s   # 内部会调用 gmake 执行编译
```

---

## 常见问题

**Q：macOS 上 `make` 报 `don't know how to make %.o`？**
A：系统 make 是 BSD make。`brew install make` 后改用 `gmake`。

**Q：`gmake: command not found`？**
A：没装 GNU make。Linux 装 `make` 包（命令就叫 make），macOS/BSD 装 `make`/`gmake` 包后命令为 gmake。

**Q：同一份源码在 Linux 编译正常，在 macOS 上失败？**
A：大概率是 BSD make 不认 GNU 扩展。用 `gmake` 代替 `make` 重试，或看项目构建文档要求。

**Q：gmake 和 make 能同时用吗？**
A：能。它们读取同一个 `Makefile`，只是执行器不同；gmake 兼容性更好。递归 make（`$(MAKE)`）会正确传递你调用的那个名字。

**Q：写 Makefile 到底用 make 还是 gmake？**
A：现代项目默认 GNU make，跨平台构建说明里写明"先安装 GNU make / 用 gmake"。可移植写法见上文。

---

## 下一步

- 完整 Makefile 语法 → [[10-Linux-Makefile构建工具]]
- Shell 脚本编程（构建脚本搭配） → [[8-Linux-Shell脚本编程]]
- 编译常见坑 → [[9-Linux-常见问题与实战]]
- 完整手册导航 → [[0-Linux 使用指南]]
