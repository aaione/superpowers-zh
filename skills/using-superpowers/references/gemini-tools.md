# Gemini CLI 工具映射 (Gemini CLI Tool Mapping)

Skills 以动作表述（"派发一个 subagent"、"创建一个 todo"、"读取一个文件"）。在 Gemini CLI 中，这些动作对应下面的工具。

| Skills 请求的动作 | Gemini CLI 等价物 |
|----------------------|----------------------|
| 读取一个文件 | `read_file` |
| 一次读取多个文件 | `read_many_files` |
| 创建一个新文件 | `write_file` |
| 编辑一个文件 | `replace` |
| 运行一条 shell 命令 | `run_shell_command` |
| 搜索文件内容 | `grep_search` |
| 按名称查找文件 | `glob` |
| 列出文件和子目录 | `list_directory` |
| 抓取一个 URL | `web_fetch` |
| 搜索网络 | `google_web_search` |
| 调用一个 skill | `activate_skill` |
| 派发一个 subagent（`Subagent (general-purpose):` 模板） | `invoke_agent`，设置 `agent_name: "generalist"`（可通过 `@generalist` 聊天语法调用——参见 [Subagent 支持](#subagent-support)） |
| 多个并行派发 | 在同一响应中进行多次 `invoke_agent` 调用 |
| 任务跟踪（"创建一个 todo"、"标记完成"） | `write_todos`（状态：pending、in_progress、completed、cancelled、blocked） |

## 指令文件

当某个 skill 提及"你的指令文件"时，在 Gemini CLI 中它是 **`GEMINI.md`**。Gemini CLI 以分层方式加载 `GEMINI.md`：全局的 `~/.gemini/GEMINI.md`、工作区目录及其祖先中的项目级文件，以及当工具访问那些目录中的文件时加载的子目录 `GEMINI.md` 文件。

## 个人 skills 目录

用户级 skills 位于 **`~/.gemini/skills/`**，并以 **`~/.agents/skills/`** 作为跨运行时别名（与 Codex 和 Copilot CLI 共享）。当两个目录在同一作用域同时存在时，`.agents/skills/` 具有优先权。每个 skill 是一个子目录，包含一个 `SKILL.md`（带有 `name` 和 `description` frontmatter）。

## Subagent 支持

Gemini CLI 通过 `invoke_agent` 工具派发 subagent，该工具接受 `agent_name` 和 `prompt` 参数。同样的派发也以聊天语法快捷方式暴露：输入 `@generalist <prompt>` 等价于以 `agent_name: "generalist"` 调用 `invoke_agent`。内置 agent 名称包括 `generalist`、`cli_help`、`codebase_investigator`，以及（在启用浏览器工具的情况下）`browser_agent`。

Skills 使用 `Subagent (general-purpose):` 派发，要么引用一个 prompt 模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`），要么提供内联 prompt。在 Gemini CLI 上：

| Skill 派发形式 | Gemini CLI 等价物 |
|---------------------|----------------------|
| 引用一个 `*-prompt.md` 模板（implementer、task-reviewer、code-reviewer 等） | 填充模板，然后用 `invoke_agent` 调用，设置 `agent_name: "generalist"` 并传入填充好的 prompt |
| 引用 `superpowers:requesting-code-review` 的 `./code-reviewer.md` | 用 `invoke_agent` 调用，设置 `agent_name: "generalist"` 并传入填充好的审查模板 |
| 内联 prompt（未引用模板） | 用 `invoke_agent` 调用，设置 `agent_name: "generalist"` 并传入你的内联 prompt |

### 填充 prompt

Skills 提供带有 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]` 等占位符的 prompt 模板。在将完整 prompt 传给 `invoke_agent` 之前填充所有占位符。prompt 模板本身包含 agent 的角色、审查标准和期望输出格式——subagent 会遵循它。

### 并行派发

Gemini CLI 支持并行 subagent 派发。在同一响应中发出多次 `invoke_agent` 调用（或在一条 prompt 中进行多次 `@generalist` 调用），以并行方式运行独立的 subagent 工作。保持依赖任务串行，但不要为了保留更简单的历史而把独立的 subagent 任务串行化。

## 其他 Gemini CLI 工具

这些工具是 Gemini CLI 特有的：

| 工具 | 用途 |
|------|---------|
| `save_memory`（遗留） | 当 `experimental.memoryV2 = false` 时跨会话持久化事实 |
| `get_internal_docs` | 查阅 Gemini CLI 的内置文档 |
| `ask_user` | 向用户提出结构化问题（文本 / 单选 / 多选） |
| `enter_plan_mode` / `exit_plan_mode` | 进入和退出只读 plan mode |
| `update_topic` | 更新当前对话的主题 / 战略意图元数据 |
| `complete_task` | 通知一个 Gemini subagent 已完成并将其结果返回给父 agent |
| `tracker_create_task`、`tracker_update_task`、`tracker_get_task`、`tracker_list_tasks`、`tracker_add_dependency`、`tracker_visualize` | 支持依赖关系和可视化的富任务跟踪器 |
| `read_mcp_resource`、`list_mcp_resources` | MCP 资源访问 |
