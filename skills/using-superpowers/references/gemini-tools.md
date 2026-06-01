# Gemini CLI 工具映射

Skills 使用 Claude Code 工具名称。当你在 skill 中遇到这些时，使用你的平台等价物：

| Skill 引用 | Gemini CLI 等价物 |
|-----------------|----------------------|
| `Read`（文件读取） | `read_file` |
| `Write`（文件创建） | `write_file` |
| `Edit`（文件编辑） | `replace` |
| `Bash`（运行命令） | `run_shell_command` |
| `Grep`（搜索文件内容） | `grep_search` |
| `Glob`（按名称搜索文件） | `glob` |
| `TodoWrite`（任务跟踪） | `write_todos` |
| `Skill` 工具（调用 skill） | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（dispatch subagent） | `@agent-name`（见 [Subagent 支持](#subagent-support)） |

## Subagent 支持

Gemini CLI 通过 `@` 语法原生支持 subagents。使用内置的 `@generalist` agent 来 dispatch 任何任务 — 它可以访问所有工具并遵循你提供的 prompt。

当 skill 说要 dispatch 命名的 agent 类型时，使用 `@generalist` 并提供 skill prompt 模板中的完整 prompt：

| Skill 指令 | Gemini CLI 等价物 |
|-------------------|----------------------|
| `Task tool (superpowers:implementer)` | `@generalist` 与填充的 `implementer-prompt.md` 模板 |
| `Task tool (superpowers:spec-reviewer)` | `@generalist` 与填充的 `spec-reviewer-prompt.md` 模板 |
| `Task tool (superpowers:code-reviewer)` | `@code-reviewer`（bundled agent）或 `@generalist` 与填充的审查 prompt |
| `Task tool (superpowers:code-quality-reviewer)` | `@generalist` 与填充的 `code-quality-reviewer-prompt.md` 模板 |
| `Task tool (general-purpose)` 与内联 prompt | `@generalist` 与你的内联 prompt |

### Prompt 填充

Skills 提供带有占位符如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]` 的 prompt 模板。填充所有占位符并将完整 prompt 作为消息传递给 `@generalist`。Prompt 模板本身包含 agent 的角色、审查标准和预期的输出格式 — `@generalist` 将遵循它。

### 并行 dispatch

Gemini CLI 支持并行 subagent dispatch。当 skill 要求你在并行中 dispatch 多个独立的 subagent 任务时，在同一 prompt 中一起请求所有那些 `@generalist` 或命名 subagent 任务。保持依赖任务顺序，但不要仅为了保留更简单的历史而序列化独立的 subagent 任务。

## 额外的 Gemini CLI 工具

这些工具在 Gemini CLI 中可用但没有 Claude Code 等价物：

| 工具 | 目的 |
|------|---------|
| `list_directory` | 列出文件和子目录 |
| `save_memory` | 将事实持久化到 GEMINI.md 跨会话 |
| `ask_user` | 请求用户的结构化输入 |
| `tracker_create_task` | 丰富的任务管理（创建、更新、列出、可视化） |
| `enter_plan_mode` / `exit_plan_mode` | 在进行更改之前切换到只读研究模式 |
