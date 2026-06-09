# MCP 应用安装指南

MCP（Model Context Protocol）是 Anthropic 推出的开放协议，允许 AI 客户端通过标准接口调用外部工具。Claude Code 原生支持 MCP，安装后 Claude 可以直接操作数据库、浏览器、GitHub、代码图谱等外部服务。

## 安装方式

Claude Code 支持两种 MCP 配置方式。

### 方式一：CLI 命令（推荐）

```bash
# stdio 类型（最常见，本地进程）
claude mcp add <名称> -- <命令> [参数...]

# 示例：添加 filesystem MCP
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /your/path

# SSE / HTTP 类型（远程服务）
claude mcp add <名称> --transport sse --url http://localhost:3000/sse
```

`claude mcp add` 默认写入全局配置 `~/.claude/mcp.json`，加 `--scope project` 写入当前项目的 `.mcp.json`。

管理命令：

```bash
claude mcp list          # 列出已配置的 MCP
claude mcp remove <名称>  # 删除
claude mcp get <名称>     # 查看详情
```

### 方式二：直接编辑 .mcp.json

适合批量配置或团队共享。项目级配置放在 `.mcp.json`（提交到仓库），全局配置放在 `~/.claude/mcp.json`。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

编辑后在 Claude Code 内执行 `/mcp` 或重启客户端生效。

### 控制项目 MCP 的启用

`settings.json` 中可控制哪些项目 MCP 自动启用：

```json
{
  "enableAllProjectMcpServers": true,
  "enabledMcpjsonServers": ["github", "postgres"],
  "disabledMcpjsonServers": ["unused-server"]
}
```

---

## 实用 MCP 推荐

### 官方 MCP 包（`@modelcontextprotocol`）

| MCP | 用途 | 安装命令 |
|-----|------|---------|
| **filesystem** | 读写本地文件系统，突破项目目录限制 | `npx @modelcontextprotocol/server-filesystem <路径>` |
| **github** | 操作 PR、Issue、仓库、搜索代码 | `npx @modelcontextprotocol/server-github` |
| **postgres** | 直接查询 PostgreSQL，无需写 SQL 工具 | `npx @modelcontextprotocol/server-postgres <连接串>` |
| **sqlite** | 本地 SQLite 操作 | `npx @modelcontextprotocol/server-sqlite <db路径>` |
| **puppeteer** | 控制 Chrome 浏览器，截图、点击、抓取 | `npx @modelcontextprotocol/server-puppeteer` |
| **fetch** | 抓取任意 URL 内容（绕过内置限制） | `npx @modelcontextprotocol/server-fetch` |
| **memory** | 持久化键值记忆，跨会话保留上下文 | `npx @modelcontextprotocol/server-memory` |

### 第三方精选

| MCP | 用途 | 来源 |
|-----|------|------|
| **sequential-thinking** | 结构化多步推理，适合复杂决策 | `npx @modelcontextprotocol/server-sequential-thinking` |
| **Playwright** | 浏览器自动化，比 Puppeteer 更强 | `npx @playwright/mcp` |
| **Linear** | 读写 Linear 任务和项目 | `@linear/mcp-server` |
| **Slack** | 读取 Slack 频道消息、发消息 | `@modelcontextprotocol/server-slack` |
| **CodeGraph** | 语义代码图谱，减少 token 消耗 | 见下方专项说明 |

---

## CodeGraph 安装

CodeGraph 是一个本地代码语义图谱 MCP，通过预建索引让 Claude 以更少的 token 理解代码结构。官方数据：减少约 58% 的工具调用，降低约 16% 的费用。

### 1. 安装 CLI

```bash
# macOS / Linux（无需 Node.js）
curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh

# 已有 Node.js
npm i -g @colbymchenry/codegraph
```

安装完后**开一个新终端**，旧终端的 PATH 还没有 `codegraph`。

### 2. 连接到 Claude Code（全局，一次）

```bash
claude mcp add codegraph -- codegraph mcp
```

这一步把 CodeGraph MCP server 写入 `~/.claude/mcp.json`，Claude Code 重启后生效。

如果你的项目 graph.db 不在默认位置，可以指定路径：

```bash
claude mcp add codegraph -- codegraph mcp --db /path/to/your-project/.codegraph/graph.db
```

### 3. 为每个项目建立索引

```bash
cd your-project
codegraph build         # 解析代码并生成 .codegraph/graph.db
```

索引保存在 `.codegraph/` 目录，建议加入 `.gitignore`。后续代码变更可用 `codegraph watch` 增量更新。

### 升级与卸载

```bash
# 升级（npm 安装方式）
npm update -g @colbymchenry/codegraph

# 移除 MCP 配置
claude mcp remove codegraph

# 删除当前项目的索引
rm -rf .codegraph/
```

---

## 选择原则

- 需要操作外部服务（GitHub、数据库、浏览器）：装对应官方 MCP，按需配置 token。
- 需要代码理解加速（大型项目、降低费用）：装 CodeGraph，每个项目跑一次 `codegraph init -i`。
- 团队共享配置：用项目级 `.mcp.json` 提交到仓库，个人 token 放 `env` 字段由各自环境变量注入。
