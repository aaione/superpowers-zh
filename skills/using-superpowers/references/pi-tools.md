# Pi 工具映射 (Pi Tool Mapping)

Skills 以动作表述（"派发一个 subagent"、"创建一个 todo"、"读取一个文件"）。在 Pi 中，这些动作对应下面的工具。

| Skills 请求的动作 | Pi 等价物 |
| --- | --- |
| 调用一个 skill | Pi 原生 skills：用 `read` 加载相关 `SKILL.md`，或让人类使用 `/skill:name` |
| 读取一个文件 | `read` |
| 创建一个文件 | `write` |
| 编辑一个文件 | `edit` |
| 运行一条 shell 命令 | `bash` |
| 搜索文件内容 | 活跃时用 `grep`；否则用 `bash` 配合 `rg`/`grep` |
| 按名称查找文件 | `find` 或用 `bash` 配合 shell glob |
| 列出文件和子目录 | 活跃时用 `ls`；否则用 `bash` 配合 `ls` |
| 派发一个 subagent（`Subagent (general-purpose):` 模板） | 如果可用，使用一个已安装的 subagent 工具，例如 `pi-subagents` 的 `subagent` |
| 任务跟踪（"创建一个 todo"、"标记完成"） | 如果可用，使用一个已安装的 todo/task 工具，否则在计划或 `TODO.md` 中跟踪任务 |

## Skills

Pi 从已配置的 skill 目录和已安装的 Pi 包中发现 skills。一个 Superpowers Pi 包应通过其 `pi.skills` manifest 条目暴露 `skills/`。Pi 不暴露 Claude Code 的 `Skill` 工具，但 agent 仍应遵循 Superpowers 规则：当某个 skill 适用时，在响应之前加载并遵循它。

## Subagents

Pi 核心不附带标准的 subagent 工具。`pi-subagents` 包是一个强有力的可选配套，提供了一个 `subagent` 工具，支持单 agent、链式、并行、异步、分叉上下文以及恢复/状态等工作流。如果没有可用的 subagent 工具，不要编造 `Task` 调用；在当前会话中串行执行，或说明可选的 subagent 能力未安装。

## 任务列表

Pi 核心不附带标准的任务列表工具。如果安装了 todo/task 扩展，使用其文档化的工具。否则使用 Superpowers 计划文件、Markdown 检查清单或仓库本地的 `TODO.md` 进行任务跟踪。较旧的 Superpowers 文档可能提到 `TodoWrite`；请将其视为上面的任务跟踪动作。
