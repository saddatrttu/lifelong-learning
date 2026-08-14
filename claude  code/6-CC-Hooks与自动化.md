# 6. Hooks 与自动化

> Hooks（钩子）是挂在 Claude Code 生命周期事件上的**确定性脚本**——到点自动执行，不依赖 LLM 判断。它是把"建议"变成"强制"的唯一机制，也是无人值守自动化的基石。

---

## 目录

1. [Hooks 是什么](#hooks-是什么)
2. [核心价值：建议 vs 法律](#核心价值建议-vs-法律)
3. [生命周期事件](#生命周期事件)
4. [Hook 类型](#hook-类型)
5. [配置方式](#配置方式)
6. [典型应用场景](#典型应用场景)
7. [Hooks vs CLAUDE.md](#hooks-vs-claudemd)
8. [CI/CD 集成](#cicd-集成)
9. [团队协作共享](#团队协作共享)

---

## Hooks 是什么

Hooks 是在特定事件触发时自动运行的脚本，用来：

- **强制执行规范**（如写文件后必跑格式化）
- **检查权限**（如拦截危险命令）
- **发通知**（如任务完成提醒）
- **审计记录**

与 [[4-CC-Skills与子代理|Skills]]、[[5-CC-MCP与外部集成|MCP]] 依赖 LLM 推理不同，Hooks 是**确定性执行**——shell 脚本跑不跑是确定的，不受模型概率性影响。

---

## 核心价值：建议 vs 法律

> CLAUDE.md 里写"每次都要跑测试"，社区普遍反馈 Claude 的遵守率不稳定，压力情境下常被跳过。而用 **Stop Hook** 强制跑测试，遵守率是 **100%**——因为这是确定性的 shell 执行。

这就是"**建议**"和"**法律**"的差距。规则放 CLAUDE.md（AI 可能偷懒），强制约束用 Hooks（必然执行）。

---

## 生命周期事件

| 事件 | 触发时机 | 是否需要 matcher |
|------|---------|-----------------|
| `SessionStart` | 会话开始时 | 否 |
| `SessionEnd` | 会话结束时 | 否 |
| `UserPromptSubmit` | 用户提交提示词时 | 否 |
| `PreToolUse` | 工具执行前 | 是（匹配工具名称） |
| `PostToolUse` | 工具执行后 | 是（匹配工具名称） |
| `Stop` | Claude 完成回应时 | 否 |
| `Notification` | Claude 发送通知时 | 否 |
| `SubagentStart` | 子代理启动时 | 否 |
| `SubagentStop` | 子代理结束时 | 否 |

---

## Hook 类型

| 类型 | 触发方式 | 场景 |
|------|---------|------|
| `command` | 执行 Shell 脚本 | Git 检查、代码格式化 |
| `http` | POST 请求到端点 | 远程通知、外部服务调用 |
| `mcp_tool` | 调用 MCP 工具 | 更新项目系统、Slack 消息 |
| `prompt` | LLM 评估（是/否） | 内容审核、合规检查 |
| `agent` | 子代理执行工具 | 复杂验证逻辑 |

---

## 配置方式

配置在项目 `.claude/settings.json`（团队共享）或 `~/.claude/settings.json`（全局）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "your-guard-script.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "npm test" }]
      }
    ]
  }
}
```

> ⚠️ **常见错误**：Hooks 配置在 `settings.json` 的 `hooks` 字段，**不是** CLAUDE.md 里。

---

## 典型应用场景

| 场景 | 用哪个事件 | 效果 |
|------|-----------|------|
| 拦截危险命令 | `PreToolUse` 匹配 `Bash` | 挡住 `rm -rf`、`git push --force`、`DROP TABLE` |
| 锁定目录防误改 | `PreToolUse` | 阻止写入 `themes/` 等关键目录 |
| 写文件后强制格式化 | `PostToolUse` 匹配 `Edit`/`Write` | 自动跑 prettier / lint |
| 收尾前跑测试 | `Stop` | 结束前强制 `npm test` |
| 危险操作确认 | `prompt` 类型 | 高风险操作弹 LLM 审核 |
| 任务完成通知 | `http` 类型 | 通知 Slack/远程服务 |

**简单判断法则**：
- 需要**判断**（"这段代码有没有问题"）→ 让 Agent 做（[[4-CC-Skills与子代理|Skills]]/[[5-CC-MCP与外部集成|MCP]]）
- 是**固定流程**（"每次编辑后格式化"）→ 用 Hooks

---

## Hooks vs CLAUDE.md

| 维度   | Hooks           | CLAUDE.md   |
| ---- | --------------- | ----------- |
| 执行方式 | 脚本进程，事件上运行      | 文本文档，会话启动读取 |
| 动态性  | 每次执行，反映最新状态     | 静态，加载后不更新   |
| 用途   | **强制执行**规范、权限检查 | 指导、教育、约定    |
| 性能   | 较慢（进程开销）        | 快速（内存中）     |
| 遵守率  | 确定性 100%        | 概率性，可能被跳过   |

---

## CI/CD 集成

Hooks 与非交互模式（`claude -p`，见 [[2-CC-核心命令与交互]]）配合，实现**无人值守**自动化：

```bash
# 在 CI 流水线中
claude -p "跑测试并修复失败" --allowedTools "Bash(npm run test:*)"
```

典型流水线：
1. CI 触发 → `claude -p` 执行任务
2. Hooks 强制质量门禁（格式化、测试、lint）
3. `--output-format json` 输出结构化结果
4. Git 工作流自动化（提交、PR）

配套：`/install-github-app` 可配置 GitHub Actions 集成。

---

## 团队协作共享

- 团队级 Hooks 放项目 `.claude/settings.json`，**commit 进仓库**，所有成员自动生效
- 个人偏好放 `~/.claude/settings.json`
- 团队可组合：**Subagents 配 Hooks**（子代理干活，钩子把质量关）、**Plan Mode 配 Hooks**（先计划再自动跑）

---

## 相关笔记

- 非交互模式与 CI → [[2-CC-核心命令与交互]]
- 规则与流程的层级 → [[3-CC-CLAUDE.md与上下文管理]]
- 与 Skills/MCP 的配合 → [[4-CC-Skills与子代理]]、[[5-CC-MCP与外部集成]]
- 安全与权限 → [[7-CC-最佳实践与FAQ]]
