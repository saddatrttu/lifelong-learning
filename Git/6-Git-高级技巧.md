# 6. 高级技巧

> 让 Git 效率倍增的进阶工具：暂存工作现场（stash）、精选提交（cherry-pick）、子模块（submodule）、二分定位（bisect）与标签（tag）。

---

## 目录

1. [git stash：暂存工作现场](#git-stash暂存工作现场)
2. [git cherry-pick：精选提交](#git-cherry-pick精选提交)
3. [git tag：标签](#git-tag标签)
4. [git bisect：二分定位](#git-bisect二分定位)
5. [子模块与子树](#子模块与子树)
6. [worktree 并行工作区](#worktree-并行工作区)
7. [大文件处理](#大文件处理)
8. [性能与维护](#性能与维护)

---

## git stash：暂存工作现场

把未提交的改动**暂时存起来**，工作区变干净，稍后再取回：

```bash
git stash                    # 暂存所有未提交改动
git stash -u                 # 连未跟踪文件一起存（-u = --include-untracked）
git stash list               # 查看暂存列表
git stash pop                # 取回最近一个并删除记录
git stash apply              # 取回但保留记录（可多次应用）
git stash drop stash@{1}     # 删除指定暂存
git stash clear              # 清空全部
```

**典型场景**：正改到一半，突然需要切分支干别的事。

---

## git cherry-pick：精选提交

把**某次提交**（不一定是本分支的）复制到当前分支：

```bash
git cherry-pick <commit>          # 应用该提交
git cherry-pick A..B              # 应用一段范围
git cherry-pick --no-commit <c>   # 只改文件不提交
git cherry-pick --abort           # 放弃，回到之前
```

**典型场景**：
- 把 hotfix 单独摘到其他发布分支
- 把一个分支的某个提交应用到当前分支（不合并整条分支）

---

## git tag：标签

给特定提交打"里程碑"标记（通常用于发布版本）：

```bash
git tag                    # 列出标签
git tag v1.0.0             # 创建轻量标签
git tag -a v1.0.0 -m "版本1.0"   # 创建带注释标签（推荐）
git tag v1.0.0 <commit>    # 给历史提交打标签
git push origin v1.0.0     # 推送标签
git push origin --tags     # 推送所有标签
git tag -d v1.0.0          # 删除本地标签
git checkout v1.0.0        # 检出到该版本
```

> 💡 语义化版本规范 `MAJOR.MINOR.PATCH`（如 `v2.1.3`）配提交规范可自动生成 changelog。

---

## git bisect：二分定位

用**二分搜索**快速定位"哪个提交引入了 bug"：

```bash
git bisect start
git bisect bad                 # 当前版本是坏的
git bisect good <sha>          # 标记一个已知正常的提交
# 每次 Git 检出中间提交，你运行测试并标记：
git bisect good | bad          # 反复标记，直到定位
git bisect reset               # 结束，回到原分支
```

**自动化版本**（配合测试命令）：

```bash
git bisect run npm test        # 自动跑测试判断好坏
```

> 💡 先找到 bug 引入的提交，再 `git show <sha>` 看改动，往往比猜更快。这是 [[5-Git-历史与回滚]] 的进阶配合。

---

## 子模块与子树

### 子模块（submodule）

在仓库中引用**另一个 Git 仓库**（保持独立历史）：

```bash
git submodule add <url> path/to/sub
git clone --recurse-submodules <url>   # 克隆时一并拉取子模块
git submodule update --init --recursive
git submodule foreach git pull         # 更新所有子模块
```

> ⚠️ 子模块有"指针"复杂性（子模块自身也有历史），团队需约定统一更新方式。能用包管理（npm/pip）解决的优先用包管理。

### 子树（subtree）

把另一个仓库的内容**合并进当前仓库**（无指针概念，历史内联）：

```bash
git subtree add --prefix=libs/foo <url> main
git subtree pull --prefix=libs/foo <url> main
```

---

## worktree 并行工作区

同一个仓库挂载**多个工作目录**，互不干扰地并行处理多分支：

```bash
git worktree add ../feature-a feature-a
git worktree list
git worktree remove ../feature-a
```

**典型场景**：main 有 hotfix 要立刻处理，同时 feature 分支正在开发中——不需要 stash，直接开新 worktree。

> 💡 AI 编程工具（如 [[0-Claude Code 使用指南]]）常用 worktree 隔离并行任务。

---

## 大文件处理

### 大文件不适合进 Git

- Git 会永久保存每个版本的快照，大文件会让仓库膨胀
- 超过 100MB 的文件应使用 LFS 或外部存储

### Git LFS（Large File Storage）

```bash
git lfs install
git lfs track "*.psd" "*.zip"
git add .gitattributes
git lfs ls-files          # 查看 LFS 管理文件
```

---

## 性能与维护

```bash
git gc                        # 垃圾回收，压缩对象库
git gc --aggressive --prune=now   # 彻底压缩（慎用）
git count-objects -vH         # 查看仓库体积
git repack -a -d              # 打包优化
```

**仓库变大的自查**：
- 检查是否有大文件混入历史（`git rev-list --objects --all | ...`）
- 历史清理用 `git filter-repo`（取代已过时的 `filter-branch`）

---

## 相关笔记

- 历史与回滚 → [[5-Git-历史与回滚]]
- 分支与合并 → [[3-Git-分支与合并]]
- 团队工作流 → [[7-Git-工作流与团队协作]]
- 仓库规范 → [[8-Git-最佳实践与FAQ]]
