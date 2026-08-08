# 4. 远程协作与 PR

> 把本地仓库连上远程（GitHub/GitLab/Gitee），学会 push/pull/fetch、fork 协作模式和 Pull Request 流程。

---

## 目录

1. [远程仓库的概念](#远程仓库的概念)
2. [remote 管理](#remote-管理)
3. [git push 推送](#git-push-推送)
4. [git pull 与 git fetch](#git-pull-与-git-fetch)
5. [认证方式](#认证方式)
6. [fork 协作模式](#fork-协作模式)
7. [Pull Request / Merge Request](#pull-request--merge-request)
8. [代码评审](#代码评审)
9. [常见场景](#常见场景)

---

## 远程仓库的概念

- **远程（remote）**：服务器上仓库的别名，默认叫 `origin`
- **跟踪分支**：`origin/main` 表示你本地缓存的远程状态
- 本地与远程通过 push/pull/fetch 同步

```
本地 ──push──▶ 远程(origin)
本地 ◀─pull─── 远程(origin)
本地 ◀─fetch── 远程(origin)   # 只下载不合并
```

---

## remote 管理

```bash
git remote -v                     # 查看远程地址
git remote add origin <url>       # 添加远程
git remote set-url origin <url>   # 修改远程地址
git remote rename origin upstream # 重命名
git remote remove origin          # 删除远程
```

---

## git push 推送

```bash
git push                      # 推送当前分支到同名远程分支
git push origin main          # 推送 main 到 origin
git push -u origin feature    # 推送并设置上游（首次推送推荐 -u）
git push origin --delete feat # 删除远程分支
git push --tags               # 推送标签
```

> 💡 `-u`（--set-upstream）会记住"这个本地分支对应哪个远程分支"，之后直接 `git push` 即可。

---

## git pull 与 git fetch

```bash
git fetch                    # 只下载远程更新，不改变工作区
git pull                     # fetch + merge
git pull --rebase            # fetch + rebase（推荐，避免多余合并提交）
git pull origin main         # 从指定分支拉取
```

**fetch vs pull**：

| 命令 | 下载 | 合并/改动工作区 | 适用 |
|------|------|----------------|------|
| `fetch` | ✅ | ❌（只更新 origin/* 指针） | 先看看远程有什么 |
| `pull` | ✅ | ✅（merge 或 rebase） | 直接更新到最新 |

> ⚠️ `git pull` 默认 merge 会产生额外合并提交；多人共享分支建议用 `git pull --rebase`，历史更干净。共享分支禁止 rebase 的规定见 [[3-Git-分支与合并]]。

---

## 认证方式

### HTTPS + Token

```bash
git clone https://github.com/user/repo.git
# 推送时输入用户名 + Personal Access Token（PAT）
```

### SSH

```bash
# 生成密钥
ssh-keygen -t ed25519 -C "you@example.com"

# 把 ~/.ssh/id_ed25519.pub 内容加到 GitHub/GitLab 设置
git clone git@github.com:user/repo.git
```

### 凭据管理器

- Windows：Git Credential Manager（随 Git for Windows 安装）
- 存储后无需重复输入

---

## fork 协作模式

适用于**开源贡献**或**无法直接 push** 的场景：

```
1. 在 GitHub 上 Fork 原仓库（得到自己的副本）
2. git clone 自己 fork 的仓库
3. 添加原仓库为 upstream 远程
4. 从 upstream 拉取更新，保持同步
5. 改完后 push 到自己的 fork
6. 发起 Pull Request 到原仓库
```

```bash
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git checkout main
git merge upstream/main          # 同步原仓库最新
git push origin main
```

---

## Pull Request / Merge Request

**Pull Request（GitHub）/ Merge Request（GitLab）** 是"请别人审查并合并我的分支"的请求。

### 流程

```
1. 从 main 建功能分支
2. 提交并推送：git push -u origin feature
3. 在平台上发起 PR（选择 base=main, compare=feature）
4. 写清 PR 描述（改动内容、测试方法、关联 issue）
5. 触发 CI（见 [7-工作流与团队协作]）
6. 等评审 → 修改 → 通过 → 合并
```

### 合并方式选择

| 方式 | 效果 |
|------|------|
| Create a merge commit | 保留合并提交 |
| Squash and merge | 所有提交压成一个（推荐，历史干净） |
| Rebase and merge | 线性历史 |

---

## 代码评审

评审是 PR 的核心环节：

**作者**：
- PR 越小越好，评审负担小
- 提供上下文：改动是什么、为什么、怎么测
- 及时回应评论，必要时补充提交

**评审者**：
- 关注设计、逻辑、边界、测试，而非格式（格式交给 CI/lint）
- 分级反馈：阻塞性问题（Blocking）> 建议（Suggestion）> 可选（Nit）
- 关注"为什么"而非"怎么改"

> 💡 高效评审工作流也可以借助 AI 工具（如 [[0-Claude Code 使用指南]] 的 `/review` 与子代理）。

---

## 常见场景

### 场景 1：push 被拒绝（远程有新提交）

```bash
git pull --rebase          # 先同步再推
git push
```

### 场景 2：改错分支想改

```bash
git branch -f main          # 未推送前可直接移动分支
```

### 场景 3：远程分支被删，本地还在

```bash
git remote prune origin     # 清理已删除的远程分支引用
```

---

## 相关笔记

- 分支合并基础 → [[3-Git-分支与合并]]
- 撤销与历史改写 → [[5-Git-历史与回滚]]
- 团队分支与 CI 策略 → [[7-Git-工作流与团队协作]]
- 协作规范 → [[8-Git-最佳实践与FAQ]]
