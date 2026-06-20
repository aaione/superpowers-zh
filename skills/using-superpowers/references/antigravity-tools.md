# Antigravity CLI (`agy`) 工具映射 (Antigravity CLI (`agy`) Tool Mapping)

Skills 以动作表述（"派发一个 subagent"、"创建一个 todo"、"读取一个文件"）。在 Antigravity CLI（`agy`）中，这些动作对应下面的工具。

| Skills 请求的动作 | Antigravity CLI 等价物 |
|----------------------|----------------------|
| 读取一个文件 | `view_file` |
| 创建一个新文件 | `write_to_file` |
| 编辑一个文件 | `replace_file_content` |
| 同时在多处编辑一个文件 | `multi_replace_file_content` |
| 运行一条 shell 命令 | `run_command` |
| 搜索文件内容 | `grep_search` |
| 按名称查找文件 / 列出目录 | `list_dir`（没有专用的 glob 工具——将 `list_dir` 与 `grep_search` 结合使用） |
| 抓取一个 URL | `read_url_content` |
| 搜索网络 | `search_web` |
| 向你的人类伙伴提出一个结构化问题 | `ask_question` |
| 派发一个 subagent（`Subagent (general-purpose):` 模板） | `invoke_subagent`，使用内置的 `TypeName`——`self` 用于全能力工作，`research` 用于只读（参见 [Subagent 支持](#subagent-support)） |
| 多个并行派发 | 在一次 `invoke_subagent` 调用的 `Subagents` 数组中放入多个条目 |
| 任务跟踪（"创建一个 todo"、"标记完成"） | 一个 **task artifact**——以 `write_to_file` 写出，设置 `IsArtifact: true` 且 `ArtifactType: "task"`（参见[任务跟踪](#task-tracking)）。**不是** `manage_task`，后者用于管理后台进程。 |

## 调用一个 skill——读取其 `SKILL.md`

Antigravity 在每次会话开始时会向你呈现每个已安装 skill 的 `name` + `description`，但它**没有 `Skill`/`activate_skill` 工具**。要加载一个
skill，**用 `view_file` 读取它的 `SKILL.md`，并在该 skill 适用时设置 `IsSkillFile: true`**——例如对
`.../plugins/superpowers/skills/<skill-name>/SKILL.md` 执行 `view_file` 并设置 `IsSkillFile: true`。
（`IsSkillFile` 是 agy 自己的信号，表示你在读取一个文件以*执行其指令*，而非编辑或预览它——只要加载 skill 就应设置它。）

这是该 harness 上官方认可的 skill 加载机制。通用规则
"绝不手动读取 skill 文件" 的含义是 "不要绕过你所在平台的
skill 加载机制"——而在 Antigravity 上，读取 `SKILL.md` *就是*那个
机制。读取它是在遵循该规则，而非破坏规则。

你已经知道存在哪些 skills 以及它们的用途：它们的名称和
描述在会话开始时就在你面前。当某个描述匹配你即将做的事情时，在行动之前先读取那个 skill 的 `SKILL.md`。

## Subagent 支持

Antigravity 通过 `invoke_subagent` 派发 subagent，在 `Subagents` 数组中为每个 subagent 传入一个
`TypeName`。有两个 `TypeName` 是**内置的**——可以直接使用，无需 `define_subagent`：

- **`self`**——你的完整克隆，拥有你所拥有的每一个工具（包括
  `write_to_file`/`replace_file_content`/`run_command`）。通用工作的安全默认选择：
  实现、修复、任何编辑文件或运行命令的事情。
- **`research`**——只读（读取文件、`grep_search`、web/URL 抓取；没有写
  或命令权限）。当你明确想要一个无法做出更改的 subagent 时使用它——调查和只读审查。

仅当需要自定义系统 prompt 或能力组合时才调用 `define_subagent`：设置
`enable_write_tools: true` 以授予文件编辑**以及** `run_command`，
`enable_subagent_tools` 用于嵌套派发，`enable_mcp_tools` 用于 MCP。然后
以你赋予的名称调用它。（`manage_subagents` 列出/终止运行中的
subagent。）

Skills 使用 `Subagent (general-purpose):` 派发，要么引用一个
prompt 模板文件（例如 `superpowers:subagent-driven-development` 的
`./implementer-prompt.md`），要么提供内联 prompt。在 Antigravity 上：

| Skill 派发形式 | Antigravity 等价物 |
|---------------------|----------------------|
| 一个实现者风格的 `*-prompt.md` 模板（编写代码、运行测试） | 填充模板，然后用 `invoke_subagent` 调用，设置 `TypeName: "self"` 并传入填充好的 prompt |
| 一个只读审查者模板（`task-reviewer`、`code-reviewer`、`requesting-code-review` 的 `./code-reviewer.md`） | 用 `invoke_subagent` 调用，设置 `TypeName: "research"` 并传入填充好的审查模板 |
| 内联 prompt（未引用模板） | 用 `invoke_subagent` 调用，设置 `TypeName: "self"`（或如果任务只读取则用 `"research"`）并传入你的内联 prompt |

### 填充 prompt

Skills 提供带有 `{WHAT_WAS_IMPLEMENTED}` 或
`[FULL TEXT of task]` 等占位符的 prompt 模板。在将完整 prompt 传给
`invoke_subagent` 之前填充所有占位符。prompt 模板本身包含 agent 的角色、审查
标准和期望输出格式——subagent 会遵循它。

### 并行派发

在单次 `invoke_subagent` 调用的 `Subagents` 数组中放入多个条目，以并行方式运行
独立的 subagent 工作。保持依赖任务串行，但不要
为了保留更简单的历史而把独立的 subagent 任务串行化。

## 任务跟踪

Antigravity **没有 todo / `TodoWrite` 工具**（`manage_task` 管理后台
进程——`list`/`kill`/`status`/`send_input`——它*不是*检查清单）。当
某个 skill 要求创建一个 todo 列表或跟踪任务时，维护一个 **task artifact**：一个
markdown 检查清单，用 `write_to_file` 保存（`IsArtifact: true`，
`ArtifactMetadata.ArtifactType: "task"`），并随进展用 `replace_file_content` /
`multi_replace_file_content` 编辑。

在任何多步任务开始时，创建列出计划中每一步的 task artifact。每完成一步，就编辑该 artifact 将其标记为完成（`- [x]`）。
如果计划发生变化，更新检查清单。保持其最新——它是你剩余工作的真相来源；一旦
对话变长，在开始每一步之前重新读取它。
