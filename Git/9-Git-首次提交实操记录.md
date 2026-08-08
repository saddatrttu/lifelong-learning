# 9. 首次提交实操记录

> 一次真实的「本地笔记库 → GitHub 远程仓库」完整流程实录：从初始化到推送，每一步的命令、含义与踩过的坑。适合对照 [[1-Git-安装与入门]] 边学边练。

---

## 目录

1. [操作背景](#操作背景)
2. [前置检查](#前置检查)
3. [完整操作步骤](#完整操作步骤)
4. [本次踩过的坑](#本次踩过的坑)
5. [日常更新三板斧](#日常更新三板斧)
6. [相关笔记](#相关笔记)

---

## 操作背景

- **目标**：把本地目录 `E:\Obhub\huan`（Obsidian 知识库）提交到 GitHub 远程仓库 `https://github.com/saddatrttu/lifelong-learning`
- **环境**：Windows + Git 2.53.0（`git --version` 查看）
- **现状**：本地目录还不是 git 仓库，远程仓库已有 1 个 `README.md`（创建仓库时自动生成）

---

## 前置检查

### 1. 检查 git 是否安装

```bash
git --version
```

### 2. 检查身份是否已配置

```bash
git config --global user.name
git config --global user.email
```

> 如果没有输出，必须先用下面两条命令配置，否则无法提交（提交记录里会没有作者）：
>
> ```bash
> git config --global user.name "你的名字"
> git config --global user.email "你的邮箱@example.com"
> ```
>
> 邮箱建议用 GitHub 账号绑定的邮箱，这样提交会关联到你的头像。`--global` 表示全局生效（本机所有仓库共用）。

### 3. 查看远程仓库现状

```bash
git ls-remote https://github.com/saddatrttu/lifelong-learning
```

> 输出里有 `refs/heads/main`，说明远程有内容（至少一个分支），后面需要先 pull 再 push。

---

## 完整操作步骤

### 第 1 步：配置身份（首次使用必做）

```bash
git config --global user.name "saddatrttu"
git config --global user.email "27766161830@qq.com"
```

### 第 2 步：初始化本地仓库

```bash
git init
```

> 在目录下生成隐藏的 `.git` 文件夹，从此该目录归 git 管理。

### 第 3 步：关联远程仓库

```bash
git remote add origin https://github.com/saddatrttu/lifelong-learning
git remote -v          # 验证：显示 fetch 和 push 两条地址
```

> `origin` 是远程仓库的**代称**（默认名字），以后 `git push origin main` 就是推送到这个地址。

### 第 4 步：先拉取远程已有内容

```bash
git pull origin main
```

> 远程已有 `README.md`，不先拉下来直接 push 会被拒绝（远程有新提交而本地没有）。输出 `* [new branch] main -> origin/main` 表示成功。

### 第 5 步：统一分支名（关键一步）

```bash
git branch -M main
```

> **为什么需要**：`git init` 默认创建 `master` 分支，而 GitHub 默认分支叫 `main`。`-M` 把本地分支强制重命名为 `main`，与远程对齐，否则 push 时分支名不一致。

### 第 6 步：查看状态确认文件

```bash
git status
```

> 未跟踪文件显示在 `Untracked files` 下，确认没有意外文件后再进行下一步。

### 第 7 步：加入暂存区

```bash
git add .
```

> `.` 表示"当前目录下所有文件"。`git status` 里 `A` 前缀（Added）表示已进入暂存区。

### 第 8 步：提交

```bash
git commit -m "初始提交：添加全部学习笔记"
```

> `-m` 后面是本次提交的说明。输出 `[main 4bfb0e7]` 表示提交成功，`4bfb0e7` 是提交编号（commit hash 的前几位）。

### 第 9 步：推送

```bash
git push -u origin main
```

> `-u` 记住本地 `main` 和远程 `main` 的对应关系，以后只需 `git push`。输出 `0982c08..4bfb0e7 main -> main` 表示推送成功（旧提交 → 新提交）。

### 第 10 步：验证

```bash
git status        # 显示 up to date，working tree clean
git log --oneline # 查看提交历史
```

---

## 本次踩过的坑

### 坑 1：本地 master 与远程 main 不一致

| 问题 | 本地默认分支叫 `master`，远程叫 `main`，直接 push 会错乱 |
|------|------|
| 解决 | `git branch -M main` 强制改名对齐 |

### 坑 2：LF 换行符警告刷屏

| 问题 | `git add` 时提示 `LF will be replaced by CRLF` |
|------|------|
| 原因 | Windows 用 CRLF 换行，Linux/Mac 用 LF，git 在做自动转换提示 |
| 结论 | **无害警告**，可忽略 |

### 坑 3：PowerShell 里 git 输出显示红色"错误"

| 问题 | PowerShell 把 git 写入 stderr 的信息标红，看起来像报错 |
|------|------|
| 结论 | 看具体文字判断：`main -> main`、`up to date` 等就是成功 |

### 坑 4：中文文件名乱码

| 问题 | `git status` 显示 `\344\275\277\347\224\250...` |
|------|------|
| 原因 | 终端编码显示问题 |
| 结论 | 实际文件名正常，提交到 GitHub 网页上显示中文 |

---

## 日常更新三板斧

以后每次修改笔记，只需要三步：

```bash
git add .
git commit -m "修改说明"
git push
```

> `git add .` 全量暂存 → `git commit` 固化版本 → `git push` 上传远程。详见 [[2-Git-核心命令与提交]] 与 [[4-Git-远程协作与PR]]。

---

## 相关笔记

- 安装与概念 → [[1-Git-安装与入门]]
- add/commit 原理 → [[2-Git-核心命令与提交]]
- 远程协作与 PR → [[4-Git-远程协作与PR]]
- 提交规范与常见坑 → [[8-Git-最佳实践与FAQ]]
