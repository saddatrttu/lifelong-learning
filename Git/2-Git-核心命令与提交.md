# 2. 核心命令与提交

> 理解三区域模型，掌握 `add`/`commit`/`diff` 这些最高频的命令，以及写好提交信息的方法。

---

## 目录

1. [三区域模型](#三区域模型)
2. [git add：加入暂存区](#git-add加入暂存区)
3. [git commit：提交](#git-commit提交)
4. [git diff：查看变更](#git-diff查看变更)
5. [提交规范](#提交规范)
6. [查看历史](#查看历史)
7. [常见操作场景](#常见操作场景)

---

## 三区域模型

这是理解一切 Git 操作的基础：

```
工作区(Working Directory)
    │  git add            （把改动从工作区搬进暂存区）
    ▼
暂存区(Staging Area / Index)
    │  git commit         （把暂存区内容固化为一个提交）
    ▼
本地仓库(Repository / HEAD)
```

| 区域 | 是什么 | 你能看到 |
|------|--------|---------|
| 工作区 | 磁盘上真实文件 | `git status` 显示"Changes not staged" |
| 暂存区 | 即将提交的清单 | `git status` 显示"Changes to be committed" |
| 仓库 | 已保存的历史 | `git log` |

**核心心法**：`git add` 决定"这次提交包含什么"，`git commit` 决定"什么时候固化"。

---

## git add：加入暂存区

```bash
git add file.txt        # 暂存单个文件
git add src/            # 暂存整个目录
git add .               # 暂存当前目录全部改动（谨慎）
git add -p              # 交互式逐块暂存（精细控制）
git add -A              # 暂存所有变更（含删除）
```

### 撤销暂存

```bash
git restore --staged file.txt   # 从暂存区移出（不丢改动）
git reset HEAD file.txt         # 旧写法，等效
```

---

## git commit：提交

```bash
git commit -m "描述信息"          # 单行提交
git commit -am "描述"             # 跳过 add，直接提交所有已跟踪文件的改动
git commit --amend -m "新信息"    # 修改最近一次提交（见 [5-历史与回滚]）
```

### 提交要点

1. **提交要小**：一次提交只做一个逻辑变更，便于回滚和审查
2. **提交信息要清晰**：能解释"为什么"，而不仅是"改了什么"
3. **不要提交无关文件**：先 `git status` 确认再提交
4. **未暂存先检查**：`git diff` 确认无误再 add/commit

---

## git diff：查看变更

```bash
git diff                     # 工作区 vs 暂存区（未暂存的改动）
git diff --staged            # 暂存区 vs 上次提交（已暂存）
git diff --cached            # 同 --staged
git diff HEAD                # 工作区 vs 最新提交（全部未提交改动）
git diff main..feature       # 两个分支之间的差异
```

> 💡 `git diff --stat` 只看变更统计，`git diff --word-diff` 逐词显示更细腻。

---

## 提交规范

推荐的提交信息格式（Conventional Commits）：

```
<type>(<scope>): <subject>

<body>
```

常用 type：

| type | 含义 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 bug |
| `docs` | 文档 |
| `style` | 格式（不影响逻辑） |
| `refactor` | 重构（不改行为） |
| `test` | 测试 |
| `chore` | 杂项/构建 |
| `perf` | 性能优化 |
| `revert` | 回滚 |

示例：

```
feat(auth): 添加 OAuth 登录
fix(parser): 修复空字符串崩溃
docs(readme): 补充安装说明
```

> 统一的提交规范让 `git log` 可读、可自动生成 changelog、可做语义化版本发布。详见 [[8-Git-最佳实践与FAQ]]。

---

## 查看历史

```bash
git log                     # 完整历史
git log --oneline           # 一行一个提交
git log --oneline --graph   # 图形化展示分支
git log --oneline -10       # 最近 10 条
git log --author="名字"      # 按作者过滤
git log --since="1 week ago" # 按时间过滤
git log -p                  # 带每个提交的完整 diff
git log -- <file>           # 只看某个文件的历史
git show <commit>           # 查看某个提交的详情
```

---

## 常见操作场景

### 场景 1：改错了文件，想放弃修改

```bash
git restore file.txt        # 丢弃工作区改动（危险：不可恢复）
git restore --staged f.txt  # 只是不暂存，保留改动
```

### 场景 2：提交后发现漏了文件

```bash
git add missing.txt
git commit --amend --no-edit    # 补进上一个提交，不修改信息
```

### 场景 3：想撤销某次提交

```bash
git revert <commit>     # 生成反向提交，保留历史（推荐，见 [5-历史与回滚]）
git reset --hard <sha>  # 回到某个提交并丢弃其后所有（危险）
```

---

## 相关笔记

- 分支与合并 → [[3-Git-分支与合并]]
- 历史查看与回滚 → [[5-Git-历史与回滚]]
- 提交规范与工作流 → [[8-Git-最佳实践与FAQ]]、[[7-Git-工作流与团队协作]]
