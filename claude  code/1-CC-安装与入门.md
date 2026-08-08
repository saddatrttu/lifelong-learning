# 1. 安装与入门

> 从零开始安装 Claude Code 并完成第一次会话。上手很快，但先搞清楚它是什么、跑在哪里，能少踩很多坑。

---

## 目录

1. [Claude Code 是什么](#claude-code-是什么)
2. [支持的平台](#支持的平台)
3. [安装要求](#安装要求)
4. [安装步骤](#安装步骤)
5. [登录与认证](#登录与认证)
6. [首次会话](#首次会话)
7. [核心概念](#核心概念)
8. [常见问题](#常见问题)

---

## Claude Code 是什么

Claude Code 是 Anthropic 开发的命令行 AI 编码助手（CLI Tool），能直接在终端：

- **读取并理解**整个项目结构
- **编辑**多个文件（单次操作自主完成）
- **运行**测试、构建命令和脚本
- **自动化** Git 工作流（提交、分支、PR）
- 通过 **MCP**、**Hooks**、**Skills** 和自定义命令扩展

本质上它是一套"AI 程序员"，你通过自然语言交代任务，它自主规划并执行。

> ⚠️ Claude Code 目前仍在**活跃开发**中，功能迭代快，建议定期查看官方文档和更新日志。

---

## 支持的平台

Claude Code 跑在多个界面上，但**共享同一套引擎**——`CLAUDE.md`、settings、MCP 配置在所有平台通用，换的只是"壳"：

| 平台 | 说明 |
|------|------|
| 终端（Terminal） | 原生主平台，功能最全 |
| VS Code 扩展 | IDE 内使用 |
| JetBrains 插件 | IntelliJ 系列 |
| 桌面应用（Desktop） | 独立 GUI |
| Web 网页版 | 浏览器使用 |
| Slack | 集成到 Slack 频道 |

挑顺手的用即可，配置跟着账号走。

---

## 安装要求

- **Node.js** 版本要求见官方文档（一般需要 Node 18+）
- 或使用原生安装脚本（自动处理运行时）
- Windows 支持 WSL、Git Bash、原生 PowerShell/Cmd

---

## 安装步骤

### 方式一：npm 安装（推荐，可版本控制）

```bash
npm install -g @anthropic-ai/claude-code
```

### 方式二：原生安装脚本（无需 Node 预装）

```bash
# macOS / Linux
curl -fsSL https://claude.ai/install.sh | bash

# Windows (PowerShell)
irm https://claude.ai/install.ps1 | iex
```

### 方式三：查看/切换版本

```bash
# 查看已安装版本
claude --version

# 安装指定版本
npm install -g @anthropic-ai/claude-code@1.0.0
```

### 验证安装

```bash
claude --help    # 显示帮助
claude doctor    # 诊断安装问题
```

---

## 登录与认证

```bash
claude
```

首次运行会引导登录，几种认证方式：

| 方式 | 场景 |
|------|------|
| Claude 账号（免费/Pro/Max） | 个人使用 |
| Anthropic Console API Key | 按量付费、API 场景 |
| 企业 SSO | 团队组织账号 |

切换认证：

```bash
claude --login      # 重新登录
claude --logout     # 退出
```

会话中可用 `/login` 斜杠命令切换认证方式。

---

## 首次会话

```bash
cd /path/to/your/project
claude
```

启动后直接**用自然语言**描述任务即可，例如：

```
解释一下这个项目是干什么的
修复 src/utils.ts 里的 bug，并补上测试
帮我把新功能提交到 main 分支并创建 PR
```

几点提示：

- 首次在项目里使用，建议先运行 `/init` 生成 `CLAUDE.md`（见 [[3-CC-CLAUDE.md与上下文管理]]）
- 会话中按 `?` 查看所有快捷键
- 输入 `/` 查看所有可用命令和自定义技能（见 [[2-CC-核心命令与交互]]）

---

## 核心概念

### Agent 工作方式

Claude Code 不是"自动补全"，而是 **Agentic（代理式）**——它自主：

1. 读取代码库理解上下文
2. 规划执行步骤
3. 调用工具（编辑文件、跑命令、查 Git）
4. 自我检查并迭代

### 四种权限/交互模式

日常可用 `Shift+Tab` 循环切换（详见 [[2-CC-核心命令与交互]]）：

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| 默认模式 | 请求授权后执行 | 日常开发 |
| 自动编辑（acceptEdits） | 自动执行编辑类操作 | 高频小改动 |
| 计划模式（Plan Mode） | 只读，只输出方案不改文件 | 复杂功能设计 |
| 旁路/跳过权限 | 不请求直接执行 | 信任的自动化场景 |

> 💡 **建议**：开始复杂功能前先用**计划模式**确认架构，研究显示先规划再实现平均可节省约 10 倍的重构成本。

### 上下文（Context）

Claude Code 每个会话从**全新上下文窗口**开始（最高 200K token，部分模型支持 1M）。会话启动时按顺序加载：

```
系统提示 → 自动记忆 MEMORY.md → 环境信息 → MCP 工具列表
→ Skill 描述清单 → 用户级 CLAUDE.md → 项目 CLAUDE.md → 路径作用域规则
```

上下文管理是控制质量和成本的关键，详见 [[3-CC-CLAUDE.md与上下文管理]]。

---

## 常见问题

**Q：安装后 `claude` 命令找不到？**
A：检查 npm 全局目录是否在 PATH；Windows 用户可尝试 WSL。

**Q：想卸载怎么办？**
```bash
npm uninstall -g @anthropic-ai/claude-code
```

**Q：与 Cursor / Copilot 有什么区别？**
A：Claude Code 是终端原生全库代理，适合深度工程任务；Cursor/Copilot 更适合 IDE 内补全。可以混合使用，配置通用。

---

## 下一步

- 熟悉命令和交互 → [[2-CC-核心命令与交互]]
- 配置项目规则 → [[3-CC-CLAUDE.md与上下文管理]]
- 完整手册导航 → [[0-Claude Code 使用指南]]
