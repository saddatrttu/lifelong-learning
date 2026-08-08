# 5. 历史与回滚

> Git 几乎能找回一切——关键在于先理解**从哪里找回**。本笔记覆盖历史查看（log/blame）、撤销（reset/revert）与"后悔药"（reflog）。

---

## 目录

1. [查看历史](#查看历史)
2. [查看文件来源：git blame](#查看文件来源git-blame)
3. [搜索历史](#搜索历史)
4. [撤销工作区/暂存区改动](#撤销工作区暂存区改动)
5. [撤销提交：reset vs revert](#撤销提交reset-vs-revert)
6. [改写历史：amend / rebase -i](#改写历史amend--rebase--i)
7. [最后的救星：reflog](#最后的救星reflog)
8. [决策速查表](#决策速查表)

---

## 查看历史

```bash
git log --oneline                  # 简洁历史
git log --oneline --graph --all    # 分支图
git log --stat                     # 每次提交改了哪些文件
git log -p                         # 完整 diff
git log --follow -- file.txt       # 跟踪文件重命名
git show <commit>                  # 查看单个提交
git show <commit>:path/to/file     # 查看某提交时的文件内容
```

---

## 查看文件来源：git blame

逐行查看"每一行是谁、在哪个提交里写的"：

```bash
git blame src/app.ts
git blame -L 100,120 src/app.ts    # 只看 100-120 行
git blame -w src/app.ts            # 忽略空白变化
```

> 💡 配合 `git log -p <file>` 能找到该行后续的修改。定位"哪次提交引入的 bug"用它最高效。

---

## 搜索历史

```bash
git log -S "关键词"            # 找新增/删除该关键词的提交（pickaxe）
git log -G "regex"            # 按正则搜索变更
git log --grep="bug"          # 按提交信息搜索
git grep "关键词" <commit>      # 在某提交的代码里搜
```

---

## 撤销工作区/暂存区改动

| 目标 | 命令 | 效果 |
|------|------|------|
| 丢弃工作区改动 | `git restore <file>` | 回到暂存区/HEAD 状态 |
| 撤出暂存区 | `git restore --staged <file>` | 保留改动，仅取消暂存 |
| 放弃全部未提交 | `git checkout -- .` | 危险，不可恢复 |
| 清理未跟踪文件 | `git clean -fd` | 删除未跟踪文件（危险） |

> ⚠️ `git restore` 丢弃的改动**不可找回**（除非 IDE 缓存）。删除前先确认或备份。

---

## 撤销提交：reset vs revert

### git reset（移动指针，改写历史）

```bash
git reset --soft HEAD~1      # 撤销提交但保留改动在暂存区
git reset --mixed HEAD~1     # 撤销提交，改动回工作区（默认）
git reset --hard HEAD~1      # 撤销提交并丢弃改动（危险）
git reset --hard <sha>       # 回到任意历史提交
```

三种模式对比：

| 模式 | 提交 | 暂存区 | 工作区 |
|------|------|--------|--------|
| `--soft` | 撤销 | 保留 | 保留 |
| `--mixed`（默认） | 撤销 | 清空 | 保留 |
| `--hard` | 撤销 | 清空 | 清空 |

### git revert（生成反向提交，保留历史）

```bash
git revert <commit>      # 生成一个"反向提交"抵消该提交
git revert -n <commit>   # 不自动提交，可连续 revert 多个
```

### 什么时候用哪个

| 场景 | 用哪个 |
|------|--------|
| 提交还没推送（本地） | `reset`（随意改） |
| 提交已推送到共享分支 | **`revert`**（不重写他人历史） |
| 只想放弃提交，保留改动 | `reset --soft` / `--mixed` |
| 想完全抹掉一段历史 | 本地 `reset --hard`；共享分支禁止 |

> ⚠️ **共享分支绝不 reset/rebase 已推送提交**——会让他人仓库陷入混乱。共享分支的撤销一律用 `revert`。

---

## 改写历史：amend / rebase -i

### 修改最近一次提交

```bash
git commit --amend -m "新信息"       # 改提交信息
git add .; git commit --amend --no-edit   # 补文件，不改信息
```

> ⚠️ `--amend` 会改变 commit hash。仅用于**未推送**的提交。

### 修改多个提交（交互式变基）

```bash
git rebase -i HEAD~3
```

常用指令：`pick`/`reword`/`edit`/`squash`/`drop`。改写后历史 hash 全部变化，**只用于本地未推送**的历史。

---

## 最后的救星：reflog

`reflog` 记录 HEAD 的**每一次移动**——包括 reset、rebase、删除分支。它是最后的后悔药。

```bash
git reflog                 # 查看 HEAD 移动记录
git reflog --oneline
git reset --hard HEAD@{2}  # 回到两小时前的状态
git branch recover <sha>   # 从 reflog 找回被删的分支
```

> 💡 **reflog 只在本地有效**，且不会永久保留（默认 90 天）。所以：已推送的远程数据靠远程仓库恢复；本地被删的数据靠 reflog 恢复；想彻底抹除需 `git gc` + 远程强制覆盖。

---

## 决策速查表

遇到"改错了"，按顺序问自己：

| 问题 | 答案 |
|------|------|
| 改动还在工作区/暂存区？ | `git restore` / `git restore --staged` |
| 提交还没推送？ | `git reset` 或 `git commit --amend` |
| 提交已推送、是共享分支？ | `git revert` |
| 一切都被删了找不到？ | `git reflog` |
| 只想找到"哪行谁写的"？ | `git blame` |
| 想找"哪个提交引入的 bug"？ | `git log -S 关键词` |

---

## 相关笔记

- 提交与 diff 基础 → [[2-Git-核心命令与提交]]
- 分支合并/变基 → [[3-Git-分支与合并]]
- 远程协作注意事项 → [[4-Git-远程协作与PR]]
- 安全提交规范 → [[8-Git-最佳实践与FAQ]]
