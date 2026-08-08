# Git 使用说明

> Git 是目前最流行的**分布式版本控制系统（DVCS）**。它记录代码的每次变更、支持并行分支开发、协调多人协作，是软件工程的基本功。
>
> 本文是一套完整的参考手册，采用 **MOC + 多笔记** 结构，与 [[0-Claude Code 使用指南]] 同一套整理思路。

---

## 阅读路径

- **新手**：按顺序 [[1-Git-安装与入门]] → [[2-Git-核心命令与提交]] → [[3-Git-分支与合并]] → [[4-Git-远程协作与PR]] → [[8-Git-最佳实践与FAQ]]
- **进阶**：[[5-Git-历史与回滚]] → [[6-Git-高级技巧]] → [[7-Git-工作流与团队协作]]
- **快速查表**：直接跳到目标笔记，每个笔记开头带目录锚点

---

## 目录

| # | 笔记 | 一句话内容 |
|---|------|-----------|
| 0 | [[0-Git 使用说明\|本页 MOC]] | 导航入口 |
| 1 | [[1-Git-安装与入门]] | 安装、配置、仓库概念、首次提交 |
| 2 | [[2-Git-核心命令与提交]] | add/commit、暂存区、提交规范、diff |
| 3 | [[3-Git-分支与合并]] | 分支、merge/rebase、冲突解决 |
| 4 | [[4-Git-远程协作与PR]] | remote、push/pull、fork、PR、评审 |
| 5 | [[5-Git-历史与回滚]] | log/blame、reset/revert、reflog |
| 6 | [[6-Git-高级技巧]] | stash、cherry-pick、submodule、bisect |
| 7 | [[7-Git-工作流与团队协作]] | GitFlow、GitHub Flow、分支策略 |
| 8 | [[8-Git-最佳实践与FAQ]] | 提交规范、.gitignore、安全、常见坑 |

---

## 核心心智模型

**三个区域**是理解 Git 的关键（详见 [[2-Git-核心命令与提交]]）：

```
工作区(Working Directory)
    │  git add
    ▼
暂存区(Staging Area / Index)
    │  git commit
    ▼
本地仓库(Repository / HEAD)
    │  git push
    ▼
远程仓库(Remote)
```

**Git 是什么（一句话）**：Git 不是保存文件"快照"的差异记录器，而是保存**完整快照流** + 指向它们的指针（commit hash）。分支只是指向某个提交的可移动指针。

---

## 快速参考

### 最常用的 10 个命令

| 命令 | 作用 |
|------|------|
| `git init` | 初始化仓库 |
| `git clone <url>` | 克隆远程仓库 |
| `git status` | 查看当前状态 |
| `git add <file>` | 加入暂存区 |
| `git commit -m "msg"` | 提交 |
| `git pull` | 拉取远程更新 |
| `git push` | 推送到远程 |
| `git branch` | 分支管理 |
| `git merge <branch>` | 合并分支 |
| `git log --oneline` | 查看提交历史 |

---

## 参考资源

- 官方文档：https://git-scm.com/doc
- 交互式学习：https://learngitbranching.js.org
- Pro Git 中文版：https://git-scm.com/book/zh/v2
- 相关：[[0-Claude Code 使用指南]] 中的 Git 自动化工作流（见其 [[2-CC-核心命令与交互]]）
