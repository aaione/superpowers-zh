# Copilot CLI 工具映射

Skills 使用 Claude Code 工具名称。当你在 skill 中遇到这些时，使用你的平台等价物：

| Skill 引用 | Copilot CLI 等价物 |
|-----------------|----------------------|
| `Read`（文件读取） | `view` |
| `Write`（文件创建） | `create` |
| `Edit`（文件编辑） | `edit` |
| `Bash`（运行命令） | `bash` |
| `Grep`（搜索文件内容） | `grep` |
| `Glob`（按名称搜索文件） | `glob` |
| `Skill` 工具（调用 skill） | `skill` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（dispatch subagent） | `task` 与 `agent_type: "general-purpose"` 或 `"explore"` |
| 多个 `Task` 调用（并行） | 多个 `task` 调用 |
| Task 状态/输出 | `read_agent`、`list_agents` |
| `TodoWrite`（任务跟踪） | `sql` 与内置 `todos` 表 |
| `WebSearch` | 无等价物 — 使用 `web_fetch` 与搜索引擎 URL |
| `EnterPlanMode` / `ExitPlanMode` | 无等价物 — 留在主会话中 |

## 异步 shell 会话

Copilot CLI 支持持持久的异步 shell 会话，没有直接的 Claude Code 等价物：

| 工具 | 目的 |
|------|---------|
| `bash` 与 `async: true` | 在后台启动长时间运行的命令 |
| `write_bash` | 向运行的异步会话发送输入 |
| `read_bash` | 从异步会话读取输出 |
| `stop_bash` | 终止异步会话 |
| `list_bash` | 列出所有活动的 shell 会话 |

## 额外的 Copilot CLI 工具

| 工具 | 目的 |
|------|---------|
| `store_memory` | 将有关 codebase 的事实持久化以供将来会话使用 |
| `report_intent` | 使用当前意图更新 UI 状态行 |
| `sql` | 查询会话的 SQLite 数据库（todos、元数据） |
| `fetch_copilot_cli_documentation` | 查找 Copilot CLI 文档 |
| GitHub MCP 工具（`github-mcp-server-*`） | 原生 GitHub API 访问（issues、PRs、代码搜索） |
