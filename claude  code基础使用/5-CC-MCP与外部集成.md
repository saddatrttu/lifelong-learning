# 5. MCP 与外部集成

> MCP（Model Context Protocol，模型上下文协议）是 Anthropic 提出的开放标准，让 Claude Code 通过统一协议连接外部工具、数据源和企业系统——数据库、设计工具、项目管理、浏览器自动化等。

---

## 目录

1. [MCP 是什么](#mcp-是什么)
2. [工作原理](#工作原理)
3. [配置方式](#配置方式)
4. [会话内管理](#会话内管理)
5. [常用 MCP 服务器](#常用-mcp-服务器)
6. [MCP vs Skills](#mcp-vs-skills)
7. [安全注意事项](#安全注意事项)
8. [最佳实践](#最佳实践)

---

## MCP 是什么

MCP 是一个**开源标准协议**，让 Claude Code 能连接到本地文件系统之外的外部工具和数据源。每个 MCP 服务器是一个独立进程，把一组工具和数据源提供给 Claude 使用：

- 数据库（PostgreSQL、SQLite 等）
- 设计工具（Figma）
- 项目管理（Jira、Linear、GitHub Issues）
- 团队协作（Slack、Notion）
- 浏览器自动化
- 企业内部系统

没有 MCP，Claude 无法查询你的数据库或发布到 Slack；有了它，Agent 才能真正"动手"。

---

## 工作原理

```
Claude Code ──MCP 协议──> MCP Server（独立进程）
                              ├── 工具（Tools）如：查询数据库、读 PR
                              └── 数据源（Resources）
```

- 每个 MCP 服务器以**你的用户账号权限**运行
- 会话内用 `/mcp` 管理连接
- Claude 使用任何 MCP 工具前会请求权限，可选"总是允许"或"本次询问"

---

## 配置方式

### 用户级（settings.json）

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "your_token" }
    }
  }
}
```

### 项目级（.mcp.json）

团队共享的 MCP 配置放项目根目录 `.mcp.json`，随仓库一起 commit：

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgres://..." }
    }
  }
}
```

> ⚠️ **不要把敏感值直接写在 .mcp.json**——用环境变量引用。

---

## 会话内管理

| 操作 | 命令 |
|------|------|
| 查看/管理连接 | `/mcp` |
| 查看全部命令 | `/help` |

---

## 常用 MCP 服务器

| 服务器 | 用途 |
|--------|------|
| GitHub | 读取 PR、提交 Review Comments、创建 Issue |
| PostgreSQL | 查询数据库、分析数据 |
| Slack | 发消息、读频道 |
| Notion | 文档管理 |
| Figma | 设计稿读取 |
| Jira / Linear | 项目管理 |

> 💡 优先使用**官方或社区维护**的 MCP 服务器，降低安全风险。

---

## MCP vs Skills

两者是**互补**关系，不是替代：

| 维度 | MCP | Skills |
|------|-----|--------|
| 它是什么 | 连接外部服务的协议 | 知识、工作流和参考材料 |
| 提供 | 工具和数据访问 | 知识、工作流 |
| 示例 | Slack 集成、数据库查询 | 代码审查清单、部署流程 |

- **MCP 给予 Claude 与外部系统交互的能力**
- **Skill 给予 Claude 如何有效使用这些工具的知识**（如团队数据库架构、消息格式规则）

最佳组合：**MCP 配 Skills**——接上外部工具，再按固定流程处理。

---

## 安全注意事项

1. MCP 服务器以**你的用户账号权限**运行，小心处理数据库访问令牌和 API 密钥
2. 敏感值放**环境变量**，不写进配置文件
3. **按需接入**，不要一次连 20 个 Server——每个 MCP 都会把所有工具定义、参数描述完整加载到上下文，可能占 10,000–20,000 Tokens
4. **权限最小化**：只给 Agent 必要的访问权限
5. 大型 MCP 可能比整个对话历史还重——精简到真正需要的 2-3 个

---

## 最佳实践

1. 按需接入，控制数量（2-3 个大型 MCP 已是上限）
2. 权限最小化，只给必要访问
3. 优先官方/社区维护的服务器
4. 敏感值用环境变量，不用明文
5. 把"怎么用好 MCP 工具"沉淀成 [[4-CC-Skills与子代理|Skills]]
6. 用完及时用 `/mcp` 断开不用的连接，节省 token

---

## 相关笔记

- MCP 给 Skills 提供能力 → [[4-CC-Skills与子代理]]
- 上下文与 token 管理 → [[3-CC-CLAUDE.md与上下文管理]]
- 权限模型与安全 → [[7-CC-最佳实践与FAQ]]
- 用 Hooks 自动化校验 → [[6-CC-Hooks与自动化]]
