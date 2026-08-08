 # 1. 安装与入门

> 从零开始安装、配置 Git，理解仓库基本概念，并完成第一次提交。

---

## 目录

1. [安装](#安装)
2. [全局配置](#全局配置)
3. [仓库的概念](#仓库的概念)
4. [获取仓库](#获取仓库)
5. [首次提交](#首次提交)
6. [查看状态](#查看状态)
7. [常见问题](#常见问题)

---

## 安装

### Windows

1. 下载安装包：https://git-scm.com/download/win
2. 安装时建议选择 **Git Bash**（自带 Unix 风格终端）和 **"use Git from the command line"**
3. 验证：`git --version`

### macOS

```bash
# 方式一：Homebrew
brew install git

# 方式二：Xcode Command Line Tools
xcode-select --install
```

### Linux

```bash
# Debian/Ubuntu
sudo apt install git

# CentOS/RHEL
sudo yum install git
```

---

## 全局配置

首次使用必须配置**用户名**和**邮箱**——它们会被记录在每个提交里：

```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"
```

常用配置：

```bash
# 查看配置
git config --list

# 设置默认编辑器
git config --global core.editor "code --wait"

# 设置默认分支名（新仓库默认用 main 而非 master）
git config --global init.defaultBranch main

# 行尾换行处理（Windows 推荐）
git config --global core.autocrlf true

# 常用别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --all"
```

> 💡 配置作用域：`--global`（当前用户所有仓库）> `--local`（当前仓库，默认）。仓库级配置会覆盖全局。

---

## 仓库的概念

一个 Git 仓库（repository）是项目的完整历史。核心结构：

| 区域 | 说明 | 对应命令 |
|------|------|---------|
| 工作区 | 你实际编辑的文件 | 普通编辑 |
| 暂存区（Index） | 记录"准备提交哪些变更" | `git add` |
| 本地仓库（HEAD） | 已提交的历史 | `git commit` |
| 远程仓库 | 服务器上的副本 | `git push` / `git pull` |

详细三区域模型见 [[0-Git 使用说明]] 首页和 [[2-Git-核心命令与提交]]。

---

## 获取仓库

### 新建仓库（git init）

```bash
git init                # 在当前目录初始化
git init my-project     # 初始化到指定目录
```

初始化后目录中会生成隐藏的 `.git/` 文件夹（仓库的核心，不要手动删改）。

### 克隆已有仓库（git clone）

```bash
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git my-folder   # 指定目录名
git clone git@github.com:user/repo.git                  # SSH 方式
```

---

## 首次提交

```bash
# 1. 初始化仓库
git init

# 2. 添加文件到暂存区（暂存全部）
git add .
# 或指定文件
git add README.md

# 3. 提交
git commit -m "init: 初始化项目"

# 4. 查看结果
git log --oneline
```

---

## 查看状态

```bash
git status                # 查看工作区/暂存区状态
git status --short        # 精简显示（-s）
git diff                  # 查看未暂存的改动
git diff --staged         # 查看已暂存的改动（--cached 同义）
```

`git status` 是最常用的命令之一，它会明确告诉你：改了哪些文件、哪些已暂存、哪些还没跟踪。

---

## 常见问题

**Q：提交时提示 "Please tell me who you are"？**
A：还没配全局用户名邮箱，回到[全局配置](#全局配置)设置。

**Q：不小心把大文件或密钥提交了怎么办？**
A：立即用 `git rm --cached` 从索引移除，并修改历史（见 [[5-Git-历史与回滚]] 的改写历史部分），同时务必更换密钥。

**Q：`git init` 和 `git clone` 什么区别？**
A：`init` 从零建一个空仓库；`clone` 复制一个已有仓库（含全部历史）。

**Q：想忽略某些文件？**
A：创建 `.gitignore` 文件（详见 [[8-Git-最佳实践与FAQ]]）。

---

## 下一步

- 学会暂存区与提交 → [[2-Git-核心命令与提交]]
- 分支与合并 → [[3-Git-分支与合并]]
- 推送到远程 → [[4-Git-远程协作与PR]]
- 完整手册导航 → [[0-Git 使用说明]]
