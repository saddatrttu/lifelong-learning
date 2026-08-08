# 2. 核心命令与交互

> Claude Code 的核心交互方式：交互模式、斜杠命令、快捷键、非交互 Headless 模式与会话管理。这是日常使用频率最高的一张速查表。

---

## 目录

1. [启动与退出](#启动与退出)
2. [四种权限/交互模式](#四种权限交互模式)
3. [常用快捷键](#常用快捷键)
4. [斜杠命令速查](#斜杠命令速查)
5. [常用 CLI 参数](#常用-cli-参数)
6. [非交互模式（Headless / CI）](#非交互模式headless--ci)
7. [会话管理](#会话管理)
8. [上下文与成本](#上下文与成本)

---

## 启动与退出

```bash
claude                # 在当前目录启动交互会话
claude /path/to/dir   # 指定工作目录
claude --resume       # 恢复最近的会话
claude --continue     # 继续最近的会话
claude --debug        # 调试模式（打印详细日志）
```

退出：输入 `/exit` 或按 `Ctrl+C` / `Ctrl+D`。

---

## 四种权限/交互模式

用 `Shift+Tab` 循环切换，或通过 `/mode` 查看当前模式。

| 模式 | 名称 | 行为 | 适用场景 |
|------|------|------|---------|
| 默认 | default | 每个操作请求授权 | 日常开发（推荐默认） |
| 自动编辑 | acceptEdits | 自动接受文件编辑，命令仍询问 | 高频小改动 |
| 计划模式 | plan | **只读**，只输出方案，不修改文件 | 复杂功能先设计 |
| 旁路 | bypass | 跳过权限请求 | 完全信任的自动化 |

> 💡 **黄金法则**：日常用**自动编辑**；开始复杂功能前务必先用**计划模式**确认架构，写错代码的重构成本远高于预先规划。

---

## 常用快捷键

| 快捷键 | 功能 |
|--------|------|
| `Shift+Tab` | 循环切换权限模式 |
| `Ctrl+R` | 最近会话历史 |
| `Ctrl+O` | 最近文件历史 |
| `Ctrl+B` | 打开 Bash 子命令（输入 `!`） |
| `Ctrl+D` | 退出会话 |
| `Ctrl+C` | 中断当前操作 |
| `?` | 查看所有快捷键帮助 |
| `Esc` | 中断/进入计划模式，进入时可选"只读"计划 |
| `-` | 折叠/展开最近工具调用 |
| `]` / `[` | 在工具调用之间跳转 |

> ⚠️ 注意：在 macOS Terminal 等应用里 `Ctrl+D` 可能有冲突，建议在设置里调整或使用 `/exit`。

---

## 斜杠命令速查

输入 `/` 会列出全部命令（含你安装的自定义 Skills）。常用命令：

| 命令 | 功能 |
|------|------|
| `/help` | 显示所有可用命令 |
| `/clear` | 清空当前对话上下文（切换任务用） |
| `/compact [保留内容]` | 压缩对话历史，保留关键决策 |
| `/context` | 查看当前 Token 使用量 |
| `/status` | 显示当前会话状态 |
| `/cost` | 显示 Token 使用量和费用 |
| `/login` | 切换认证方式 |
| `/model` | 切换 AI 模型 |
| `/mode` | 切换权限/交互模式 |
| `/init` | 为项目初始化 CLAUDE.md |
| `/agents` | 查看和管理子代理 |
| `/skills` | 查看已安装的 Skills |
| `/mcp` | 管理 MCP 连接（见 [[5-CC-MCP与外部集成]]） |
| `/hooks` | 查看钩子配置（见 [[6-CC-Hooks与自动化]]） |
| `/review` | 审查最近的代码变更 |
| `/pr-comments` | 处理 PR 审查评论 |
| `/doctor` | 诊断常见问题 |
| `/teleport` | 将 Web/移动端会话拉到终端 |
| `/desktop` | 将会话转移到桌面应用 |
| `/add-dir` | 给会话增加工作目录 |
| `/statusline` | 配置状态栏 |
| `/diff` | 可视化 diff |
| `/feedback` | 报告 Bug（别名 `/bug`） |
| `/exit` | 退出 Claude Code |

按需才用的命令：`/install-github-app`（GitHub Actions 集成）、`/bashes`（查看后台任务）、`/status` 等。

---

## 常用 CLI 参数

```bash
claude -p "提示词"                    # 非交互单次执行（print 模式）
claude -p "提示词" --output-format json  # JSON 输出
claude --resume <session_id>          # 恢复指定会话
claude --model <model>                # 指定模型
claude --settings <path>              # 指定 settings 文件
claude --allowedTools "Bash(npm run test:*)"  # 预授权工具
claude --permission-mode acceptEdits  # 默认权限模式
```

---

## 非交互模式（Headless / CI）

`-p`（print）模式用于脚本和 CI/CD，无人值守执行：

```bash
# 单次任务，输出到 stdout
claude -p "给 README.md 补充安装说明"

# 与 CI 结合
claude -p "跑测试并修复失败" --allowedTools "Bash(npm run test:*)"

# 配合 stdin / 输出重定向
echo "总结最近的 git 提交" | claude -p --output-format json > result.json
```

在 CI/CD 中配合权限白名单、**Hooks**（见 [[6-CC-Hooks与自动化]]）和 `--output-format` 实现无人值守流水线。

---

## 会话管理

### 会话概念

每个会话（session）对应一次独立的上下文窗口。切换任务时若不清理，旧内容会占用 token 并干扰新任务。

### 常用操作

| 操作 | 命令 | 场景 |
|------|------|------|
| 查看会话列表 | `claude --list-sessions` | 找回历史会话 |
| 恢复会话 | `claude --resume` | 接续之前工作 |
| 清空上下文 | `/clear` | 切换全新任务 |
| 压缩上下文 | `/compact` | 保留关键信息并释放空间 |
| 查看用量 | `/context` | 定期检查成本 |

---

## 上下文与成本

**防止 Context Rot（记忆衰退）**：上下文窗口填满后 AI 推理能力显著下降。三个关键习惯：

1. **随时用 `/context` 查看 Token 用量**
2. **切换任务时 `/clear` 清空记忆**
3. **复杂任务前先用计划模式确认架构**——重构成本远高于预先规划

更多成本控制技巧见 [[7-CC-最佳实践与FAQ]] 与 [[3-CC-CLAUDE.md与上下文管理]]。

---

## 相关笔记

- 会话启动时加载的 CLAUDE.md 如何影响交互 → [[3-CC-CLAUDE.md与上下文管理]]
- 自定义命令本质是 Skills → [[4-CC-Skills与子代理]]
- 权限模型与安全 → [[7-CC-最佳实践与FAQ]]
