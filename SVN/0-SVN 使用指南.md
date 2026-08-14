# SVN 使用指南

> SVN（Subversion）是经典的**集中式版本控制系统（CVCS）**：所有历史都存在中心服务器，客户端从服务器 checkout 工作副本，所有提交都直接写入服务器。与 Git 的分布式思路不同，但至今仍大量运行在传统企业项目中。
>
> 本文是一套完整的参考手册，采用 **MOC + 多笔记** 结构，与 [[0-Git 使用说明]]、[[0-Linux 使用指南]] 同一套整理思路。全程与 Git 对照，方便已会 Git 的人快速上手。

---

## 阅读路径

- **新手**：按顺序 [[1-SVN-安装与入门]] → [[2-SVN-日常操作]]
- **分支协作**：[[3-SVN-分支与标签]]
- **冲突与历史**：[[4-SVN-冲突解决与历史]]
- **进阶**：[[5-SVN-进阶与迁移]]
- **快速查表**：直接跳到目标笔记，每个笔记开头带目录锚点

---

## 目录

| # | 笔记 | 一句话内容 |
|---|------|-----------|
| 0 | [[0-SVN 使用指南\|本页 MOC]] | 导航入口 |
| 1 | [[1-SVN-安装与入门]] | 集中式模型、安装、checkout、与 Git 核心差异 |
| 2 | [[2-SVN-日常操作]] | add/commit/update/status/diff/switch |
| 3 | [[3-SVN-分支与标签]] | trunk/branches/tags 目录规范、svn copy |
| 4 | [[4-SVN-冲突解决与历史]] | 冲突处理、merge、log、blame、revert |
| 5 | [[5-SVN-进阶与迁移]] | hooks、属性、externals、从 SVN 迁移到 Git |

---

## 核心心智模型

**集中式 vs 分布式**是理解 SVN 的关键（对照 [[0-Git 使用说明]]）：

```
        SVN（集中式）                     Git（分布式）
      ┌─────────────┐                  ┌─────────────┐
      │  中心服务器  │                  │  本地仓库    │
      │  唯一权威历史 │                 │  (clone 全量)│
      └─────────────┘                  └──────┬──────┘
            ▲                                 │
     checkout/commit                    push/pull
            │                                 ▼
      ┌─────┴─────┐                  ┌─────────────┐
      │ 工作副本    │                  │ 远程仓库     │
      └───────────┘                  └─────────────┘
```

**SVN 的关键差异**：
- **无本地仓库**：离线不能提交，历史只在服务器
- **版本号是全局递增数字**：`r1`、`r2`…（Git 是哈希）
- **版本号 = 整个仓库**：`r10` 是整个仓库第 10 次提交，不是某个文件
- **分支 = 服务器上的目录拷贝**（`svn copy`），不是指针

---

## 快速参考

### 最常用的 10 个命令

| 命令 | 作用 | 大致对应 Git |
|------|------|-------------|
| `svn checkout <url>` | 检出工作副本 | `git clone` |
| `svn update` | 更新到最新 | `git pull` |
| `svn add <file>` | 添加文件 | `git add` |
| `svn commit -m "msg"` | 提交到服务器 | `git commit` + `push` |
| `svn status` | 查看状态 | `git status` |
| `svn diff` | 查看改动 | `git diff` |
| `svn log` | 查看历史 | `git log` |
| `svn revert <file>` | 撤销本地修改 | `git checkout -- file` |
| `svn copy <src> <dst>` | 分支/标签 | `git branch` / `git tag` |
| `svn merge <src>` | 合并分支 | `git merge` |

---

## 关联指南

- [[0-Git 使用说明]] — 分布式版本控制，与 SVN 对照学习效果最佳
- [[0-Linux 使用指南]] — SVN 命令行在 Linux 上运行，常配合 Makefile 构建
- [[0-Claude Code 使用指南]] — 自动化脚本常需兼容 SVN/Git 两套命令
- 官方文档：https://svnbook.red-bean.com/（Version Control with Subversion，免费电子书）
