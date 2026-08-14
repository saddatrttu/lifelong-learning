# 4. 冲突解决与历史

> 多人协作不可避免遇到冲突。掌握冲突的识别、手动解决、`svn resolve` 收尾，以及 log/blame/revert 等历史操作。

---

## 目录

1. [冲突的产生](#冲突的产生)
2. [识别冲突](#识别冲突)
3. [手动解决冲突](#手动解决冲突)
4. [svn resolve 收尾](#svn-resolve-收尾)
5. [merge 分支合并](#merge-分支合并)
6. [log 查看历史](#log-查看历史)
7. [blame 追溯](#blame-追溯)
8. [revert 撤销](#revert-撤销)
9. [回退已提交的改动](#回退已提交的改动)
10. [常见问题](#常见问题)

---

## 冲突的产生

两个人修改了**同一文件的同一区域**，后 update/commit 的一方就会冲突：

```
你改了 login.c 的 50 行
同事也改了 login.c 的 50 行并已提交
你执行 svn update 或 svn commit  → 冲突
```

冲突文件会产生三个额外文件：

```
login.c          # 冲突后的文件（含 <<<<<<< 标记）
login.c.mine     # 你修改前的版本
login.c.r140     # 冲突时的基线版本
login.c.r141     # 服务器上的新版本
```

---

## 识别冲突

```bash
svn status        # 冲突文件显示 C
# C       login.c

svn update        # update 时报 Conflict
#   C    login.c
#   Conflict discovered in 'login.c'
```

`svn status -u` 的 `*` 表示服务器有更新，结合本地状态可预判冲突风险。

---

## 手动解决冲突

### 方式一：直接编辑（最常用）

打开冲突文件，看到冲突标记：

```
<<<<<<< .mine
你写的代码
=======
同事写的代码
>>>>>>> .r141
```

手动取舍两段内容，删除 `<<<<<<<`、`=======`、`>>>>>>>` 三行标记，保留正确代码。

### 方式二：命令辅助

```bash
svn resolve --accept=mine-conflict login.c
# 保留我的修改（丢弃对方，谨慎）

svn resolve --accept=theirs-conflict login.c
# 保留对方的修改（丢弃我的，谨慎）

svn resolve --accept=working login.c
# 保留当前已编辑好的文件内容
```

| accept 参数 | 行为 |
|-------------|------|
| `working` | 用当前编辑过的文件（推荐，手动解决后用它） |
| `mine-conflict` | 用我的版本 |
| `theirs-conflict` | 用服务器版本 |
| `mine-full` / `theirs-full` | 整文件采用（不按冲突块） |

> ⚠️ 除非明确要"全用我的/全用对方的"，否则手动编辑后 `--accept=working` 最安全。`mine-full`/`theirs-full` 会丢弃另一方的**所有**改动，包括不冲突的部分。

---

## svn resolve 收尾

解决冲突后的固定收尾动作（不加会一直显示 C，无法 commit）：

```bash
# 1. 手动编辑冲突文件后
svn resolve --accept=working login.c

# 2. 确认无残留冲突
svn status          # login.c 应显示 M

# 3. 提交
svn commit -m "解决 login.c 冲突"
```

> **铁律**：`resolve` 之后才能 `commit`。没 resolve 就提交会被拒绝。

---

## merge 分支合并

```bash
# 合并整个分支到当前工作副本
svn merge https://server/repos/project/branches/feature-login

# 合并指定版本范围（增量合并）
svn merge -r 100:150 https://server/repos/project/branches/feature-login

# 合并后
svn status          # 查看被改动的文件
svn diff            # 审查合并结果
svn commit -m "合并 feature-login 到 trunk"
```

**merge 冲突**处理与普通冲突完全一样：编辑 → `resolve` → `commit`。

> SVN 不会自动记录"这已合并过"。重复合并同一分支会重复应用，可能产生伪冲突。完整分支合并用 `--reintegrate`：`svn merge --reintegrate 分支URL`（会打底标记，之后该分支应删除重建）。

---

## log 查看历史

```bash
svn log                  # 完整历史
svn log -l 10            # 最近 10 条
svn log src/login.c      # 单文件历史
svn log -r 100:150       # 版本范围
svn log -v               # 详细（显示每版改了哪些路径）
svn log --xml            # XML 格式（脚本处理用）
```

输出结构（注意是**全局版本号**）：

```
------------------------------------------------------------------------
r151 | huan | 2026-08-11 15:20:00 +0800 (Tue, 11 Aug 2026) | 2 lines
修复登录超时问题
------------------------------------------------------------------------
```

> `svn log` 里看的是 `rNNN` 全局版本号，不是文件的版本号。想看某文件的版本历史用 `svn log 文件路径`。

---

## blame 追溯

```bash
svn blame src/login.c      # 每行是谁在哪个版本写的
# r120  huan  优化登录逻辑
# r135  mary  修复空指针
# r151  huan  修复超时
```

> 排查"这行代码谁加的、为什么"用 blame，配合 `svn log -r 135 -v` 看那次提交改了什么。

---

## revert 撤销

```bash
# 撤销本地未提交的修改（无法恢复！）
svn revert login.c
svn revert -R src/        # 递归撤销整个目录

# 撤销 add（未提交的新文件）
svn revert newfile.txt
```

> `revert` 只作用于**未提交**的修改。已提交的要靠反向 merge（见下）。

---

## 回退已提交的改动

SVN 没有 `git revert` 的一键命令，用 **反向 merge** 实现：

```bash
# 撤销 r130 这版提交（在最新版本上反向应用）
svn merge -c -130 https://server/repos/project/trunk
#                  ↑ 负号 = 反向合并（相当于撤销）

# 撤销版本范围
svn merge -r 150:130 https://server/repos/project/trunk
#              ↑ 前>后 = 反向

# 审查并提交
svn diff
svn commit -m "回退 r130 的改动"
```

> 撤销已发布到服务器的改动：**反向 merge 后在 trunk 上产生一条新提交**，历史保留（与 Git 的 `git revert` 思路一致，但更繁琐）。

---

## 常见问题

**Q：`svn: E200009` 提交被拒，提示有冲突？**
A：有文件还是 `C` 状态。`svn status` 找冲突文件，`resolve` 解决后再提交。

**Q：`merge` 后出现 `Tree conflict`？**
A：文件被删除/移动与修改重叠。看 `svn status` 的树冲突，手动保留一边，`svn resolve --accept=working`。

**Q：反向 merge 错版了？**
A：`svn revert` 撤销本地未提交的 merge 结果，再重新反向合并正确版本。

**Q：`svn log` 太慢/太大？**
A：`-l N` 限制条数，`svn log -r 100:120` 缩小范围，或对单文件查历史。

---

## 下一步

- 进阶（hooks/属性/externals/迁移 Git） → [[5-SVN-进阶与迁移]]
- Git 的回滚与历史对照 → [[5-Git-历史与回滚]]
- 分支与标签 → [[3-SVN-分支与标签]]
- 完整手册导航 → [[0-SVN 使用指南]]
