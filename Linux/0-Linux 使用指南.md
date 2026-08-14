# Linux 使用指南

> Linux 是一个开源的多用户、多任务操作系统内核，常见发行版有 Ubuntu、CentOS、Debian 等。掌握命令行是使用 Linux 的基础，也是自动化、运维与后端开发的必备技能。
>
> 本文是一套完整的参考手册，采用 **MOC + 多笔记** 结构，与 [[0-Git 使用说明]]、[[0-Claude Code 使用指南]] 同一套整理思路。

---

## 阅读路径

- **新手**：按顺序 [[1-Linux-入门与环境]] → [[2-Linux-文件与目录操作]] → [[3-Linux-文件内容查看与编辑]] → [[4-Linux-权限与用户管理]]
- **实战提升**：[[5-Linux-进程与系统管理]] → [[6-Linux-网络与远程管理]]
- **进阶**：[[7-Linux-文本处理三剑客]] → [[8-Linux-Shell脚本编程]] → [[10-Linux-Makefile构建工具]] → [[11-Linux-gmake与Make变体]]
- **查坑排错**：[[9-Linux-常见问题与实战]]
- **快速查表**：直接跳到目标笔记，每个笔记开头带目录锚点

---

## 目录

| # | 笔记 | 一句话内容 |
|---|------|-----------|
| 0 | [[0-Linux 使用指南\|本页 MOC]] | 导航入口 |
| 1 | [[1-Linux-入门与环境]] | 发行版、Shell、路径、帮助命令、通配符 |
| 2 | [[2-Linux-文件与目录操作]] | ls/cd/cp/mv/rm、find、软硬链接、归档压缩 |
| 3 | [[3-Linux-文件内容查看与编辑]] | cat/less/head/tail、vim、重定向与管道 |
| 4 | [[4-Linux-权限与用户管理]] | chmod/chown、用户与组、sudo、进程权限 |
| 5 | [[5-Linux-进程与系统管理]] | ps/top/kill、systemctl、df/du、cron 定时任务 |
| 6 | [[6-Linux-网络与远程管理]] | ip/ss、ping、curl/wget、SSH 远程、scp |
| 7 | [[7-Linux-文本处理三剑客]] | grep、sed、awk、sort/uniq、xargs 数据加工 |
| 8 | [[8-Linux-Shell脚本编程]] | 变量、判断、循环、函数、脚本实战 |
| 9 | [[9-Linux-常见问题与实战]] | 权限/路径/编码等常见坑、速查表 |
| 10 | [[10-Linux-Makefile构建工具]] | 规则、变量、模式规则、自动依赖、并行构建 |
| 11 | [[11-Linux-gmake与Make变体]] | GNU make 与 BSD make、gmake 安装、GNU 扩展、可移植写法 |

---

## 核心心智模型

**命令的通用结构**——几乎所有的 Linux 命令都长这样：

```
命令 [-选项] [参数...]
    ↑       ↑        ↑
 做什么    怎么做的     对谁做
```

例：`ls -l /home` → 用 `-l` 长格式查看 `/home` 目录。

**一切皆文件**：Linux 里设备、网络、进程状态都以文件形式暴露在 `/dev`、`/proc`、`/sys` 下，所以文件操作命令（ls/cat/重定向）几乎可以处理一切对象。

**组合优于单兵**：单条命令能力有限，通过**管道 `|` 串联**和**重定向 `> / >>`** 可组成强大的处理流水线（详见 [[3-Linux-文件内容查看与编辑]] 与 [[7-Linux-文本处理三剑客]]）。

---

## 快速参考

### 最常用的 10 个命令

| 命令 | 作用 |
|------|------|
| `ls` | 列出目录内容 |
| `cd <dir>` | 切换目录 |
| `cat <file>` | 查看文件内容 |
| `cp <src> <dst>` | 复制文件 |
| `mv <src> <dst>` | 移动/重命名 |
| `rm <file>` | 删除文件 |
| `mkdir <dir>` | 创建目录 |
| `grep <pattern> <file>` | 文本搜索 |
| `chmod` | 修改权限 |
| `sudo <cmd>` | 以管理员权限执行 |

---

## 关联指南

- [[0-Git 使用说明]] — Git 在 Linux 命令行上运行，常与 `vim`、管道配合使用
- [[0-SVN 使用指南]] — SVN 是集中式版本控制，命令风格与 Git 不同但思路相通
- [[0-Claude Code 使用指南]] — Claude Code 的 Hooks/脚本自动化依赖 Shell 语法
- [[0-自动化测试方法论]] — 接口测试、Appium 等常在 Linux/容器环境执行
- 官方文档：https://www.gnu.org/software/coreutils/（GNU 核心工具集）
- 在线练习：https://linuxcommand.org / https://overthewire.org/wargames/bandit
