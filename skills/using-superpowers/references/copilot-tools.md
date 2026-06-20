# Copilot CLI 工具映射 (Copilot CLI Tool Mapping)

Skills 以动作表述（"派发一个 subagent"、"创建一个 todo"、"读取一个文件"）。在 Copilot CLI 中，这些动作对应下面的工具。

| Skills 请求的动作 | Copilot CLI 等价物 |
|----------------------|----------------------|
| 读取一个文件 | `view` |
| 创建 / 编辑 / 删除一个文件 | `apply_patch`（Copilot CLI 没有独立的创建/编辑/写入工具） |
| 运行一条 shell 命令 | `bash` |
| 搜索文件内容 | `rg`（ripgrep；Copilot CLI 不暴露 `grep` 工具） |
| 按名称查找文件 | `glob` |
| 抓取一个 URL | `web_fetch` |
| 搜索网络 | `web_search` |
| 调用一个 skill | `skill` |
| 派发一个 subagent（`Subagent (general-purpose):` 模板） | `task`，设置 `agent_type: "general-purpose"`（其他可接受的类型：`explore`、`task`、`code-review`、`research`、`configure-copilot`） |
| 多个并行派发 | 在一次响应中进行多次 `task` 调用 |
| Subagent 状态/输出/控制 | `read_agent`、`list_agents`、`write_agent` |
| 任务跟踪（"创建一个 todo"、"标记完成"） | `update_todo` |
| 进入 / 退出 plan mode | 没有等价物——留在主会话中 |

## 指令文件

当某个 skill 提及"你的指令文件"时，在 Copilot CLI 中它是位于仓库根目录的 **`AGENTS.md`**。如果同时存在 `AGENTS.md` 和 `.github/copilot-instructions.md`，Copilot 会同时读取两者。

## 个人 skills 目录

用户级 skills 位于 **`~/.copilot/skills/`**。Copilot CLI 还识别跨运行时别名 **`~/.agents/skills/`**，它与 Codex 和 Gemini CLI 共享。每个 skill 是一个子目录，包含一个 `SKILL.md`（带有 `name` 和 `description` frontmatter）。

## 异步 shell 会话

Copilot CLI 支持持久的异步 shell 会话：

| 工具 | 用途 |
|------|---------|
| `bash`，设置 `mode: "async"`（并可选 `detach: true`） | 在后台启动一条长时间运行的命令；返回一个 `shellId` |
| `write_bash` | 向运行中的异步会话发送输入 |
| `read_bash` | 从异步会话读取输出 |
| `stop_bash` | 终止一个异步会话 |
| `list_bash` | 列出所有活动的 shell 会话 |

## 其他 Copilot CLI 工具

| 工具 | 用途 |
|------|---------|
| `store_memory` | 为未来的会话持久化关于代码库的事实 |
| `report_intent` | 用当前意图更新 UI 状态栏 |
| `sql` | 查询会话的 SQLite 数据库（todos、元数据） |
| `fetch_copilot_cli_documentation` | 查阅 Copilot CLI 文档 |
| GitHub MCP 工具（`github-mcp-server-*`） | 原生 GitHub API 访问（issues、PRs、代码搜索） |
