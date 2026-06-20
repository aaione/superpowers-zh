# Claude Code 工具映射 (Claude Code Tool Mapping)

Skills 以动作表述（"派发一个 subagent"、"创建一个 todo"、"读取一个文件"）。在 Claude Code 中，这些动作对应下面的工具。

## 工具

| Skills 请求的动作 | Claude Code 工具 |
|----------------------|------------------|
| 读取一个文件 | `Read` |
| 创建一个新文件 | `Write` |
| 编辑一个文件 | `Edit` |
| 运行一条 shell 命令 | `Bash` |
| 搜索文件内容 | `Grep` |
| 按名称查找文件 | `Glob` |
| 抓取一个 URL | `WebFetch` |
| 搜索网络 | `WebSearch` |
| 调用一个 skill | `Skill` |
| 派发一个 subagent（`Subagent (general-purpose):` 模板） | `Agent`（较早的版本中名为 `Task`） |
| 多个并行派发 | 在一次响应中进行多次 `Agent` 调用 |
| 任务跟踪（"创建一个 todo"、"标记完成"） | `TaskCreate`、`TaskUpdate`、`TaskList`、`TaskGet`；在 `claude -p` / Agent SDK 中为 `TodoWrite`，除非设置了 `CLAUDE_CODE_ENABLE_TASKS=1` |
| 后台进程 / subagent 生命周期（读取输出、取消） | `TaskOutput`、`TaskStop`——这些与上面的 todo 工具不同，适用于运行中的 shell、agent 和远程会话 |

## 指令文件

当某个 skill 提及"你的指令文件"时，在 Claude Code 中它是 **`CLAUDE.md`**。Claude Code 会从当前工作目录沿目录树向上查找，并把沿途找到的每个 `CLAUDE.md` 和 `CLAUDE.local.md` 拼接起来。标准位置：

| 范围 | 位置 |
|-------|----------|
| 项目（团队共享） | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` |
| 用户全局 | `~/.claude/CLAUDE.md` |
| 本地私有（被 gitignore） | `./CLAUDE.local.md` |
| 托管策略（组织范围） | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS）、`/etc/claude-code/CLAUDE.md`（Linux/WSL）、`C:\Program Files\ClaudeCode\CLAUDE.md`（Windows） |

CLAUDE.md 文件可以通过 `@path/to/file` 导入引入额外内容（相对路径或绝对路径，最深五跳）。子目录中的 `CLAUDE.md` 文件也会被自动发现，并在 Claude Code 读取那些子目录中的文件时按需加载。

Claude Code **不会**直接读取 `AGENTS.md`。如果某个项目已经为其他 agent 维护了 `AGENTS.md`，请从 `CLAUDE.md` 中导入它，这样两个运行时就能共享相同的指令：

```markdown
@AGENTS.md

## Claude Code

（Claude Code 特定的指令放在这里。）
```

关于路径作用域规则和大型项目的组织方式，参见 `.claude/rules/`（规则可通过 `paths` frontmatter 限定到特定文件，并按需加载）。

## 个人 skills 目录

用户级 skills 位于 **`~/.claude/skills/`**。每个 skill 是一个子目录，包含一个 `SKILL.md`（带有 `name` 和 `description` frontmatter）以及任何支持文件。Claude Code 目前不识别 Codex、Copilot CLI 和 Gemini CLI 读取的跨运行时路径 `~/.agents/skills/`；如果你未来打算依赖跨运行时支持，请对照[官方 skills 文档](https://code.claude.com/docs/en/skills)核实。
