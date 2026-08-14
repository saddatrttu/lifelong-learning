# 1. 安装与入门

> 认识集中式版本控制模型，安装 SVN，完成第一次 checkout 与 commit，并理清与 Git 的核心差异。

---

## 目录

1. [SVN 是什么](#svn-是什么)
2. [安装](#安装)
3. [集中式模型](#集中式模型)
4. [checkout 检出](#checkout-检出)
5. [首次提交](#首次提交)
6. [与 Git 的核心差异](#与-git-的核心差异)
7. [URL 规范](#url-规范)
8. [常见问题](#常见问题)

---

## SVN 是什么

Subversion（SVN）是 2000 年代主流的版本控制系统，采用**集中式**模型：所有版本历史存在中心服务器，开发者只保留"工作副本"。

如今多数新项目用 Git，但 SVN 仍在大量存量企业项目、旧系统中运行，作为开发者仍需会读会改。

---

## 安装

```bash
# Ubuntu/Debian
sudo apt install subversion

# CentOS/RHEL
sudo yum install subversion

# macOS
brew install subversion

# Windows
# 安装 TortoiseSVN（图形界面）或 VisualSVN / 命令行 svn
# 或 winget install TortoiseSVN.TortoiseSVN

# 验证
svn --version
```

---

## 集中式模型

**核心流程**：

```
         中心服务器（权威历史）
             │
   checkout（首次） / update（更新） / commit（提交）
             │
         工作副本（本地）
```

- **checkout**：首次从服务器拉取全部代码到本地
- **update**：把服务器上的新提交拉下来（别人改的）
- **commit**：把本地修改推上服务器（SVN 里提交 = Git 的 commit + push 合一）

> ⚠️ SVN 没有"本地提交"概念。所有 `commit` 直接进服务器，且**需要网络**。

---

## checkout 检出

```bash
# 检出仓库某路径到当前目录
svn checkout https://svn.example.com/repos/project/trunk
# 简写
svn co https://svn.example.com/repos/project/trunk

# 指定本地目录名
svn co https://svn.example.com/repos/project/trunk myproj

# 检出指定版本号
svn co -r 120 https://svn.example.com/repos/project/trunk

# 只检出子目录（节省时间，但功能受限）
svn co https://svn.example.com/repos/project/trunk/src
```

checkout 成功后，目录下出现隐藏的 `.svn/` 文件夹，记录工作副本元数据（相当于 Git 的 `.git/`）。

---

## 首次提交

```bash
# 1. 进入工作副本
cd myproj

# 2. 添加新文件（SVN 需要显式 add 才被跟踪）
svn add README.md
svn add src/            # 递归添加目录

# 3. 提交
svn commit -m "初始化项目"
# 简写
svn ci -m "初始化项目"

# 4. 查看日志
svn log -l 5
```

> 与 Git 不同：SVN 的 `add` 是**把文件纳入版本控制**（告诉服务器"我要跟踪它"），`commit` 才真正上传。新文件不 add 直接 commit 不会提交。

---

## 与 Git 的核心差异

| 维度 | SVN | Git |
|------|-----|-----|
| 模型 | 集中式 | 分布式 |
| 本地历史 | 无 | 有完整历史 |
| 离线提交 | 不可 | 可以 |
| 版本号 | 全局递增数字 `r120` | SHA-1 哈希 |
| 版本号范围 | 整个仓库 | 每次提交 |
| 分支 | 目录拷贝 | 指针 |
| 跟踪元数据 | 目录内 `.svn/` | 仓库根 `.git/` |
| 提交粒度 | 一个仓库一个版本号 | 每个文件可独立提交 |
| 撤销已发布提交 | 难（反向 merge） | `git revert` 容易 |

> 一句话：**Git 是"每个开发者一台完整服务器"；SVN 是"只有一台服务器，大家都连它"。** 详细分布式模型见 [[0-Git 使用说明]]。

---

## URL 规范

SVN 用 URL 定位仓库路径，常见结构（规范见 [[3-SVN-分支与标签]]）：

```
https://server/repos/项目名/trunk
https://server/repos/项目名/branches/feature-x
https://server/repos/项目名/tags/v1.0.0
```

常用查看命令：

```bash
svn info                  # 查看当前工作副本的 URL/版本号/作者
svn ls https://.../trunk  # 列出服务器目录
svn list -v URL           # 详细列表（含版本号、作者）
```

---

## 常见问题

**Q：`Unable to connect to a repository`？**
A：服务器地址不对或网络不通。用浏览器打开该 URL 验证；检查 HTTPS 证书、代理。

**Q：`Authentication failed` / 提示输密码？**
A：SVN 服务器需要账号密码。凭据会缓存在 `~/.subversion/auth/`（Linux/macOS）或 `%APPDATA%\Subversion\auth\`（Windows），输错后清除重试。

**Q：`svn: E155010: The node ... was not found`？**
A：工作副本与服务器状态不一致。先 `svn update` 同步，再操作。

**Q：checkout 到一半断了？**
A：重新执行 `svn checkout` 即可续传（会跳过已存在的完整部分）。

---

## 下一步

- 日常操作（add/commit/update/status/diff） → [[2-SVN-日常操作]]
- 分支与标签 → [[3-SVN-分支与标签]]
- 冲突解决 → [[4-SVN-冲突解决与历史]]
- 完整手册导航 → [[0-SVN 使用指南]]
