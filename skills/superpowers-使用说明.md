# Superpowers 使用说明（完整参考手册）

> Superpowers 是一套面向 AI 编程助手（agent）的"技能"系统，把经过验证的工程方法论（如 TDD、系统化调试、写计划、代码审查等）封装成可复用的技能文件，让 AI 在合适的时机自动加载并严格执行。
>
> 本手册基于已安装的 Superpowers v5（OpenCode 插件版）编写，面向需要系统了解其全部能力的使用者。

> 📌 **本仓库相关手册**：本手册是"方法论"层；工具本身的使用见 [[0-Claude Code 使用指南]]，团队协作与版本控制见 [[0-Git 使用说明]]。三者可互相配合使用。

---

## 目录

1. [简介](#1-简介)
2. [安装](#2-安装)
3. [工作原理](#3-工作原理)
4. [核心规则：使用超能力之前](#4-核心规则使用超能力之前)
5. [流程一：规划与设计](#5-流程一规划与设计)
6. [流程二：实施](#6-流程二实施)
7. [流程三：审查与验证](#7-流程三审查与验证)
8. [流程四：收尾与交付](#8-流程四收尾与交付)
9. [辅助技能](#9-辅助技能)
10. [端到端示例](#10-端到端示例)
11. [故障排查与 FAQ](#11-故障排查与-faq)
12. [参考资料](#12-参考资料)

---

## 1. 简介

### 1.1 Superpowers 是什么

Superpowers（超能力）是一个由 [obra](https://github.com/obra/superpowers) 维护的开源项目。它通过 **插件 + 技能（Skills）** 的方式，把一套严格、可验证的工程方法论注入 AI 编程助手：

- **技能**：每个技能是一个 `SKILL.md` 文件，内含一套流程（何时用、怎么用、常见坑、红旗信号）。
- **插件**：负责把技能注册到 OpenCode，并在每次会话开始时把"使用技能"的规则注入上下文。

它解决的问题：**AI 编程助手在复杂任务中容易走捷径**——不写测试就写代码、不改根因就乱试修复、不验证就声称完成。Superpowers 用结构化的流程约束这些行为。

### 1.2 适用平台

- **OpenCode**（本手册主平台）
- Claude Code（使用指南见 [[0-Claude Code 使用指南]]）
- Codex / Copilot CLI
- Gemini CLI
- 其他支持 Anthropic Agent Skills 规范的运行时

每个平台需要单独安装 Superpowers。

### 1.3 核心理念

> **1% 原则**：只要有一丁点可能某个技能适用于当前任务，就必须先加载该技能再行动。

Superpowers 不是"可选工具"，而是 AI 与人类协作时共同遵守的工作协议。

### 1.4 已安装的技能清单（14 个）

| 技能 | 用途 | 所属流程 |
|------|------|---------|
| `using-superpowers` | 元技能：决定何时调用其他技能 | 核心规则 |
| `brainstorming` | 任何创造性工作前的需求探索与设计 | 规划与设计 |
| `writing-plans` | 根据规格编写实施计划 | 规划与设计 |
| `test-driven-development` | 先写测试再写实现的 TDD 纪律 | 实施 |
| `systematic-debugging` | 先找根因再修复的系统化调试 | 实施 |
| `subagent-driven-development` | 用子代理逐任务执行计划 | 实施 |
| `dispatching-parallel-agents` | 并行派发独立子代理 | 实施 |
| `using-git-worktrees` | 在隔离工作区中开发 | 实施 |
| `requesting-code-review` | 请求代码审查 | 审查与验证 |
| `receiving-code-review` | 接收审查反馈（技术核实） | 审查与验证 |
| `verification-before-completion` | 声称完成前必须验证 | 审查与验证 |
| `executing-plans` | 按计划执行（独立会话/批处理） | 收尾与交付 |
| `finishing-a-development-branch` | 收尾并决定如何整合分支 | 收尾与交付 |
| `writing-skills` | 创建/编辑/验证技能 | 辅助 |

---

## 2. 安装

### 2.1 标准安装（macOS / Linux / 无 git 问题的 Windows）

在 `opencode.json`（全局或项目级）的 `plugin` 数组中添加：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode，插件管理器会自动安装并注册全部技能。

**验证安装**：问 AI "Tell me about your superpowers"，应能列出全部技能。

### 2.2 Windows 安装（本机采用方案）

部分 Windows 版本的 OpenCode 对 `git+https` 插件规范存在上游安装问题（缓存路径、找不到 `git.exe` 等）。**推荐直接用系统 npm 安装并指向本地包**：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json`（此处为全局 `opencode.jsonc`）中：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

安装后结构如下：

```
~/.config/opencode/
  opencode.jsonc          # 配置文件
  node_modules/superpowers/
    skills/               # 14 个技能目录
    .opencode/plugins/superpowers.js  # 插件入口
```

### 2.3 从旧版（symlink 方式）迁移

若之前用 `git clone` + 软链接安装过，先清理：

```bash
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers
rm -rf ~/.config/opencode/superpowers
```

同时删除 `opencode.json` 中为 superpowers 添加的 `skills.paths`，然后按上述步骤重新安装。

### 2.4 更新

OpenCode 通过 git 依赖安装插件，部分版本会锁定已解析的 git 依赖，重启可能不会拉取最新提交。若更新不生效：

1. 清除 OpenCode 包缓存，或
2. 重装插件，或
3. 固定版本：`"plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]`

---

## 3. 工作原理

插件（`.opencode/plugins/superpowers.js`）做两件事：

1. **注入引导上下文**：通过 `experimental.chat.messages.transform` 钩子，把 `using-superpowers` 技能内容注入每次会话的第一条用户消息，使 AI 在无额外提示的情况下就知道"要用技能"。
   - 缓存在模块级别，只读盘一次，不重复消耗磁盘 I/O。
   - 若首条消息已包含 `EXTREMELY_IMPORTANT` 标记，则跳过注入，避免重复。

2. **注册技能目录**：通过 `config` 钩子，把 `node_modules/superpowers/skills` 自动追加到 OpenCode 的 `skills.paths`，无需手动软链接或改配置。

技能发现机制：OpenCode 扫描 `skills.paths` 下每个目录的 `SKILL.md`，读取其 YAML frontmatter（`name` + `description`），在 AI 需要时按描述匹配加载。

**技能优先级**：项目技能（`.opencode/skills/`）> 个人技能（`~/.config/opencode/skills/`）> Superpowers 内置技能。

### 3.1 技能文件结构

```markdown
---
name: my-skill
description: Use when [触发条件] - [做什么]
---

# 技能正文

## Overview（核心原则）
## When to Use（何时用 / 何时不用）
## 流程步骤
## Common Mistakes（常见坑）
## Red Flags（红旗信号，强制自检）
```

### 3.2 创建个人技能

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`（格式同上）。创建前应使用 `writing-skills` 技能（见 [9.1](#91-writing-skills-创建与验证技能)）。

---

## 4. 核心规则：使用超能力之前

> 这是元技能 `using-superpowers`，每次会话启动时自动注入，是其他一切技能的前提。

### 4.1 规则

1. **先查技能再行动**：在回答、探索代码、提问之前，先判断是否有技能适用。
2. **1% 原则**：只要认为有 1% 可能某个技能适用，就必须调用它。
3. **技能优先顺序**：过程类技能（brainstorming、systematic-debugging 等）优先于实现类技能，过程决定方法论，实现负责执行。
4. **用户指令 > 技能**：用户的明确指令（如 CLAUDE.md、直接要求）优先于技能规则，技能优先于默认行为。

### 4.2 决策对照

| "我正在想…" | 实际情况 |
|-------------|---------|
| "这只是一个简单问题" | 问题也是任务，先查技能 |
| "我需要更多上下文" | 技能检查在任何澄清问题**之前** |
| "让我先看看代码" | 技能规定了探索方式，先查 |
| "我快一点查下 git/文件" | 文件缺少对话上下文，先查技能 |
| "让我先收集信息" | 技能规定了信息收集方式 |
| "这不需要正式技能" | 技能存在就用 |
| "我记得这个技能" | 技能会更新，读最新版 |
| "这不算是任务" | 行动=任务，先查技能 |
| "这个技能有点小题大做" | 简单的事往往变复杂，用 |
| "我就先做这一件事" | 任何事之前都要先查 |

### 4.3 触发映射速查

| 场景                 | 先加载                              |
| ------------------ | -------------------------------- |
| 创建功能 / 组件 / 修改行为   | `brainstorming`                  |
| 修复 bug / 测试失败      | `systematic-debugging`           |
| 实现功能 / 修 bug（写代码前） | `test-driven-development`        |
| 有规格/需求的多步任务        | `writing-plans`                  |
| 声称完成 / 提交 / PR 前   | `verification-before-completion` |
| 收到代码审查反馈           | `receiving-code-review`          |
| 完成任务 / 合并前         | `requesting-code-review`         |
|                    |                                  |

### 4.4 OpenCode 工具映射

技能里写的是一般化动作，在 OpenCode 中对应：

| 技能中的动作 | OpenCode 工具 |
|--------------|---------------|
| 创建/更新待办 | `todowrite` |
| 派发 general-purpose 子代理 | `task`（`subagent_type: "general"`）|
| 代码库探索 | `task`（`subagent_type: "explore"`）|
| 调用技能 | `skill` |
| 读文件 | `read` |
| 建/改/删文件 | `edit` / `write` |
| 运行 shell 命令 | `bash` |
| 搜索文件内容 / 文件名 | `grep` / `glob` |
| 抓取 URL | `webfetch` |

---

## 5. 流程一：规划与设计

### 5.1 brainstorming（头脑风暴）

> **触发**：创建功能、构建组件、添加功能、修改行为的任何创造性工作之前。

**核心作用**：通过对话把模糊想法打磨成完整的设计与规格。

**过程（按清单逐项执行）：**

1. **探索项目上下文**：先看文件、文档、最近提交。
2. **逐个提问澄清**：一次只问一个问题（多用选择题），理解目的、约束、成功标准。
3. **提出 2-3 个方案**：带权衡，给出推荐与理由，严格遵守 YAGNI。
4. **分节呈现设计**：按复杂度缩放篇幅，每节确认后再继续。
5. **写设计文档**：保存到 `docs/superpowers/specs/YYYY-MM-DD-<主题>-design.md` 并提交。
6. **规格自审**：检查占位符、内部矛盾、范围、歧义。
7. **用户审阅规格**：确认后再进入实施。
8. **转入实施**：调用 `writing-plans` 编写实施计划。

**关键纪律（HARD-GATE）**：在呈现设计并获得用户批准前，**不得**调用任何实现技能、写任何代码、搭建任何项目——无论看起来多简单。

**常见坑**：
- 跳过探索直接提问
- 一次问多个问题
- 方案不做取舍分析
- 未经批准就动手实现

### 5.2 writing-plans（编写实施计划）

> **触发**：已有规格或需求，要执行多步任务，且尚未写代码。

**核心作用**：把规格转成"零上下文工程师也能照做"的逐任务实施计划。

**计划保存位置**：`docs/superpowers/plans/YYYY-MM-DD-<功能名>.md`

**关键要点：**

1. **开头声明**："I'm using the writing-plans skill to create the implementation plan."
2. **范围检查**：若规格覆盖多个独立子系统，拆成多个计划，每个独立产出可测试的软件。
3. **文件结构先行**：先规划要创建/修改的文件及其职责（单元边界清晰）。
4. **任务粒度（bite-sized）**：每步一个动作（2-5 分钟），形如：
   - 写失败测试 → 运行确认失败 → 写最小实现 → 运行确认通过 → 提交
5. **计划头必须包含**：面向 agent 执行者的提示、目标、架构、技术栈、全局约束。
6. **禁止占位符**：不得出现 "TBD"、"加适当错误处理"、"类似任务 N"、"写测试"（需给出真实测试代码）等。
7. **自审**：规格覆盖度、占位符扫描、类型/签名一致性。

**交付后转交执行**，二选一：
- **子代理驱动（推荐）** → `subagent-driven-development`
- **内联执行** → `executing-plans`

**常见坑**：
- 任务粒度过大（一次干太多事，无法独立测试）
- 步骤里只有描述没有代码
- 引用未在计划中定义的函数/类型
- 占位符式模糊指令

---

## 6. 流程二：实施

### 6.1 test-driven-development（测试驱动开发）

> **触发**：实现任何功能或 bug 修复，在写实现代码之前。

**核心原则**：

> 如果你没亲眼看到测试失败，你就不知道它测试的是不是对的东西。
> 违反规则的字面意思 = 违反规则的精神。

**铁律（Iron Law）**：

```
没有失败测试之前，不写任何生产代码
```

写了测试前的代码？删掉重来。没有例外：
- 不保留作"参考"
- 不"边改边测试"
- 不看它
- 删就是删

**红-绿-重构循环（RED-GREEN-REFACTOR）：**

1. **RED — 写失败测试**：一个最小测试，描述应该发生的行为。测试要：单一行为、命名清晰、测真实代码（能不用 mock 就不用）。
2. **验证 RED — 必须亲眼看到失败**：运行测试，确认它是因为功能缺失而失败（而不是打错字）。
3. **GREEN — 写最小实现**：刚好能让测试通过的最简代码，不加多余功能。
4. **验证 GREEN — 必须亲眼看到通过**：确认该测试及全部其他测试通过、输出干净。
5. **REFACTOR — 清理**：只在绿灯后做——消除重复、改进命名、提取辅助函数，保持测试常绿，不新增行为。

**常见合理化借口**（技能明令反驳）：

| 借口 | 现实 |
|------|------|
| "太简单不用测" | 简单代码也会坏，测试只要 30 秒 |
| "我之后补测试" | 事后补的测试直接通过，证明不了什么 |
| "我已经手动测过了" | 手动测试无记录、不可重复、易遗漏 |
| "删掉几小时的工作太浪费" | 沉没成本谬误——无法信任的代码才是浪费 |
| "保留作参考，先写测试" | 你一定会去改它，那还是事后测试 |
| "TDD 会拖慢我" | TDD 才是务实路径，能在提交前抓住 bug |

**红旗信号（出现即停止重来）**：代码先于测试、测试立即通过、说不出测试为何失败、事后补测试、"就这一次"。

**集成**：发现 bug → 先写复现它的失败测试 → 走 TDD 循环。没有测试，永不修 bug。

### 6.2 systematic-debugging（系统化调试）

> **触发**：遇到任何 bug、测试失败、意外行为，在提出修复之前。

**铁律**：

```
没有根因调查之前，不修复
```

未完成第一阶段，不得提出任何修复方案。

**四阶段流程：**

**阶段 1：根因调查**（修复前必须完成）
- 仔细读错误信息与完整堆栈
- 稳定复现
- 检查近期改动（git diff、提交、依赖、配置）
- 多组件系统：在每个组件边界加诊断日志，先收集证据定位故障层，再深入
- 深层调用栈：从坏值向上追溯来源，**修源头而不是症状**

**阶段 2：模式分析**
- 找代码库中类似的可工作代码
- 完整阅读参考实现（别只扫一眼）
- 列出工作与损坏代码的每个差异
- 理解依赖关系与假设

**阶段 3：假设与测试**（科学方法）
- 形成单一明确假设："我认为 X 是根因，因为 Y"
- 最小改动、一次一个变量验证
- 没验证成功就换新假设，别叠加修复

**阶段 4：实施**
- 先写失败测试用例（最简单的复现）
- 实施单一修复（一次一处，不做顺手重构）
- 验证修复、确认无回归
- 3 次以上修复仍失败 → **停止，质疑架构**，与人类讨论，而不是第 4 次尝试

**红旗信号**："先快速修一下再说"、"试一下 X 看行不行"、"跳过测试我手动验证"、"大概就是 X 修它"。全部意味着：停下，回到阶段 1。

**辅助技术**：根因回溯（`root-cause-tracing.md`）、纵深防御（`defense-in-depth.md`）、条件等待（`condition-based-waiting.md`）。

### 6.3 subagent-driven-development（子代理驱动开发）

> **触发**：在当前会话执行实施计划，且任务大部分相互独立。

**核心模式**：每个任务派发全新实现子代理 → 每个任务后做两级审查（规格合规 + 代码质量）→ 结束时做全分支宽泛审查。

**为什么用子代理**：为每个任务精确构造隔离上下文的代理，不继承你的会话历史；把大量 diff 和评估放在子代理上下文里，只把结论带回给你，节省你的上下文。

**关键要点：**

1. **先隔离工作区**：使用 `using-git-worktrees`，未经人类明确同意，绝不在 main/master 分支上开始实现。
2. **用账本（ledger）跟踪进度**：会话记忆在压缩后会丢失，把进度写入 `.superpowers/sdd/<计划>/progress.md` 账本，而不是只记在待办里。
3. **模型选择分级**：机械化任务用便宜快模型，集成/判断任务用标准模型，架构/最终审查用最强模型，并**显式指定**（不指定会继承会话模型，默认最贵）。
4. **每任务循环**：派发实现子代理 → 处理报告（DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED）→ 生成审查包并派发任务审查 → 修复循环（每任务最多 5 轮：1-3 轮恢复原实现者，4-5 轮换更强模型）→ 记录账本。
5. **断路器**：第 5 轮仍有关键问题 → 停止派发，逐条裁决（审查者错了就搁置并写理由；真实且被下游依赖的 → 停止并上报人类）。
6. **最终审查**：对整分支做一次宽泛代码审查（最强模型），一次性派发一个修复代理处理全部结论。

**连续执行**：任务之间不要停顿问人类"还要继续吗"——除非被阻塞、有真正歧义、或全部完成。

**常见坑**：
- 多个实现子代理并行派发（会冲突）
- 控制器自己修复发现的问题（污染上下文、跳过审查）
- 审查包用 `HEAD~1` 代替真实 BASE（会静默截断多提交任务）
- 跳过审查包只发文字 diff

### 6.4 dispatching-parallel-agents（并行派发代理）

> **触发**：面对 2 个以上相互独立、无共享状态或顺序依赖的任务。

**核心原则**：每个独立问题域派一个代理，让它们并发工作。

**决策流程**：
- 多个失败？→ 是否独立？
  - 相关（修一个可能连带修好其他）→ 一个代理处理全部
  - 独立 → 能否并行？
    - 能（无共享状态）→ 并行派发
    - 否（共享状态）→ 顺序执行

**典型用例**：3+ 个测试文件各自不同的根因、多个子系统独立损坏。

**代理提示词结构**（每个代理）：
1. **聚焦**——单一清晰的问题域（"修 agent-tool-abort.test.ts"）
2. **自包含**——包含理解问题所需的全部上下文（错误信息、测试名）
3. **明确输出**——要求返回什么（根因摘要 + 改动清单）
4. **加约束**——"不要改生产代码"等边界

**常见错误**：范围过宽（"把所有测试修好"）、无上下文、无约束、输出模糊。

**不使用场景**：失败相关、需要看完整系统、探索式调试（还不知道哪里坏了）、共享状态会互相干扰。

**整合**：代理返回后逐个读摘要、检查改动是否冲突、跑完整测试套件。

### 6.5 using-git-worktrees（使用 git 工作树）

> **触发**：开始需要隔离当前工作区的功能开发，或执行实施计划之前。

**核心原则**：先检测是否已隔离 → 优先用平台原生工具 → 退化用 git worktree → 永远不要和宿主环境对着干。

**流程：**

- **步骤 0：检测现有隔离**。用 `git rev-parse --git-dir` 与 `--git-common-dir` 判断是否已在 linked worktree 中；注意排除 git submodule 的干扰。已在隔离区则跳过创建。
- **步骤 1：创建隔离工作区**。优先原生工具（`EnterWorktree`、`/worktree`、`--worktree` 等）；没有原生工具才用 `git worktree add <path> -b <branch>`。
  - 目录优先级：用户指定偏好 > 项目内 `.worktrees/` > `worktrees/` > 默认 `.worktrees/`。
  - **必须**先用 `git check-ignore` 确认目录被 .gitignore 忽略，否则会误提交整个工作树。
- **步骤 2：项目设置**。自动检测并安装依赖（package.json → npm install 等）。
- **步骤 3：验证干净基线**。跑测试；失败则上报并询问是否继续。

**报告格式**：

```
Worktree ready at <完整路径>
Tests passing (<N> tests, 0 failures)
Ready to implement <功能名>
```

**常见坑**：
- 有原生工具却用 `git worktree add`（产生宿主看不到的"幽灵状态"）
- 不检查目录是否被忽略
- 跳过基线测试（脏基线会让后续所有失败变得模糊）

---

## 7. 流程三：审查与验证

### 7.1 requesting-code-review（请求代码审查）

> **触发**：完成任务时、实现重要功能后、合并到主分支前。

**核心原则**：早审查、常审查。

**强制场景**：
- 子代理驱动开发中每个任务之后
- 完成重要功能后
- 合并到 main 之前

**过程：**

1. 获取 git SHA：
   ```bash
   BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
   HEAD_SHA=$(git rev-parse HEAD)
   ```
2. 派发 `general-purpose` 子代理，填入审查者模板（`code-reviewer.md`）：{DESCRIPTION}、{PLAN_OR_REQUIREMENTS}、{BASE_SHA}、{HEAD_SHA}。
3. 处理反馈：Critical 立即修、Important 继续前修、Minor 记录稍后处理；审查者错了要有理由地反驳。

**关键纪律**：
- 只给审查者精确构造的上下文，绝不传整个会话历史
- 不要自己内联审查 diff（会烧掉你驱动工作所需的上下文）

### 7.2 receiving-code-review（接收代码审查）

> **触发**：收到代码审查反馈时，尤其在反馈不清晰或技术上可疑时——要求技术严谨与核实，而非表演式同意或盲目执行。

**核心原则**：先核实再实现，先提问再假设，技术正确性优先于社交舒适。

**响应模式**：

```
收到审查反馈时：
1. 读——完整阅读，不急于反应
2. 理解——用自己的话复述需求（或提问）
3. 核实——对照代码库现实
4. 评估——对这个代码库技术上是否成立
5. 回应——技术性确认或有理由的反驳
6. 实施——一次一项，逐项测试
```

**禁止的回应**（表演式）：
- "你说得完全对！"
- "好点子！""绝佳反馈！"
- "我这就去实现"（在核实之前）

**正确做法**：复述技术要求、提澄清问题、有理由地反驳、直接用行动说话。

**多条反馈的秩序**：
1. 先澄清任何不清楚的条目（部分理解 = 错误实现）
2. 阻塞性问题（破坏/安全）→ 简单修复（typo/import）→ 复杂修复（重构/逻辑）
3. 逐项单独测试，验证无回归

**何时反驳**：建议会破坏现有功能、审查者缺上下文、违反 YAGNI（未使用的功能）、对该技术栈不正确、存在兼容性/遗留原因、与人类的架构决策冲突。

**YAGNI 检查**："要'正经实现'这个功能？" → grep 实际用法，没人用就删（YAGNI），用了才实现。

**承认正确反馈**：直接说 "Fixed. [改了什么]"，不要感谢——行动比言辞更有力，代码本身证明你听到了反馈。

### 7.3 verification-before-completion（完成前验证）

> **触发**：准备声称工作已完成/已修复/已通过时，提交或创建 PR 之前。

**核心原则**：永远先有证据，再下结论。

**铁律**：

```
没有新鲜的验证证据，不得声称完成
```

如果验证命令不是在这条消息里运行的，你就不能声称它通过。

**门控函数**：

```
声称任何状态或表达满意之前：
1. 识别：什么命令能证明这个说法？
2. 运行：执行完整命令（全新、完整）
3. 阅读：完整输出、退出码、失败计数
4. 核实：输出是否证实说法？
   - 否：给出带证据的实际状态
   - 是：带着证据陈述说法
5. 只有此时才能下结论

跳过任何一步 = 说谎，而非验证
```

**常见失败对照**：

| 说法 | 需要 | 不够 |
|------|------|------|
| 测试通过 | 测试命令输出 0 失败 | 之前跑过、"应该能过" |
| 静态检查干净 | 检查输出 0 错误 | 部分检查、外推 |
| 构建成功 | 构建命令退出码 0 | 检查通过、日志看着不错 |
| Bug 已修 | 复测原症状通过 | 改了代码、假定已修 |
| 回归测试有效 | 红-绿循环验证过 | 测试通过一次 |
| 代理完成 | VCS diff 显示改动 | 代理报告"成功" |

**红旗信号**：使用"应该""大概""似乎"；验证前就表示满意（"太好了！""完美！""完成！"）；未验证就提交/推送/PR；相信代理的成功报告；部分验证。

**适用时机**：任何成功/完成类表述、表达满意、正向陈述工作状态、提交/PR/任务完成、进入下一任务、委托代理——之前。

---

## 8. 流程四：收尾与交付

### 8.1 executing-plans（执行计划）

> **触发**：在独立会话（带审查检查点）执行已写好的实施计划。

**提示**：Superpowers 在可用子代理的环境下效果更好（Claude Code、Codex、Copilot CLI、Gemini CLI 均可）。若有子代理，优先用 `subagent-driven-development` 而不是本技能。

**过程：**

1. **加载并批判性审查计划**：确保隔离工作区 → 读计划 → 批判性审查 → 有疑问先提出 → 无问题则建待办并开始。
2. **逐任务执行**：标记 in_progress → 严格照步骤执行 → 按要求验证 → 标记 completed。
3. **完成开发**：调用 `finishing-a-development-branch` 收尾。

**何时停下求助**（立即停止执行）：
- 遇到阻塞（缺依赖、测试失败、指令不清）
- 计划有关键缺口无法开始
- 不理解指令
- 验证反复失败

**纪律**：阻塞时停下提问，不猜；不跳过验证；未经用户明确同意，绝不在 main/master 分支开始实现。

### 8.2 finishing-a-development-branch（收尾开发分支）

> **触发**：实现完成、所有测试通过，需要决定如何整合工作。

**核心原则**：验证测试 → 检测环境 → 提供选项 → 执行选择 → 清理。

**流程：**

1. **验证测试**：跑完整测试套件。失败则上报并停止——选项菜单在绿灯之后出现。
2. **检测环境**：判断是普通仓库、具名分支 worktree、还是 detached HEAD，决定菜单项与清理方式。
3. **确定基线分支**：从计划/对话/上游推断；不确定就问，合并错基线代价高昂。
4. **呈现选项**（普通仓库/具名分支恰好 3 项，原文呈现，等用户决定）：
   ```
   Implementation complete. What would you like to do?
   1. Merge back to <base-branch> locally
   2. Push and create a Pull Request
   3. Keep the branch as-is (I'll handle it later)
   ```
   detached HEAD 则只有 2 项（无本地合并）。
5. **执行选择**：
   - 本地合并：切到基线、拉取、合并、合并后跑测试；失败则停下调查（本地可恢复）；绿了再清理并删分支。
   - 推送并建 PR：推送，用 forge 工具建 PR，报告 URL，保留 worktree（改 PR 反馈用）。
   - 保留原样：报告保留状态。
   - **丢弃**：仅当用户明确输入 `discard` 确认，才强制删分支。
6. **清理**：只清理自己创建的 worktree（`.worktrees/` 下），宿主环境的留着。

**常见坑**：
- 拿之前会话跑过的测试当"现在通过"（要对即将整合的树跑）
- 替用户决定合并/丢弃（整合是用户的决定）
- "嗯，去掉吧"不算确认，只有输入 `discard` 才授权删除
- 基线分支想当然（先确认分叉点）

---

## 9. 辅助技能

### 9.1 writing-skills（创建与验证技能）

> **触发**：创建新技能、编辑既有技能、验证技能在上线前可用。

**核心观点**：编写技能 = 把 TDD 应用到流程文档上。先跑基线场景看代理失败（RED）→ 写技能文档（GREEN）→ 封堵漏洞（REFACTOR）。

**铁律**：

```
没有失败测试之前，不写技能
```

对新技能和既有技能的编辑同样适用。不测试就改？删掉重来。

**技能 frontmatter 规范：**
- 必填：`name`（仅字母、数字、连字符）+ `description`（第三人称，**只描述何时用，不总结流程**，以 "Use when..." 开头，建议 <500 字符）
- 描述写"何时用"而非"做什么"——测试证明，总结流程的描述会让 agent 只照着描述做一次，而不读完整技能。

**TDD 映射**：测试用例=带压力的子代理场景；生产代码=SKILL.md；RED=无技能时 agent 违规；GREEN=有技能时 agent 遵守；REFACTOR=封堵新借口。

**防合理化技术**：
- 显式堵住每个漏洞（"No exceptions:" 列表）
- 回应"精神 vs 字面"争论（"违反字面=违反精神"）
- 建立合理化借口表（Excuse | Reality）
- 建立红旗信号清单
- 用微测试验证措辞（5+ 次重复、含无指导对照组）

**什么时候该创建技能**：技巧不明显、跨项目需要复用、模式通用、对他人有益。**不创建**：一次性方案、别处已有文档的标准做法、项目专属约定（放指令文件）。

**部署纪律**：每个技能写完必须走完整的 RED-GREEN-REFACTOR + 验证，禁止批量不测试就上线。

### 9.2 customize-opencode（配置 OpenCode）

> **触发**：编辑或创建 OpenCode 自身配置时：`opencode.json` / `opencode.jsonc`、`.opencode/` 下的文件、`~/.config/opencode/` 下的文件；创建或修复 agent、子代理、技能、插件、MCP 服务器、权限规则时。

**不适用**：用户自己的应用代码，或任何非 OpenCode 配置的项目。

**典型用途**：注册插件、配置 `skills.paths`、创建子代理与权限规则、配置 MCP。

---

## 10. 端到端示例

一个需求从想法到合并的完整旅程（在 OpenCode 中）：

**第 1 步：想法 → 设计（brainstorming）**

```
你：我想给这个项目加一个导出功能。
AI：（加载 brainstorming）先探索项目结构，然后逐个提问澄清：
   - "导出格式需求是什么？CSV / JSON / Excel？"（一次一问）
   - 提出 2-3 个方案及权衡
   - 分节呈现设计，获得你的批准
   - 写入 docs/superpowers/specs/2026-08-08-export-feature-design.md
```

**第 2 步：设计 → 计划（writing-plans）**

```
AI：（加载 writing-plans）按规格拆解任务：
   - Task 1：数据模型与读取层（先写失败测试）
   - Task 2：CSV 序列化器
   - Task 3：命令行入口
   每个任务含完整代码、测试步骤、验证命令、提交信息。
   计划保存至 docs/superpowers/plans/2026-08-08-export-feature.md。
```

**第 3 步：计划 → 执行（subagent-driven-development / using-git-worktrees）**

```
AI：（加载 using-git-worktrees）创建隔离 worktree。
（加载 subagent-driven-development）逐任务派发实现子代理：
   - 每个任务：实现 → 测试 → 提交 → 审查（规格+质量）→ 账本记录
   - 修复循环最多 5 轮，必要时换更强模型
   - 每任务结束后（加载 requesting-code-review）派审查子代理
```

**第 4 步：验证（verification-before-completion）**

```
AI：（加载 verification-before-completion）跑完整测试：
   ✅ [npm test]  [See: 42/42 pass]  "All tests pass"
   ✅ [npm run lint] [See: 0 errors]  "Lint clean"
   只有看到输出才声称通过。
```

**第 5 步：收尾（finishing-a-development-branch）**

```
AI：（加载 finishing-a-development-branch）复跑测试 → 检测环境 → 呈现：
   1. Merge back to main locally
   2. Push and create a Pull Request
   3. Keep the branch as-is
你选择后执行并清理 worktree。
```

---

## 11. 故障排查与 FAQ

### 11.1 插件未加载

1. 检查日志：`opencode run --print-logs "hello" 2>&1 | Select-String -Pattern "superpowers"`
2. 检查 `opencode.json` 中的 plugin 行
3. 确认 OpenCode 版本足够新

### 11.2 Windows 安装问题

症状：`git+https` 插件规范安装失败（缓存路径问题、找不到 `git.exe`）。
解决：用系统 npm 安装并指向本地路径（见 [2.2](#22-windows-安装本机采用方案)）。

### 11.3 技能找不到

1. 用 `skill` 工具列出已发现的技能
2. 检查插件是否加载（见 11.1）
3. 每个技能目录需含带合法 YAML frontmatter 的 `SKILL.md`

### 11.4 引导上下文不出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.messages.transform` 钩子
2. 改配置后重启 OpenCode

### 11.5 FAQ

**Q：我可以只挑几个技能用吗？**
A：技能是自动按 1% 原则触发的，但你可以明确要求"这次不要用 X 技能"。用户指令优先于技能。

**Q：技能会增加每次对话的 token 开销吗？**
A：引导上下文只注入一次（首条用户消息），且模块级缓存只读盘一次。

**Q：我想自定义流程怎么办？**
A：个人技能（`~/.config/opencode/skills/`）优先级高于内置技能；也可用 `writing-skills` 创建新技能。

**Q：更新后技能没变？**
A：git 依赖可能被锁定，清除缓存或重装（见 [2.4](#24-更新)）。

**Q：在别的工具（Claude Code 等）里能用吗？**
A：能，但需要为每个工具单独安装 Superpowers。Claude Code 的安装与使用方法见 [[0-Claude Code 使用指南]]。

---

## 12. 参考资料

- 项目主页：https://github.com/obra/superpowers
- OpenCode 版文档：`node_modules/superpowers/docs/README.opencode.md`
- 问题反馈：https://github.com/obra/superpowers/issues
- OpenCode 官方文档：https://opencode.ai/docs/

### 本仓库相关手册

- [[0-Claude Code 使用指南]] — AI 编程助手工具本身的使用手册（安装、命令、CLAUDE.md、Skills、MCP、Hooks）
- [[0-Git 使用说明]] — Git 版本控制完整参考手册（分支、合并、远程协作、工作流）

### 技能文件位置

```
~/.config/opencode/node_modules/superpowers/skills/
  using-superpowers/
  brainstorming/
  writing-plans/
  test-driven-development/
  systematic-debugging/
  subagent-driven-development/
  dispatching-parallel-agents/
  using-git-worktrees/
  requesting-code-review/
  receiving-code-review/
  verification-before-completion/
  executing-plans/
  finishing-a-development-branch/
  writing-skills/
```

每个技能目录内含 `SKILL.md`（主文档），部分还含辅助参考文件（如 TDD 的 `writing-good-tests.md`、调试的 `root-cause-tracing.md` 等）。
