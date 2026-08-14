# 2. 日常操作

> 覆盖每天都会用到的 SVN 命令：添加、提交、更新、状态查看、diff 审查、switch 切换。

---

## 目录

1. [工作流总览](#工作流总览)
2. [status 查看状态](#status-查看状态)
3. [add 添加文件](#add-添加文件)
4. [commit 提交](#commit-提交)
5. [update 更新](#update-更新)
6. [diff 查看改动](#diff-查看改动)
7. [delete 与 move](#delete-与-move)
8. [switch 切换目录](#switch-切换目录)
9. [忽略文件](#忽略文件)
10. [常见问题](#常见问题)

---

## 工作流总览

日常循环：

```bash
svn update                 # 1. 拉取别人最新提交
# ……修改文件……
svn status                 # 2. 看自己改了什么
svn diff                   # 3. 审查改动
svn add 新文件             # 4. 新文件加入跟踪
svn commit -m "说明"       # 5. 提交
```

> 类比 Git：`update`=pull，`add`+`commit`=add+commit+push。SVN 的 commit 直接上服务器，**没有本地暂存区**。

---

## status 查看状态

```bash
svn status                 # 工作副本状态
svn status -u              # 对比服务器（显示远程版本号）
svn status --show-updates -v   # 详细信息
```

状态字母含义：

| 字母 | 含义 | 类似 Git |
|------|------|---------|
| `A` | 已 add，待提交 | staged new |
| `M` | 已修改 | modified |
| `D` | 已删除 | deleted |
| `?` | 未纳入版本控制 | untracked |
| `!` | 文件缺失（手动删了没 svn delete） | — |
| `C` | 冲突 | conflict |
| `X` | 外部引用（externals） | — |

> 没有输出 = 干净。`svn status` 不带参数只显示本地改动，`-u` 才连服务器。

---

## add 添加文件

```bash
svn add newfile.txt        # 添加单个文件
svn add src/               # 递归添加整个目录
svn add *.c                # 通配符批量
svn add --force src/       # 对已忽略的文件强制添加

# 误 add 了？取消（保留文件）
svn revert newfile.txt
```

> ⚠️ 新文件必须 `add` 后再 `commit`，否则不被提交。目录本身不需要手动 add，SVN 会随内部文件自动加入。

---

## commit 提交

```bash
svn commit -m "提交说明"
svn ci -m "说明"           # 简写

# 只提交部分文件
svn commit -m "改 login 逻辑" src/login.c

# 提交指定版本（少用）
svn commit --revprop  ...
```

**提交说明规范**（与 Git 提交规范同理，见 [[8-Git-最佳实践与FAQ]]）：

```
svn commit -m "fix: 修复登录超时问题

- 增加 60 秒超时重试
- 补充单元测试"
```

> ⚠️ SVN 提交**不可撤回**（不像 Git 可 reset/revert）。提交前务必 `svn diff` 审查，确认无误再 commit。

---

## update 更新

```bash
svn update                 # 更新整个工作副本
svn update src/login.c     # 只更新某文件
svn update -r 150          # 更新到指定版本（checkout 指定版本）
```

update 输出字母含义（在文件前显示）：

| 字母 | 含义 |
|------|------|
| `U` | 已更新（Updated） |
| `A` | 新增（Added） |
| `D` | 删除（Deleted） |
| `C` | 冲突（Conflict，见 [[4-SVN-冲突解决与历史]]） |
| `G` | 合并成功（自动 merge） |

> `C` 冲突不会阻止 update，但工作副本处于冲突态，必须解决后才能 commit（见 [[4-SVN-冲突解决与历史]]）。

---

## diff 查看改动

```bash
svn diff                   # 查看所有本地改动
svn diff src/login.c       # 单文件
svn diff -r 100:120        # 两个版本之间差异
svn diff -c 130            # 某次提交改了啥

# 比较工作副本与指定版本
svn diff -r 120 src/login.c
```

> 与 Git 一样，`svn diff` 输出统一 diff 格式，可用 `svn diff | less` 分页查看。**提交前的必备审查步骤**。

---

## delete 与 move

```bash
# 删除（记录进版本控制）
svn delete old.c
svn rm old.c               # 简写
# 提交后服务器也删除
svn ci -m "移除旧模块"

# 移动/重命名（SVN 用 copy+delete 模拟）
svn move a.c src/b.c
svn mv a.c b.c             # 简写
svn ci -m "重命名文件"
```

> 不要直接 `rm` 文件后提交——要用 `svn delete`，否则状态显示 `!` 且提交报错。正确流程是 `svn delete` → `svn commit`。

---

## switch 切换目录

在 trunk 与分支之间切换（类似 Git 的 checkout 切分支，但 SVN 是换 URL）：

```bash
svn switch https://server/repos/project/branches/feature-x
# 简写
svn sw URL

# 切换并保留未提交的本地修改（会尝试合并）
svn switch URL --force

# 切回主干
svn switch https://server/repos/project/trunk
```

> `switch` 只换目录内容，不换本地未提交的修改（尝试自动合并）。**切换前先 commit 或 revert 干净**，避免把修改带到别的分支。

---

## 忽略文件

让编译产物、临时文件不被 `svn status` 打扰：

```bash
# 本地忽略（推荐，仅当前工作副本）
svn propset svn:ignore "*.o" .
svn propset svn:ignore -F .svnignore .
# 递归（--recursive 简写 -R）
svn propset -R svn:ignore "build" .

# 查看忽略规则
svn propget svn:ignore .
svn proplist .             # 列出该路径所有属性
```

`.svnignore` 内容示例：

```
*.o
*.exe
build/
logs/
```

> SVN 的忽略靠 `svn:ignore` **属性**管理（区别于 Git 的 `.gitignore` 文件），且只对目录起效。提交后全组生效的是**版本化属性**（见 [[5-SVN-进阶与迁移]]）。

---

## 常见问题

**Q：`svn: E200009: Could not display info for all targets`？**
A：部分文件不在版本控制下（如 `?` 状态文件）。逐个确认或先 `svn add`。

**Q：修改的文件不想提交了？**
A：`svn revert 文件` 撤销本地修改（无法恢复，务必先确认）。已提交的看 [[4-SVN-冲突解决与历史]] 的 merge 回退。

**Q：`svn status` 显示文件是 `!`？**
A：文件被外部工具删了。`svn delete --force 文件` 或用 `svn update` 恢复。

**Q：想查看服务器上有但本地没有的文件？**
A：`svn status -u` 会列出远程新增（前缀 `*`）。

---

## 下一步

- 分支与标签 → [[3-SVN-分支与标签]]
- 冲突解决与历史 → [[4-SVN-冲突解决与历史]]
- 与 Git 对照复习 → [[0-Git 使用说明]]
- 完整手册导航 → [[0-SVN 使用指南]]
