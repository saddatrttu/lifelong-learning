# 5. 进阶与迁移

> 了解 SVN 的钩子（hooks）、属性（properties）、externals 引用，以及存量项目从 SVN 迁移到 Git 的完整方案。

---

## 目录

1. [钩子 hooks](#钩子-hooks)
2. [属性 properties](#属性-properties)
3. [externals 外部引用](#externals-外部引用)
4. [锁与二进制文件](#锁与二进制文件)
5. [备份与运维](#备份与运维)
6. [从 SVN 迁移到 Git](#从-svn-迁移到-git)
7. [常见问题](#常见问题)

---

## 钩子 hooks

SVN 服务器端的钩子 = 提交生命周期中自动触发的脚本（位于仓库的 `hooks/` 目录）：

| 钩子 | 触发时机 | 常见用途 |
|------|----------|----------|
| `pre-commit` | 提交**前**（可拒绝） | 检查提交说明格式、禁止提交某路径 |
| `post-commit` | 提交**后** | 发送邮件通知、触发 CI 构建 |
| `start-commit` | 提交开始前 | 权限控制、日志记录 |
| `pre-revprop-change` | 修改版本属性前 | 禁止修改提交说明 |

```bash
# 示例：pre-commit 检查提交说明非空（hooks/pre-commit）
#!/bin/sh
REPOS="$1"
TXN="$2"
SVNLOOK=/usr/bin/svnlook
LOGMSG=$($SVNLOOK log -t "$TXN" "$REPOS")
if [ -z "$LOGMSG" ]; then
    echo "提交说明不能为空" 1>&2
    exit 1
fi
exit 0
```

> hooks 是**服务器运维**概念，客户端开发者通常只感知其效果（如"提交被拒，提示说明格式不对"）。修改 hooks 需要服务器管理员权限。

---

## 属性 properties

SVN 的属性系统（`svn propset` / `propget`）实现 Git 里 `.gitignore`、`.gitattributes`、文件锁的功能：

### 常用属性

| 属性 | 作用 | 类似 Git |
|------|------|---------|
| `svn:ignore` | 忽略文件列表（只对目录） | `.gitignore` |
| `svn:eol-style` | 换行符统一（`native`/`LF`/`CRLF`） | `.gitattributes` eol |
| `svn:mime-type` | 文件类型（影响 diff 与合并） | 部分 `.gitattributes` |
| `svn:keywords` | 关键字替换（`$Id$`、`$Date$`） | 无直接对应 |
| `svn:executable` | 标记可执行 | `core.fileMode` |
| `svn:needs-lock` | 需先加锁才能修改 | 文件锁 |

```bash
# 设置文本文件换行符为 LF（跨平台团队必配）
svn propset svn:eol-style LF src/*.c
svn commit -m "设置换行符规范"

# 关键字替换
svn propset svn:keywords "Id Date" src/version.h
# 文件里写 $Id$，提交后自动替换为版本号等

# 查看
svn propget svn:eol-style src/login.c
svn proplist src/
```

> 二进制文件建议设置 `svn:mime-type application/octet-stream`，SVN 就不会对它做文本 diff/合并。图片、jar、Excel 都该设置。

---

## externals 外部引用

`svn:externals` 类似 Git 的 submodule（见 [[6-Git-高级技巧]]）：在仓库里引用其他仓库/其他路径的目录。

```bash
# 在 ext/ 目录挂载外部仓库
svn propset svn:externals "ext https://server/repos/libs/logger/trunk" .
svn update
# ext/ 目录下就是外部仓库的内容，可独立 update/commit
```

```bash
# 多行 externals（-F 从文件读）
svn propset svn:externals -F externals.txt .
# externals.txt:
# ext/log  https://server/repos/libs/logger/trunk
# ext/cfg  https://server/repos/configs/main
```

> externals 解决"多个项目共用公共库"的需求。缺点是版本管理弱（依赖服务器路径），且 `svn status` 显示 `X` 目录。现已逐步被 Git 的 submodule/go module/npm 取代。

---

## 锁与二进制文件

SVN 提供**文件锁**（Git 没有的内建功能）解决二进制文件不能合并的问题：

```bash
svn lock design.psd          # 加锁
svn commit -m "更新设计稿"   # 提交时自动解锁
svn unlock design.psd        # 手动解锁
svn status                   # 锁定文件显示 O/K 标记
```

```bash
# 目录级：新文件必须锁才能改
svn propset svn:needs-lock design.psd
```

> 锁适用于**无法合并的二进制文件**（PSD、Excel、视频）。文本文件应靠 merge 而非加锁。

---

## 备份与运维

```bash
# 冷备份：svnadmin dump（服务器操作）
svnadmin dump /path/to/repos > backup.dump

# 热备份：svnadmin hotcopy（在线备份）
svnadmin hotcopy /path/to/repos /backup/repos-copy

# 创建新仓库
svnadmin create /path/to/newrepos

# 校验仓库
svnadmin verify /path/to/repos
```

> 普通开发者不需运维仓库，但理解 dump/hotcopy 有助于应对"迁移仓库"需求。

---

## 从 SVN 迁移到 Git

存量 SVN 项目迁 Git 的完整方案（核心工具 `git svn`）：

```bash
# 1. 生成 authors 文件（SVN 用户名 → Git 姓名邮箱）
svn log -q | awk -F '|' '/^r/ {sub("^ ", "", $2); sub(" $", "", $2); print $2" = "$2" <"$2"@example.com>"}' \
  | sort -u > authors.txt
# 手动编辑 authors.txt 补全真实姓名邮箱

# 2. 克隆 SVN 仓库为 Git 仓库（保留全部历史与分支）
git svn clone --stdlayout --authors-file=authors.txt \
    https://server/repos/project/trunk project-git
# --stdlayout 自动识别 trunk/branches/tags 结构

# 3. 之后在 Git 仓库正常开发
git log
git branch -a          # SVN 的 branches 会变成远程分支

# 4. 推送到新的 Git 远程
git remote add origin git@github.com:user/project.git
git push -u origin --all
git push -u origin --tags
```

### 迁移决策要点

| 考虑 | 说明 |
|------|------|
| 是否保留历史 | `git svn` 可迁移全部提交、作者、时间 |
| 分支映射 | SVN 的 trunk→master，branches→远程分支，tags→Git 标签 |
| 迁移后 SVN | 冻结只读，或并行过渡期 |
| 替代方案 | `svn2git`（自动化更高）；大规模用 `git-svn` + 脚本 |

> `git svn` 只是迁移工具。迁移后**不要在 Git 里再 `git svn` 回写 SVN**，单向流动。SVN 与 Git 的模型差异对照见 [[1-SVN-安装与入门]]。

---

## 常见问题

**Q：`svn propset` 提交后别人没生效？**
A：`svn update` 后属性随文件一起同步；忽略属性只在目录上有效，确认作用对象是目录。

**Q：externals 目录 `svn status` 显示 `X` 很碍眼？**
A：`X` 是正常状态（外部引用）。用 `svn status --ignore-externals` 忽略显示。

**Q：提交被 pre-commit 钩子拒绝？**
A：看报错信息（钩子输出的提示），通常是要补提交说明、规范格式，按提示修改重提。

**Q：迁移后 tag 全跑到一个分支上了？**
A：`git svn` 的 tags 默认以 `refs/remotes/origin/tags/*` 导入。用 `git branch -r` 查看，再批量 `git tag` 转换，或迁移时加 `--tags` 参数处理。

---

## 下一步

- 与 Git 对照总览 → [[0-Git 使用说明]]
- Git 进阶技巧（迁移后的常用操作） → [[6-Git-高级技巧]]
- 构建与自动化（SVN 配套） → [[0-Linux 使用指南]]
- 完整手册导航 → [[0-SVN 使用指南]]
