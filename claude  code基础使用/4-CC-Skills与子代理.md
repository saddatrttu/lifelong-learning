# 4. Skills 与子代理

> Skills（技能）是 Claude Code 的**可执行工作流模块**——把重复性工作封装成按需加载的指令集；Subagents（子代理）是多代理并行执行的关键。本笔记覆盖两者及组合用法。

---

## 目录

1. [Skills 是什么](#skills-是什么)
2. [Skills 2.0 目录结构](#skills-20-目录结构)
3. [SKILL.md 怎么写](#skillmd-怎么写)
4. [如何触发 Skills](#如何触发-skills)
5. [自定义命令](#自定义命令)
6. [Subagents 子代理](#subagents-子代理)
7. [Agent Teams 代理团队](#agent-teams-代理团队)
8. [Skills vs 其他机制](#skills-vs-其他机制)

---

## Skills 是什么

Skills 是将重复性工作封装为**自动化流程**的指令集，存放在 `.claude/skills/` 下的独立文件夹中。

**核心优势——渐进式加载**：
- 平时每个 Skill 仅消耗约 **60 Tokens**（只加载 description）
- **触发后才加载完整指令**
- 既可由 Claude 根据 description **自动判断**调用，也可由用户**显式 `/技能名`** 触发

如果 CLAUDE.md 是"宪法"，Skills 就是"**标准操作程序**"——只在需要时才拿出来用。

---

## Skills 2.0 目录结构

2026 年的 Skills 2.0 更新引入 folder-based 结构（区别于旧版单文件命令）：

```
.claude/skills/
└── article-writing/
    ├── SKILL.md              # 主指令（含 YAML frontmatter）
    ├── scripts/
    │   └── check-seo.py      # 可执行脚本
    ├── references/
    │   └── brand-voice.md    # 参考文档（按需加载）
    └── assets/
        └── template.html     # 素材资源
```

| 组成部分 | 作用 | 加载时机 |
|---------|------|---------|
| SKILL.md 的 YAML frontmatter | 定义 name 和 description，让 Claude 知道何时自主调用 | 会话启动（仅 description） |
| SKILL.md 正文 | 完整操作指令 | 触发时 |
| scripts/ | 可执行脚本 | 触发后可用 |
| references/ | 参考文件 | 执行中按需读取 |

> 旧版 `.claude/commands/*.md` 仍可调用，但不再出现在新版 UI 自动完成列表（截至 2026 年官方尚未公布正式废弃时间表）。

---

## SKILL.md 怎么写

```markdown
---
name: code-review
description: Use when user asks to review code changes
disable-model-invocation: true
---

# 步骤
1. 获取改动 diff
2. 逐文件检查
3. 输出分级反馈（Critical/Important/Minor）
```

**frontmatter 关键字段**：
- `name`：技能名（kebab-case）
- `description`：**必须包含触发条件**（"Use when user asks to..."），这是 Claude 自动判断的依据
- `disable-model-invocation`：`true` 时禁止 Claude 自动调用，只能手动触发（适合操作型 Skills）
- `user-invocable`：控制是否出现在 `/` 菜单
- `context: fork`：隔离运行

**写作要点**：
- 单一职责，一个 Skill 只做一件事
- 步骤要具体可执行，不要含糊
- 详细文档放 `references/` 目录
- 考虑异常路径和错误处理
- 规则 → CLAUDE.md；流程 → Skills（详见 [[3-CC-CLAUDE.md与上下文管理]]）

---

## 如何触发 Skills

| 方式 | 说明 |
|------|------|
| 手动触发 | 会话中输入 `/skill-name` |
| 自动调用 | Claude 根据 `description` 判断何时需要 |
| 流程委派 | 由其他 Skill 流程自动触发 |

---

## 自定义命令

自定义命令本质上是轻量 Skills：

- **项目级**：`.claude/commands/*.md`（团队共享）
- **用户级**：`~/.claude/commands/*.md`（个人通用）

创建后可出现在 `/` 菜单中，如 `/review-pr`、`/daily-standup`、`/scout` 等。适合把高频动作固化成团队快捷命令。

---

## Subagents 子代理

Subagents 让 Claude Code 具备**多角色分工**能力：

- 定义在 `.claude/agents/` 目录
- 每个 Subagent 有**独立的上下文窗口**
- 主代理（通常最强模型）负责任务分发，Subagents（可用便宜模型）并行执行

```
.claude/agents/
├── test-runner.md      # 专门跑测试
├── code-reviewer.md    # 专门审查代码
└── doc-writer.md       # 专门写文档
```

调用方式：在对话中用 `@agent-name` 直接调用，或让 Skill 流程自动委派。

**适用场景**：明确分工的任务（如"并行审查 3 个模块"、"同时跑测试和文档"）。
**成本优势**：机械任务用便宜快模型，成本可控（见 [[7-CC-最佳实践与FAQ]] 的成本控制）。

---

## Agent Teams 代理团队

更进阶的**实验性**功能，多代理完全平等且能互相沟通：

- 需在 `settings.json` 开启
- Token 消耗是单一对话的 **7 倍以上**
- 适合探索性、需要辩论或头脑风暴的任务
- **必须搭配 Git Worktrees 防止文件冲突**

**选择建议**：

| 维度 | Subagents | Agent Teams |
|------|-----------|-------------|
| 关系 | 主代理指挥 | 完全平等 |
| 通信 | 单向派发 | 互相沟通 |
| 成本 | 可控 | 很高（7 倍+） |
| 适用 | 明确分工 | 探索/辩论 |
| 隔离 | 天然隔离 | 需 Worktrees |

---

## Skills vs 其他机制

| 维度 | CLAUDE.md | Skills | MCP | Hooks |
|------|-----------|--------|-----|-------|
| 类比 | 员工手册 | SOP 流程 | 工具箱接口 | 流水线钩子 |
| 解决 | "这是什么项目" | "遇到 X 怎么做" | "能用什么工具" | "自动执行任务" |
| 加载 | 自动加载 | 按需触发 | 按需连接 | 事件触发 |
| 执行 | LLM 推理 | LLM 推理 | LLM 推理 | 确定性脚本 |

- CLAUDE.md + Skills = **知识层**（告诉 Agent 怎么思考）
- MCP + Hooks = **能力层**（让 Agent 能动手）
- MCP 让 Skill 有工具可用；Skill 告诉 Agent 怎么用好这些工具（详见 [[5-CC-MCP与外部集成]]）

---

## 相关笔记

- Skills 与 CLAUDE.md 的分工 → [[3-CC-CLAUDE.md与上下文管理]]
- 给 Skills 接外部工具 → [[5-CC-MCP与外部集成]]
- 用 Hooks 强制执行 → [[6-CC-Hooks与自动化]]
- 多代理的成本控制 → [[7-CC-最佳实践与FAQ]]
