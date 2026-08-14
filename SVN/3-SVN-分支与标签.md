# 3. 分支与标签

> SVN 用**目录拷贝**实现分支和标签：trunk 主干、branches 分支、tags 标签。理解这个目录模型是与 Git 分支最大不同的地方。

---

## 目录

1. [目录规范](#目录规范)
2. [创建分支](#创建分支)
3. [切到分支工作](#切到分支工作)
4. [创建标签](#创建标签)
5. [分支与标签的本质](#分支与标签的本质)
6. [合并（承接）](#合并承接)
7. [常见问题](#常见问题)

---

## 目录规范

SVN 仓库的推荐布局（标准布局 Standard Layout）：

```
repos/project/
├── trunk/          # 主干（主线开发）
├── branches/       # 分支（功能/发布）
│   ├── feature-login/
│   └── release-2.0/
└── tags/           # 标签（发布快照，通常只读）
    └── v1.0.0/
```

> **trunk/branches/tags 不是 SVN 强制**，只是社区约定（create 时勾选 "Create standard layout"）。SVN 对目录没有特殊语义，全靠约定。

---

## 创建分支

用 `svn copy` 把 trunk 拷贝到 branches 目录：

```bash
# 1. 在服务器上拷贝创建分支（推荐，直接服务器操作）
svn copy https://server/repos/project/trunk \
        https://server/repos/project/branches/feature-login \
        -m "创建登录功能分支"

# 2. 本地工作副本切到该分支
svn switch https://server/repos/project/branches/feature-login

# 也可以先本地拷贝再提交（适合带未提交改动时）
svn copy trunk branches/feature-login
svn commit -m "创建登录功能分支"
```

> 每次 `svn copy` 都会产生一次提交，版本号 +1。分支创建后**分支上的所有改动独立**，不影响 trunk。

---

## 切到分支工作

```bash
# 切换到分支（工作副本换 URL，保留本地修改）
svn switch https://server/repos/project/branches/feature-login

# 查看当前 URL
svn info | grep URL

# 新检出分支副本（全新目录）
svn co https://server/repos/project/branches/feature-login feature-login
```

> 在 trunk 和分支间来回切换用 `svn switch`；想同时并行开发多分支就多 checkout 几个目录（SVN 没有 Git 的 worktree 概念）。

---

## 创建标签

标签 = 某时刻的只读快照，同样用 `svn copy`：

```bash
# 给 trunk 当前状态打标签
svn copy https://server/repos/project/trunk \
        https://server/repos/project/tags/v1.0.0 \
        -m "发布 v1.0.0"

# 给指定版本打标签
svn copy -r 200 https://server/repos/project/trunk \
        https://server/repos/project/tags/v1.0.0 \
        -m "基于 r200 打标签"
```

> **约定：tags 只读。** 团队不往 tags 里提交，否则标签失去"固定快照"意义。SVN 无强制只读，靠规范。

---

## 分支与标签的本质

| 特性 | SVN | Git |
|------|-----|-----|
| 实现 | 服务器上目录的物理拷贝 | 指向提交的轻量指针 |
| 成本 | 拷贝整个目录树（首次占空间，后续增量） | 近乎零 |
| 版本号 | 每次 copy 全局 +1 | 每次提交一个哈希 |
| 概念 | 分支/标签=目录 | 分支=指针，标签=固定指针 |
| 多个分支 | 需要多个目录 | 一个工作区可随时切换 |

> 本质区别：**SVN 的"分支"是一个目录**，git 的"分支"是一个指针。所以 SVN 分支有空间/时间成本，Git 分支几乎免费、鼓励频繁开分支。这也是 Git 流行的重要原因。

---

## 合并（承接）

合并也是 `svn merge`，把分支改动合回 trunk（详细流程见 [[4-SVN-冲突解决与历史]]）：

```bash
# 场景：feature-login 分支开发完成，合回 trunk
svn switch https://server/repos/project/trunk     # 1. 回到主干
svn merge https://server/repos/project/branches/feature-login   # 2. 合并
svn status                                          # 3. 查看合并结果
svn commit -m "合并 feature-login 到主干"          # 4. 提交
```

> 与 Git 不同：SVN 合并**不自动记录"已合并"标记**，重复合并同一分支会再次应用同一批改动（可能冲突）。推荐用 `--reintegrate` 合并完整分支，或在合并说明中记录。

---

## 常见问题

**Q：合并时说 `Tree conflict`（树冲突）？**
A：分支里删了/移动了文件，而 trunk 上改了它。手动解决：选择保留哪边的操作（`svn resolve`，见 [[4-SVN-冲突解决与历史]]）。

**Q：误在 tags 里提交了？**
A：无法直接删除历史。用 `svn delete tags/v1.0.0` 删掉重打（`svn copy` 新标签），并立规范。

**Q：分支怎么区分是"功能分支"还是"发布分支"？**
A：命名约定：`feature-xxx` 功能、`release-x.y` 发布、`bugfix-xxx` 修复。目录名只是名字，语义靠规范。

**Q：创建分支后 trunk 上继续开发，分支会过时吗？**
A：会。分支创建后 trunk 的提交不会自动进分支。需要时 `svn merge` trunk → 分支 同步，或合并分支时处理差异（见 [[4-SVN-冲突解决与历史]]）。

---

## 下一步

- 冲突解决与历史 → [[4-SVN-冲突解决与历史]]
- 进阶（hooks/属性/迁移） → [[5-SVN-进阶与迁移]]
- Git 分支模型对照 → [[3-Git-分支与合并]]
- 完整手册导航 → [[0-SVN 使用指南]]
