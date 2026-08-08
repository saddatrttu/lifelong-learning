# 7. 最佳实践与 FAQ

> 把前面各笔记串起来，总结一套**可落地**的工作流：权限模型、成本控制、安全、常见坑、故障排查和 FAQ。这是入门后最值得反复回看的一篇。

---

## 目录

1. [权限模型](#权限模型)
2. [成本控制](#成本控制)
3. [安全要点](#安全要点)
4. [完整工作流建议](#完整工作流建议)
5. [常见坑](#常见坑)
6. [故障排查](#故障排查)
7. [FAQ](#faq)
8. [进阶路线](#进阶路线)

---

## 权限模型

权限通过 `settings.json` 的 `permissions` 字段配置：

| 配置键 | 说明 | 示例 |
|--------|------|------|
| `model` | 默认模型 | `"claude-opus-4"` 或别名 `"opus"` |
| `effortLevel` | 工作量等级 | `"high"` |
| `permissions.allow` | 明确允许的工具操作 | `["Bash(npm run test:*)", "Read(src/**)"]` |
| `permissions.deny` | 明确禁止的操作 | `["Bash(rm -rf *)", "Read(.env*)"]` |
| `permissions.defaultMode` | 默认权限模式 | `"default"` / `"acceptEdits"` / `"plan"` |
| `env` | 环境变量注入 | `{"NODE_ENV": "development"}` |
| `hooks` | 自动化钩子 | 见 [[6-CC-Hooks与自动化]] |
| `autoMemoryEnabled` | 启用自动记忆 | `true` |
| `fileCheckpointingEnabled` | 启用文件快照 | `true` |
| `autoCompactEnabled` | 自动压缩对话 | `true` |

**核心习惯**：
- 用 `permissions.deny` 明确禁止危险操作（如 `rm -rf *`、读 `.env`）
- 常用安全命令用 `allow` 白名单，减少不必要的打断
- 结合 [[6-CC-Hooks与自动化|PreToolUse Hook]] 做最终兜底

---

## 成本控制

### 核心原则：防止 Context Rot

上下文窗口填满后 AI 推理能力显著下降。**三个关键习惯**：

1. 每次工作前用 `/context` 查看 Token 使用量
2. 切换任务时 `/clear` 完全清空记忆
3. 复杂任务前先用计划模式确认架构——写错代码后的重构成本远高于预先规划

### 成本控制清单

| 时机 | 动作 |
|------|------|
| 每次工作前 | `/context` 查看用量；确认 CLAUDE.md 控制在 500 行内；清理不必要的 MCP |
| 切换任务时 | `/clear` 清空记忆；或 `/compact` 保留关键信息 |
| 复杂任务前 | 先计划模式确认架构；用 Subagents 而非单一大 Context |
| MCP | 只留真正需要的 2-3 个（见 [[5-CC-MCP与外部集成]]） |
| 模型分级 | 机械任务用便宜快模型，架构/审查用最强模型 |

### 进阶技巧

| 技巧 | 说明 |
|------|------|
| Sub-agents | 明确分工任务派发便宜模型并行，成本可控（见 [[4-CC-Skills与子代理]]） |
| Fast Mode | 约 3 倍价格、2.5 倍速度，大量并行处理时用 |
| Effort Levels | 按任务复杂度调配推理投入 |
| 定时任务 `/loop` | 定期执行提示词（如每日 PR 摘要） |
| `/by the way` | 主任务中的临时提问，不污染主对话 |

---

## 安全要点

1. **CLAUDE.md 分层**：敏感规则放项目级，不要放全局 `~/.claude/CLAUDE.md`（会泄露给所有项目）
2. **密钥保护**：用 `permissions.deny` 禁止读取 `.env`；绝不让 Claude 打印密钥
3. **MCP 权限最小化**：MCP 服务器以你的账号权限运行（见 [[5-CC-MCP与外部集成]]）
4. **Hook 兜底**：PreToolUse 拦截 `rm -rf`、`git push --force` 等危险命令
5. **检查变更**：让 Claude 跑 `git diff` 后再提交，确认没有意外改动

---

## 完整工作流建议

一套**高效顺手的工作流**组合（各机制详见对应笔记）：

```
1. 会话启动 → CLAUDE.md 自动加载项目规则      [[3-CC-CLAUDE.md与上下文管理]]
2. 复杂任务 → 计划模式确认架构                [[2-CC-核心命令与交互]]
3. 需要外部能力 → 连接 MCP                     [[5-CC-MCP与外部集成]]
4. 按固定流程 → 调用 Skills / Subagents        [[4-CC-Skills与子代理]]
5. 自动执行 → Hooks 强制质量门禁               [[6-CC-Hooks与自动化]]
6. 上下文膨胀 → /compact 或 /clear             [[3-CC-CLAUDE.md与上下文管理]]
```

**进阶组合**（团队级）：
- MCP 配 Skills（接上外部工具，按固定流程处理）
- Subagents 配 Hooks（子代理干活，钩子把质量关）
- Plan Mode 配 Hooks（例行任务先过计划再自动跑）
- Custom Commands 配 Skills（团队公用快捷命令）
- Git Worktrees 隔离并行会话

---

## 常见坑

| 坑 | 后果 | 对策 |
|----|------|------|
| CLAUDE.md 过长 | 浪费 context，遵守率下降 | 拆分到 `.claude/rules/`，控制 200-500 行 |
| Skills 未迁移 2.0 格式 | 不出现在 UI 自动完成列表 | 迁移到 `.claude/skills/{name}/` 目录结构 |
| Hooks 配错位置 | 完全不生效 | 放 `settings.json` 的 `hooks` 字段，不是 CLAUDE.md |
| Auto Memory 与 CLAUDE.md 混用 | 行为不可预测 | 规则→CLAUDE.md，学习记录→Auto Memory |
| 团队项目直接改 CLAUDE.md | 影响所有人 | 个人偏好用 `CLAUDE.local.md` 覆写 |
| 一次接太多 MCP | 上下文被工具定义占满 | 只留 2-3 个真正需要的 |
| 敏感项目规则放全局 CLAUDE.md | 泄露给其他项目 | 规则按作用域分层 |
| 复杂任务不先计划 | 大重构返工 | 先计划模式 |

---

## 故障排查

| 症状 | 排查步骤 |
|------|---------|
| 命令找不到 | `claude --version`；检查 PATH；Windows 用 WSL |
| 登录异常 | `claude --login` 重新认证；`/login` 切换方式 |
| 插件/技能不生效 | `/doctor` 诊断；检查 Skills 目录结构 |
| 上下文太快耗尽 | `/context` 查看；`/compact`；清理 MCP |
| Hook 不触发 | 检查 `settings.json` 路径与格式；`/hooks` 查看 |
| 更新后行为变化 | 检查版本日志；`npm install -g @anthropic-ai/claude-code@latest` |

---

## FAQ

**Q：Claude Code 在哪些平台能用？**
A：终端、VS Code、JetBrains、桌面应用、Web、Slack，共享同一套引擎和配置（见 [[1-CC-安装与入门]]）。

**Q：CLAUDE.md 和 Skills 什么区别？**
A：CLAUDE.md 自动加载、始终在（"员工手册"）；Skills 按需触发（"SOP 流程"）。规则用 CLAUDE.md，流程用 Skills（见 [[3-CC-CLAUDE.md与上下文管理]]）。

**Q：MCP 和 Hooks 什么区别？**
A：MCP 提供外部工具能力（能力层）；Hooks 是确定性自动化（自动化层）。需要判断的让 Agent 做，固定流程用 Hooks（见 [[5-CC-MCP与外部集成]]、[[6-CC-Hooks与自动化]]）。

**Q：上下文（Context）满了怎么办？**
A：`/context` 查看 → `/compact` 压缩保留关键信息 → 或 `/clear` 完全清空切换新任务。

**Q：怎么防止 Claude 误删文件或执行危险命令？**
A：`settings.json` 配 `permissions.deny` + PreToolUse Hook 拦截 + 提交前 `git diff` 检查。

**Q：能和团队共享配置吗？**
A：能。CLAUDE.md、`.claude/skills/`、`.claude/settings.json`、MCP 配置均可 commit 进仓库自动共享。

**Q：与 Cursor、Copilot 比选哪个？**
A：Claude Code 是终端原生全库代理，适合深度工程任务；Cursor/Copilot 更适合 IDE 内补全。可以混合使用，配置通用。

---

## 进阶路线

1. **基础**：[[1-CC-安装与入门]] → [[2-CC-核心命令与交互]] → [[3-CC-CLAUDE.md与上下文管理]]
2. **进阶**：[[4-CC-Skills与子代理]] → [[5-CC-MCP与外部集成]] → [[6-CC-Hooks与自动化]]
3. **团队化**：配置共享 → 多代理流水线 → CI/CD 无人值守
4. **延伸**：本仓库 [[superpowers-使用说明]] 是一套可叠加在 Claude Code 上的工程方法论技能系统

---

## 相关笔记

- 全部导航 → [[0-Claude Code 使用指南]]
