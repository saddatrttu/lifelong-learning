# 8. 最佳实践与 FAQ

> 把前面各笔记串起来，总结一套**可落地**的 Git 使用规范：提交规范、.gitignore、安全红线、常见坑、故障排查和 FAQ。

---

## 目录

1. [每日工作流](#每日工作流)
2. [提交规范](#提交规范)
3. [.gitignore 与敏感信息](#gitignore-与敏感信息)
4. [安全红线](#安全红线)
5. [常用别名与配置](#常用别名与配置)
6. [常见坑](#常见坑)
7. [故障排查速查表](#故障排查速查表)
8. [FAQ](#faq)
9. [进阶路线](#进阶路线)

---

## 每日工作流

一套高效、不易出错的日常节奏：

```
1. 开工前：git status 看清现状，git pull --rebase 同步
2. 建分支：git switch -c feature/xxx
3. 小步开发：改一点 → git diff 检查 → git add → git commit
4. 经常推送：git push -u origin feature/xxx（尽早 push，防丢失）
5. 保持同步：git pull --rebase 合并 main 的更新
6. 提交收尾：git status / git log --oneline 检查
7. 发起 PR → 评审 → 合并
```

---

## 提交规范

### 小而独立

- **一次提交只做一个逻辑变更**——便于审查、回滚、定位
- 提交前 `git status` + `git diff` 检查，别把无关文件卷进来

### Conventional Commits

```
<type>(<scope>): <subject>
```

| type | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(auth): 添加 OAuth 登录` |
| `fix` | 修复 | `fix(parser): 修复空字符串崩溃` |
| `docs` | 文档 | `docs: 补充安装说明` |
| `refactor` | 重构 | `refactor(core): 提取校验函数` |
| `test` | 测试 | `test: 补充边界用例` |
| `chore` | 杂项 | `chore: 升级依赖` |

**好处**：可读历史 + 自动生成 changelog + 语义化版本发布。

> 💡 想要更完整的提交规范模板，可查看本仓库的 [[superpowers-使用说明]] 中关于代码审查与完成前验证的部分。

---

## .gitignore 与敏感信息

### .gitignore

忽略不该进仓库的文件，避免仓库膨胀和密钥泄漏：

```gitignore
# 依赖
node_modules/
vendor/

# 构建产物
dist/
build/

# 环境与密钥
.env
*.env.local
secrets*.json

# 系统文件
.DS_Store
Thumbs.db

# 本地配置（如 Obsidian 个人笔记）
*.local.md
```

```bash
git status          # 确认忽略生效
git check-ignore -v node_modules/   # 查看哪条规则命中
```

### 敏感信息红线

| 错误 | 后果 | 补救 |
|------|------|------|
| 提交 `.env` / 密钥 | 永久留在历史里 | 立即移除并**更换密钥**（改历史不够） |
| 提交后才知道 | 泄漏 | 换密钥 + `git filter-repo` 清理历史 |
| 大文件入库 | 仓库永久膨胀 | 用 LFS（见 [[6-Git-高级技巧]]） |

> ⚠️ **一旦密钥入库，必须视为已泄漏**——改历史没有意义，换密钥才是唯一正确动作。

---

## 安全红线

1. **绝不 force push 共享分支**（`main`/`develop`）
2. **绝不在公共分支上 rebase/reset 已推送提交**（见 [[5-Git-历史与回滚]]）
3. **密钥绝不入库**，用 `.gitignore` + 环境变量
4. **保护分支**：配置禁止直接 push main、强制 PR + CI（见 [[7-Git-工作流与团队协作]]）
5. **强制推送加保护**：若必须 `git push --force-with-lease`（比 `--force` 安全，会检查远程是否变化）
6. **匿名 URL 不用**：`.git` 泄漏可能暴露源码

---

## 常用别名与配置

把高频命令缩短：

```bash
git config --global alias.st  status
git config --global alias.co  checkout
git config --global alias.ci  commit
git config --global alias.br  branch
git config --global alias.lg  "log --oneline --graph --all"
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.uncommit "reset --soft HEAD^"
```

其他实用配置：

```bash
git config --global core.editor "code --wait"
git config --global init.defaultBranch main
git config --global pull.rebase true        # pull 默认用 rebase
git config --global rerere.enabled true     # 记住冲突解法
git config --global alias.lol "log --oneline --graph --decorate"
```

---

## 常见坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 直接改 main | 混乱、难回滚 | 功能走分支（[[3-Git-分支与合并]]） |
| 大提交 | 难审查、难回滚 | 小步提交（[[2-Git-核心命令与提交]]） |
| 把密钥 commit 进去 | 安全泄漏 | 换密钥 + .gitignore（见上） |
| 提交无关文件 | 历史污染 | 提交前 `git status` 检查 |
| 在共享分支 rebase | 团队历史崩溃 | 共享分支只用 merge/revert |
| 忘加 `-p` 乱 commit | 误提交 | `git add -p` 精细暂存 |
| 大文件入库 | 仓库膨胀 | LFS / 外部存储 |
| 提交后才发现漏文件 | 提交不完整 | `--amend`（未推送时） |

---

## 故障排查速查表

| 症状 | 排查 | 解法 |
|------|------|------|
| 提交人信息错误 | `git log -1` | 全局配置 + 补改作者 |
| push 被拒 | `git status` / `git fetch` | `git pull --rebase` 再 push |
| 冲突一堆 | `git status` | 手动解决或 `--abort`（[[3-Git-分支与合并]]） |
| 删错文件/提交 | `git reflog` | 用 reflog 找回（[[5-Git-历史与回滚]]） |
| 分支找不到了 | `git reflog` | `git branch <name> <sha>` |
| 文件被忽略却没生效 | `git check-ignore -v` | 修正 .gitignore 规则 |
| 仓库体积大 | `git count-objects -vH` | 查大文件 + LFS + filter-repo |
| 找不到 bug 引入提交 | — | `git bisect`（[[6-Git-高级技巧]]） |
| 想知道某行谁写的 | — | `git blame`（[[5-Git-历史与回滚]]） |

---

## FAQ

**Q：merge 和 rebase 到底选哪个？**
A：公共分支用 merge；本地私有分支用 rebase 整理；共享分支拉更新用 `git pull --rebase`。详见 [[3-Git-分支与合并]]。

**Q：reset 和 revert 什么区别？**
A：`reset` 移动指针改写历史（仅限本地）；`revert` 生成反向提交保留历史（共享分支用这个）。详见 [[5-Git-历史与回滚]]。

**Q：误删了文件或分支还能找回吗？**
A：本地用 `git reflog`（90 天内）；已推送的在远程仓库找回。详见 [[5-Git-历史与回滚]]。

**Q：.gitignore 不生效？**
A：可能文件已被跟踪。先 `git rm --cached` 从索引移除再 commit。

**Q：团队成员水平不齐，怎么降低踩坑？**
A：统一工作流（[[7-Git-工作流与团队协作]]）+ 保护分支 + CI + 写好 CONTRIBUTING.md。

**Q：提交信息规范太麻烦，有必要吗？**
A：团队统一后收益显著：历史可读、自动 changelog、语义化发布。选一个轻量规范即可。

**Q：Git 和 GitHub 是什么关系？**
A：Git 是版本控制工具；GitHub/GitLab/Gitee 是托管 Git 仓库的远程平台。

**Q：想系统学习 Git 有什么资源？**
A：Pro Git 中文版（https://git-scm.com/book/zh/v2）+ 交互式练习（https://learngitbranching.js.org）。

---

## 进阶路线

1. **基础**：[[1-Git-安装与入门]] → [[2-Git-核心命令与提交]] → [[3-Git-分支与合并]]
2. **协作**：[[4-Git-远程协作与PR]] → [[7-Git-工作流与团队协作]]
3. **救场**：[[5-Git-历史与回滚]] → [[6-Git-高级技巧]]
4. **延伸**：本仓库 [[0-Claude Code 使用指南]] 可让 AI 助手自动完成大部分 Git 工作流

---

## 相关笔记

- 全部导航 → [[0-Git 使用说明]]
