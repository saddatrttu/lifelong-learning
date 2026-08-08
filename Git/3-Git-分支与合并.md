# 3. 分支与合并

> 分支是 Git 最强大的特性——它是"指向某个提交的可移动指针"。掌握分支、merge/rebase 与冲突解决，是 Git 进阶的分水岭。

---

## 目录

1. [分支是什么](#分支是什么)
2. [分支基础操作](#分支基础操作)
3. [git merge 合并](#git-merge-合并)
4. [git rebase 变基](#git-rebase-变基)
5. [merge vs rebase](#merge-vs-rebase)
6. [冲突解决](#冲突解决)
7. [常见场景](#常见场景)

---

## 分支是什么

分支本质是**一个指向提交的指针**。创建分支极廉价，所以"一个功能一个分支"是标准做法。

```
main ── A ── B ── C ── D
               │
feature ────── E ── F
```

- `HEAD` 指向你当前所在的分支
- 分支移动 = 指针移动，不会复制文件
- 每个分支可以独立演化，最终通过合并汇合

> 💡 建议先到 https://learngitbranching.js.org 交互式体验分支，理解立竿见影。

---

## 分支基础操作

```bash
git branch                     # 列出本地分支（* 表示当前）
git branch -a                  # 列出全部（含远程）
git branch feature             # 创建分支
git switch feature             # 切换分支（推荐）
git checkout feature           # 切换分支（旧写法）
git switch -c feature          # 创建并切换（-c = create）
git checkout -b feature        # 同上（旧写法）
git branch -d feature          # 删除已合并分支
git branch -D feature          # 强制删除（未合并）
git branch -m newname          # 重命名当前分支
```

### 改名默认分支

```bash
git branch -m master main
```

---

## git merge 合并

把另一个分支的提交合入当前分支：

```bash
git switch main            # 先切到要接收的分支
git merge feature          # 把 feature 合入 main
```

### 三种合并结果

| 情况 | 结果 |
|------|------|
| 快进（fast-forward） | 线性推进，无合并提交 |
| 三方合并 | 生成一个 merge commit（可用 `--no-ff` 强制） |
| 冲突 | 需要手动解决 |

### 常用参数

```bash
git merge --no-ff feature    # 强制保留合并提交（利于保留"功能边界"）
git merge --squash feature   # 压成单次变更，但不自动提交
git merge --abort            # 放弃合并，回到合并前
```

---

## git rebase 变基

把当前分支的提交**重新放在**另一个分支顶端：

```bash
git switch feature
git rebase main       # 把 feature 的提交搬到 main 之后
```

```
变基前：
main ── A ── B
          │
feature ── C ── D

变基后：
main ── A ── B ── C' ── D'
```

**作用**：让历史变成干净直线，避免 merge commit 污染 log。

### 交互式变基

```bash
git rebase -i HEAD~3    # 修改最近 3 个提交
```

常用操作：`pick`（保留）、`squash`（合并进上一个）、`reword`（改信息）、`edit`（编辑）、`drop`（删除）。

> ⚠️ **铁律**：**永远不要 rebase 已经推送到远程并被人拉取的提交**——会改写他人已拥有的历史，造成灾难（见 [5-历史与回滚] 的改写历史）。

---

## merge vs rebase

| 维度 | merge | rebase |
|------|-------|--------|
| 历史 | 保留分叉与合并提交 | 线性化，干净 |
| 安全性 | 不重写历史 | **重写历史** |
| 场景 | 合并功能分支到 main、共享分支 | 本地整理、拉取远程更新前 |
| 冲突 | 一次解决 | 每个提交都要处理一次（可能多次） |
| 恢复 | 易恢复 | 改写后较难恢复（需 reflog） |

**实践建议**：
- **公共分支**（main/develop）：只用 merge
- **本地私有分支**：用 rebase 保持干净
- **拉取远程更新**：`git pull --rebase`（避免多余 merge commit）
- 团队统一约定优先于个人偏好（见 [[7-Git-工作流与团队协作]]）

---

## 冲突解决

当两处改动同一行时产生冲突，Git 会把冲突标记写进文件：

```
<<<<<<< HEAD
这里是当前分支的版本
=======
这里是合并进来分支的版本
>>>>>>> feature
```

### 解决步骤

1. **定位冲突文件**：`git status` 会列出 `both modified` 的文件
2. **编辑文件**：删除 `<<<<<<<`、`=======`、`>>>>>>>` 标记，保留正确的版本
3. **标记已解决**：`git add <file>`
4. **完成合并**：
   - merge 模式：`git commit`
   - rebase 模式：`git rebase --continue`

### 工具辅助

```bash
git mergetool        # 打开可视化合并工具
git config --global merge.conflictstyle diff3   # 显示更多冲突上下文
git config --global rerere.enabled true         # 记忆冲突解法，下次自动应用
```

> 💡 冲突不是错误，是正常协作的一部分。关键是**看全上下文**，别只留自己的版本。

---

## 常见场景

### 场景 1：切分支时未提交的改动

```bash
git stash                 # 先把改动暂存起来（见 [6-高级技巧]）
git switch feature
git stash pop             # 切完再取回
```

### 场景 2：main 有更新，想同步到自己的分支

```bash
git switch feature
git merge main            # 方式一：合并进来
git rebase main           # 方式二：变基到 main 之上
```

### 场景 3：删除了错误的分支想找回

```bash
git branch <name> <commit-sha>   # 用 reflog 找到 sha 再重建
```

---

## 相关笔记

- 推送到远程 → [[4-Git-远程协作与PR]]
- 撤销与改写历史 → [[5-Git-历史与回滚]]
- stash 与高级技巧 → [[6-Git-高级技巧]]
- 团队分支策略 → [[7-Git-工作流与团队协作]]
