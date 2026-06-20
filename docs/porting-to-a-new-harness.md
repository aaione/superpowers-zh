# 将 Superpowers 移植到新的 Harness (Porting Superpowers to a New Harness)

本指南说明如何为一个新的 harness 添加支持——一个不是 Claude Code 的 IDE、CLI 或 agent 运行器——以便 Superpowers skills 能在那里像原生环境一样自动触发。

本指南分为两层。**第 1-3 部分**解释系统的工作原理以及如何判断一个 harness 是否能被支持；在动手之前请先阅读这些。**第 4-8 部分**是供一个 agent（在人类伙伴监督下）端到端执行移植、直至发布的具体步骤。附录索引了当前的参考集成，以便你复制最接近的那一个。

集成机制因 harness 而异，并且会持续变化。本指南有意教授那些**不变量（invariants）**——无论机制如何都必须成立的东西——并指向一个活的参考实现供你复制。当本指南与代码出现分歧时，以代码为准；修复指南。

## 开始之前

添加一个 harness 是本仓库中风险最高的贡献类型。在编写任何内容之前：

- 完整阅读 `CLAUDE.md` 和 `.github/PULL_REQUEST_TEMPLATE.md`——贡献者规则和新 harness 的 PR 要求不是可选项。
- 搜索已打开**和已关闭**的 PR，看看是否有人曾尝试过为该 harness 移植。如果存在，请在自己动手之前先理解它为何停滞。

---

## 第 1 部分 — Superpowers 如何跨 harness 工作

Superpowers 的内容在任何地方都相同。每个 harness 之间变化的是那层薄薄的、负责将该内容交付给模型、并把指令翻译成 harness 原生工具的层。三个组件：

1. **Skills（与 harness 无关）。** `skills/` 中的所有内容都是事实来源，被每个 harness 原样共享。Skills 以*动作*来描述——"调用一个 skill"、"读取一个文件"、"派发一个 subagent"、"创建一个 todo"——从不命名某个特定工具。正是这一点让同一个 skill 内容无需修改就能在 Claude Code、Codex、Gemini、pi 等上运行。

2. **工具映射（每个 harness 不同）。** 每个 harness 都需要把动作词汇翻译成其真实的工具名。该翻译位于 `skills/using-superpowers/references/<harness>-tools.md` 和/或内联在 harness 的 bootstrap 注入器中（见第 5 部分）。例如它会说，"*派发一个 subagent* → 调用 `task`，传入 `subagent_type`。"

3. **Bootstrap（每个 harness 不同）。** 在每个会话开始时，完整的 `skills/using-superpowers/SKILL.md` 被注入到模型的上下文中，包裹在 `<EXTREMELY_IMPORTANT>` 标签里，并附上工具映射。那个被注入的 skill 就是教会模型"skills 存在"以及"它必须在行动前检查是否有相关 skill"的东西。**bootstrap 就是整个集成。** 没有它，skill 文件就是惰性的——存在于磁盘上，但从不被调用。

### 让这一切工作的两条规则

**1. Skills 命名动作，而非工具。** 不要为了适配你的 harness 而修改 skill 内容。移植会增加一个工具映射参考和一个 bootstrap 注入器；它绝不会伸手到 `skills/*/SKILL.md` 里去替换工具名。（本项目的贡献者指南把 skill 内容当作经过精心调校的行为塑造代码；为"合规"而重写它会被直接拒绝。）

**2. 一切都通过 harness 自身的安装机制交付。绝不编辑用户的文件。** bootstrap、skills 和工具映射都作为*harness 所安装内容的一部分*交付——一个插件、一个扩展、一个市场条目、一个扩展内置的上下文文件。一次移植**绝不能**伸手到用户的全局或个人配置（`~/.gemini/config/AGENTS.md`、`settings.json`、`trustedFolders.json`、一个手工编辑的 `~/.bashrc` 等）去注入任何东西。harness 拥有它所加载的内容；你的安装产物是你唯一可以写的东西。如果安装机制确实无法承载 bootstrap，那是一个需要说明的限制（第 6 部分）——绝不是手工编辑用户配置的许可证。（Shape C *不是*例外：Gemini 的上下文文件没问题，因为它随*已安装的扩展*一起发布，并由 manifest 的 `contextFileName` 声明——harness 加载的是扩展自己的文件，而不是你在用户主目录里编辑的文件。）

---

## 第 2 部分 — 这个 harness 能被支持吗？

只有当一个 harness 能做到以下全部内容时，才能支持 Superpowers。在写代码之前检查这些——如果第一个就失败，停下。

### 硬性要求：自动的会话开始注入

该 harness 必须让你在**每个会话开始时、无需人类伙伴逐个会话地选择启用**就能把文本注入到模型的上下文中。这是唯一不可协商的能力。它可以采取任何形式：

- 一个在会话开始时运行 shell 命令并读取其 stdout 的 **hook/事件系统**（Claude Code、Codex、Cursor、Copilot CLI），或者
- 一个**进程内插件/扩展**，带有会话开始或消息生命周期回调，能修改消息数组（OpenCode、pi），或者
- 一个**指令文件**约定，harness 加载一个*由你已安装的扩展发布并声明*的上下文文件（例如 Gemini 的 `contextFileName` 指向扩展自己的 `GEMINI.md`）——而不是你在用户主目录里编辑的文件。

如果让 Superpowers 出现在模型面前的唯一方式是让人类伙伴每个会话都去启用（粘贴一个 prompt、运行一个命令、打开一个模式），那么这个 harness **无法**被妥善支持。第 3 部分的验收测试会失败，PR 会被关闭。这是一个"移植"并非真正移植的最常见原因。

### 其余能力清单

| 能力 | 为何需要 | 如果缺失 |
|---|---|---|
| **Skill 发现 + 调用** | 模型必须能按需加载一个 skill 的完整内容 | 如果没有原生 skill 工具，受认可的后备方案是直接 `read` 相关的 `SKILL.md`——见第 5 部分。一个既没有 skill 工具又不能读文件的 harness 无法工作。 |
| **文件读 / 写 / 编辑** | 几乎每个 skill 都会操作文件 | 必需。没有变通方案。 |
| **运行 shell 命令** | TDD、验证、git 工作流 | 必需。 |
| **Subagent / 任务派发** | `dispatching-parallel-agents`、`subagent-driven-development` | 可降级：如果不可用，这些特定 skill 会告诉模型内联完成工作或报告缺失的能力——*绝不*去捏造一个 `Task` 调用。某些 harness 将其置于一个配置开关之后（例如 Codex 需要启用 multi-agent）。 |
| **Todo / 任务追踪** | 若干 skill 中的进度追踪 | 可降级：回退到一个 plan 文件或 `TODO.md`。 |
| **网络获取 / 搜索** | 少数 skill | 可降级。 |
| **Shell 或 polyglot 脚本执行（Windows）** | 仅对 shell-hook 形态、且仅当你想要 Windows 支持时 | 见第 7 部分。进程内插件的 harness 可完全绕开这一点。 |

"可降级"意味着：该 skill 已经针对缺失工具有了后备措辞。你在工具映射中的工作是：当真实工具存在时指向它，当不存在时复用那段后备措辞。

### 你可能根本不需要一个新目录

某些"新 harness"实际上只是换了不同安装器的现有集成。例如 Factory 的 Droid 通过其自己的 `plugin install` 命令消费 Claude Code 插件，在此处不需要任何新文件。在构建之前，先检查该 harness 是否能直接加载一个现有的 manifest。一个在本仓库中除 README 的一段话之外什么都不添加的移植，是完全合格的成果。

---

## 第 3 部分 — 完成的定义

当一个移植的**全部**条件都成立时，它才算完成：

1. `using-superpowers` bootstrap 在每个会话开始时加载，无需逐个会话地启用。
2. 该 harness 存在一个工具映射（在 `references/<harness>-tools.md`、内联在 bootstrap 中，或两者兼有——按第 5 部分）。
3. Skills 能真正被调用——原生地，或通过文档记载的读 `SKILL.md` 后备方案——并且模型遵循它们。
4. **验收测试通过。** 在一个干净的会话中，用户消息：

   > Let's make a react todo list

   会在*编写任何代码之前*自动触发 `brainstorming` skill。捕获完整的会话记录——PR 要求这样做。
5. 测试覆盖该集成（第 5 部分）并通过。
6. 一个真实用户能通过 harness 自身的机制安装它（不是手工复制文件），且版本在适用时记录在 `.version-bump.json` 中（第 6 部分）。注意某些安装器在安装时会重写或剥离 manifest（其中一个把它精简成只剩 `{"name": …}`），所以"已安装的文件报告仓库版本"并不总是可实现——在源 manifest 处追踪版本，不要把被重写的已安装 manifest 当作失败。

在完整验收测试之前的一个快速冒烟检查：启动一个会话并让模型描述它的 superpowers。如果 bootstrap 被注入了，它会知道自己拥有它们。（OpenCode 的安装文档用 `opencode run --print-logs "hello" 2>&1 | grep -i superpowers` 通过不同机制达成同一目标——日志 grep，而非询问模型；`2>&1` 很重要，因为日志会输出到 stderr。找出你的 harness 的等价做法。）

---

## 第 4 部分 — 选择你的集成形态

有三种结构形态，按*你如何让 bootstrap 出现在模型面前*来区分。选择与你 harness 暴露的能力相匹配的那一种，然后复制那个参考实现。这个形态几乎决定了第 5 部分的一切——下面的步骤会根据它分叉。

### 如何判断你属于哪种形态

在路由之前，先了解 harness 的*实际*机制——不要假设它文档完备，也不要假设它的行为与它所 fork 自的某个 harness 相同。

**找到入口面：**

- **在网上搜索该 harness 的文档**（扩展 / 插件 / hook / skill / MCP / "context file" / "rules file"）。厂商工具变化很快；要搜索，而非依赖训练知识。
- **找到并阅读一个该 harness 的现成第三方扩展/插件。** 一个真实可用的例子胜过文档——它展示了 manifest 形态、安装命令，以及 harness 真正加载哪些组件。
- 检查 harness 在启动时加载什么：一个设置文件？一个扩展目录？一个项目级或全局的指令文件（`AGENTS.md`、`<NAME>.md`）？

**如果文档不足，就用经验方式逆向工程它**（一个真实的移植者不得不做过下面每一项）：

- 对二进制文件运行 `strings` / grep 安装目录树，查找 hook 事件名、配置路径以及它读取的指令文件。
- **让运行中的模型枚举它自己的工具名**——例如"列出你能调用的每个工具的精确机器名。"这是不凭空捏造获取工具名的权威方式（见步骤 4）。
- 用一个**唯一标记测试**证明每一个假设：通过你认为有效的机制注入一个无意义 token，启动一个全新会话，确认该 token 确实到达了模型。

**一个 fork 不会继承其父级的行为。** 一个派生自另一个的 harness（例如一个派生自 Gemini 的 CLI）可能暴露父级的 manifest 字段和 `@`-include 语法，却*仍然以不同方式处理它们*。用标记验证；绝不要假设父级的配方能照搬。

然后路由到一种形态：

- 会话开始时运行一个 shell 命令并读取其 stdout → **Shape A**。
- 一个你在其中运行代码、带生命周期回调的插件/扩展模块 → **Shape B**。
- 只有一个常驻的指令文件，没有 hook 也没有代码插件 → **Shape C**。

**各形态可组合——它们并非互斥。** *skill 发现*机制和 *bootstrap*机制不必是同一种形态——但**两者仍必须搭载安装机制**（规则 2）。分开决定这两个问题：*skills 在哪里被发现？*以及*bootstrap 如何在每个会话到达模型？*一个 harness 可能通过插件安装 skills，却需要 bootstrap 以另一种随安装发布的方式交付（一个扩展声明的上下文文件，或者——见下文——由 harness 在会话开始时展示已安装的 `using-superpowers` skill 自身的描述）。如果有不止一个安装机制入口能自动注入，优先选择最可靠的那个。你**不可**做的是通过编辑用户的全局配置来桥接一个缺口。

### Shape A — Shell-hook

该 harness 有一个 hook 系统，在会话开始时运行 shell 命令并从其 stdout 读取 JSON。配置的命令运行 `run-hook.cmd`，这是一个 polyglot 包装器，只负责定位 bash 并派发被命名的脚本；该脚本（`hooks/session-start`，或一个 harness 专用变体如 `hooks/session-start-codex`）负责读取 `using-superpowers/SKILL.md` 并打印一个 JSON 对象，其**字段名和嵌套结构因 harness 而异**。

- 参考：`hooks/session-start`（以及 `hooks/session-start-codex`）、`hooks/run-hook.cmd`，以及每个 harness 的 hook 配置 `hooks/hooks.json`（Claude Code）、`hooks/hooks-codex.json`（Codex）、`hooks/hooks-cursor.json`（Cursor）。
- Manifest：`.codex-plugin/plugin.json`、`.cursor-plugin/plugin.json` 将 harness 指向 `./skills/` 和正确的 `hooks-*.json`。（Claude Code 的 `.claude-plugin/plugin.json` 不设置这两个字段——它按约定自动发现 `skills/` 和 `hooks/hooks.json`。）

> **一个 hook *系统*不等于一个会话开始*事件*。** 一个 harness 可能拥有一个 `hooks.json` 机制——甚至其二进制中包含字面字符串 `SessionStart`——却没有能在会话开始时触发并能注入上下文的 hook 事件。（一个真实 harness 只暴露了 pre/post-tool 和 stop 事件；那些 `SessionStart` 字符串是遥测。）在确定使用 Shape A 之前，确认你需要的*具体事件*确实存在并能写入模型的上下文。如果不能，bootstrap 应放到一个指令文件中（Shape C）。

### Shape B — 进程内插件 / 扩展

该 harness 加载一个暴露生命周期回调的 JS/TS 模块。你通过 harness 的 API 注册 skills 目录，并通过在代码中修改消息数组来注入 bootstrap。

- 参考：`.opencode/plugins/superpowers.js`（JavaScript）和 `.pi/extensions/superpowers.ts`（TypeScript）。对于任何**没有原生 skill 工具**的 harness，pi 是最接近的参考。

### Shape C — 指令文件

该 harness 既没有 shell hook 也没有代码插件——它的会话开始入口是一个上下文文件，*由你已安装的扩展发布并由 manifest 声明*（例如 Gemini 的 `contextFileName` → 扩展自己的 `GEMINI.md`）。你无法运行代码或修改消息；扩展的上下文文件指向 bootstrap。没有注入器去组装字符串或剥离 frontmatter——harness 原样加载被引用的内容。**这之所以能工作，是因为该文件是已安装扩展的一部分**——绝不要用"编辑用户全局的 `GEMINI.md`/`AGENTS.md`"来替代发布你自己的文件（规则 2）。

- 参考：`gemini-extension.json`（manifest，带 `contextFileName`）、`GEMINI.md`（两个 `@`-include——bootstrap skill 和工具映射参考）、`skills/using-superpowers/references/gemini-tools.md`。
- 注意：`@`-include 是 Gemini 的特性。如果你的 harness 加载指令文件却没有 include 语法，你必须把 bootstrap 内容内联到该文件中。
- **不要相信 `@`-include 真的会被展开——要证明它。** 一个派生自 Gemini 的 harness 可能接受 `@./path` 语法，却把它当作一个*模型可以选择去读的提示*（它发出一个文件读取工具调用），而非一个有保证的内联展开。这就是"bootstrap 在每个会话都可靠存在"与"模型也许会去读它"之间的差别。运行一个唯一标记测试：如果标记*无需*工具调用就出现在上下文中，那就**内联内容**，而不是 `@`-include 它。

### 路由表

| 如果该 harness… | 使用形态 | 从哪里复制 |
|---|---|---|
| 在会话开始时运行一个 shell 命令并读取其 stdout | A（shell-hook） | Codex（`hooks/session-start-codex` + `hooks/hooks-codex.json` + `.codex-plugin/`） |
| 是一个带会话/消息生命周期回调的 JS/TS 插件宿主 | B（进程内） | OpenCode（`.opencode/`）——或如果没有原生 skill 工具则用 pi（`.pi/`） |
| 发布一个扩展声明、且始终加载的上下文文件 | C（指令文件） | Gemini（`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`） |
| 有一个 plugin install 命令，以及一个安装器会保留的 manifest `contextFileName`（或等价物） | 通过插件安装器的 C | Antigravity（`.antigravity-plugin/` —— `agy plugin install` 发布一个生成的上下文文件；验证安装器保留它——第 6 部分） |

大多数真实 harness 干净地符合一行；最后一行是混合情形（规则 2 仍然成立——bootstrap 搭载安装机制，绝不是用户配置编辑）。

---

## 第 5 部分 — 移植流程

### 步骤 1 — 研究最接近的参考实现

打开第 4 部分为你的形态所命名的文件并从头到尾阅读。下面的模式只是摘要；代码才是规范。

### 步骤 2 — 创建 manifest / 入口点

创建该 harness 用来识别插件的任何东西。在精神上与现有的一致：

- **Shape A：** 一个 `*-plugin/plugin.json`（见 `.codex-plugin/plugin.json`），包含 `name`、`version`、`description`、author/license/keywords、`"skills": "./skills/"`，以及 `"hooks": "./hooks/hooks-<harness>.json"`。再加上 `hooks-<harness>.json` 本身，它注册一个会话开始 hook，其命令调用 `run-hook.cmd`。
- **Shape B：** harness 加载的模块（例如 `.<harness>/plugins/*.js`）加上使其能被发现的任何包元数据。被提交的包元数据是**仓库根 `package.json`**：`main` 指向 OpenCode 插件，`pi` 字段（`pi.extensions`、`pi.skills`）加上 `pi-package` 关键字声明 pi 扩展。每个 harness 的本地 manifest 和 lockfile 都被排除在 git 之外——`.opencode/.gitignore` 排除了 `node_modules`、`package.json` 和 lockfile。对你的 harness 的*本地*安装产物也这样做，以免污染仓库——但绝不要 gitignore 仓库根 `package.json`，它是被追踪的事实来源。
  - **构建/依赖检查。** 决定 harness 如何加载你的模块：它是直接运行源码（pi 的 `.ts` 被原样从 `package.json` 引用；OpenCode 发布纯 `.js`），还是需要一个转译/构建步骤？Superpowers 是零运行时依赖。pi 的 `import type { ExtensionAPI }` 之所以能工作，正是因为 harness 直接运行 `.ts`、在加载时提供该类型，而仓库从不在 CI 中对该文件做类型检查——这个 import 甚至没有被声明为依赖。如果你的 harness 确实做类型检查或打包插件，那就行不通了：一个未声明的类型导入会失败，而 PR 规则只为新 harness 豁免*运行时*依赖，不包括开发/类型包。如果你遇到这种情况，请向维护者确认做法，而不是悄悄添加一个依赖。把任何构建产物排除在 git 之外并记录该命令。
- **Shape C（指令文件）：** 一个小型 manifest（见 `gemini-extension.json`：`name`、`description`、`version`、`contextFileName`）加上上下文文件本身（`GEMINI.md` 只是两个 `@`-include：bootstrap skill 和工具映射参考）。Gemini manifest 没有 `skills` 字段——Gemini 自动发现打包在已安装扩展中的 `skills/` 目录。如果你的 harness 有原生 skill 工具却没有注册目录的 manifest 字段，你必须找到它的发现约定（阅读其扩展文档），然后通过经验验证：接线之后，让模型列出其可用 skills——如果打包的 skills 没出现，说明发现还没工作。

### 步骤 3 — 接通 bootstrap 注入

这是移植的核心。共同目标：在会话开始时，把 `using-superpowers` skill 内容（包裹在 `<EXTREMELY_IMPORTANT>` 标签中）加上 harness 的工具映射放到模型面前，并附上一条说明表示该 skill 已激活，以免模型再次尝试加载它。你*如何*做到这一点——以及你组装什么 vs 让 harness 原样加载什么——完全取决于你的形态。**不要**把一种形态的配方套用到另一种。

**Shape A — 一个脚本读取 `SKILL.md` 并打印该 harness 的 JSON。** 被派发的脚本（`hooks/session-start`）`cat` 整个 `SKILL.md`（包括 frontmatter——没关系；它原样输出），用 "You have superpowers… for all other skills use the Skill tool" 前言包裹它，转义，并打印该 harness 的 JSON 形态。Shape A 的工具映射**不**在此内联——它位于 `references/<harness>-tools.md`（步骤 4）。把 JSON 输出形态搞对。`hooks/session-start` 通过环境变量检测 harness 并打印*三种*形态之一：

- Cursor（设置了 `CURSOR_PLUGIN_ROOT`）：`{ "additional_context": "…" }`
- Claude Code（设置了 `CLAUDE_PLUGIN_ROOT`，未设置 `COPILOT_CLI`）：`{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "…" } }`
- Copilot CLI / SDK 标准（其他）：`{ "additionalContext": "…" }`

这是一个陷阱。发出错误的字段，或多发一个，意味着 bootstrap 要么从不注入，要么注入两次（Claude Code 同时读取 `additional_context` 和 `hookSpecificOutput` 且不去重，所以两者都发会双重注入）。找出你的 harness 期望的精确字段、嵌套和事件匹配值。然后决定：给 `hooks/session-start` 加第四个分支，或者——如果 harness 需要不同的 bootstrap 消息或 env 契约——加一个专用的 `hooks/session-start-<harness>` 脚本，像 Codex 那样。如果你加了一个分支，而你的 harness *也*设置了某个更早分支所依据的环境变量（某些 harness 也设置 `CLAUDE_PLUGIN_ROOT`），请把你的分支排在原本会遮蔽它的那个分支之前。匹配 harness 自身的事件匹配字符串（Claude Code 用 `startup|clear|compact`，Codex 用 `startup|resume|clear`，Cursor 用 `sessionStart`）；错误的匹配意味着 hook 静默地永不触发。

**hook 配置 schema 本身因 harness 而异**——不要假设 Claude/Codex 的形态是通用的。对比 `hooks/hooks.json`、`hooks/hooks-codex.json` 和 `hooks/hooks-cursor.json`：Cursor 的使用 `"version": 1`、小写的 `sessionStart` 键、相对的 `./hooks/run-hook.cmd` 命令，并省略了其他文件使用的 `matcher`/`type`/`async` 字段。把你的 `hooks-<harness>.json` 匹配到最接近的现有文件，而不是某个单一的规范模板。

hook **命令字符串引用一个 harness 提供的 plugin-root 变量**，其名称因 harness 而异：`hooks.json` 用 `${CLAUDE_PLUGIN_ROOT}`，`hooks-codex.json` 用 `${PLUGIN_ROOT}`，Cursor 用相对路径。使用你的 harness 导出的那个。（`session-start` 脚本本身通过 `dirname` 重新推导根目录，所以脚本内容不依赖这一点——但 manifest 中的命令依赖。）

**发现 harness 的契约。** 上述三件事——环境变量、JSON 字段/嵌套、匹配字符串——是 harness 的契约，不是 Superpowers 的，所以你必须自行获取。阅读 harness 的 hook 文档，或者用经验方式找出：注册一个一次性的会话开始 hook，转储其环境并发出一个标记，然后观察哪个环境变量标识该 harness，以及 harness 如何/是否摄取你的 stdout。在编写真正的分支之前先把这些敲定。

**Shape B — 在代码中组装字符串，然后作为 user 消息注入。** 这里你自己构建 bootstrap：读取 `SKILL.md`，剥离其 YAML frontmatter，然后组装 `<EXTREMELY_IMPORTANT>` + 一段简短前言（说明该 skill 已加载、绝不可再次调用）+ 剥离后的内容 + 内联工具映射 + `</EXTREMELY_IMPORTANT>`。一个参考之间存在分歧的细节：OpenCode 的前言说 "do NOT use the skill tool…"（假设存在一个 `skill` 工具），而 pi 的只是说 "do not try to load using-superpowers again." 如果你的 harness 没有 skill 工具，请用 pi 的措辞，而不是 OpenCode 的。

把结果作为一条 **user 角色消息注入，而非 system 消息**——system 消息在每轮重复时会膨胀 token（#750），且多条 system 消息会破坏某些模型（#894）。你必须复刻三件事：

- **去重保护。** 生命周期回调可能反复触发（OpenCode 的 transform 在*每个* agent 步骤运行；pi 的 `context` 每轮触发）。在注入之前，检查是否已存在一个 bootstrap 标记，若存在则跳过。（各参考选用不同的标记——pi 用一个自定义字符串，OpenCode 用 `EXTREMELY_IMPORTANT` 标签；匹配标签更稳健，因为它不需要 harness 专用的常量。）把 bootstrap 内容缓存在模块级，以免每次调用都重新读取并重新解析 `SKILL.md`（#1202）。
- **压缩（Compaction）。** 如果 harness 会压缩/摘要历史，则在之后重新注入。pi 在 `session_start` 和 `session_compact` 上设置 `injectBootstrap` 标志，在 `agent_end` 上清除它，并把消息插入到任何开头的压缩摘要消息*之后*。OpenCode 依赖其每步重新注入加上去重保护。
- **消息对象形态因 harness 而异——发现你自己的，不要照搬字面。** 两个参考使用*不兼容*的形态：pi 构建 `{ role, content: [{ type, text }], timestamp }`；OpenCode 操作 `message.info.role` 和 `message.parts[]`。从它的 API 找到你 harness 的消息形态；逐字复制参考的对象字面量会静默失败。

**Shape C — 把你扩展的上下文文件指向 bootstrap；什么都不组装。** 这里没有注入器，所以你*不*剥离 frontmatter 或构建包裹字符串。你的扩展发布的上下文文件（由 manifest 声明——*不是*用户自己的全局文件）引入两样东西：`using-superpowers` skill 和该 harness 的工具映射参考。`GEMINI.md` 用两个 `@`-include 完成（`@./skills/using-superpowers/SKILL.md` 和 `@./skills/using-superpowers/references/<harness>-tools.md`）；harness 原样加载它们（包括 frontmatter），而 `SKILL.md` 内部已经自带了它自己的 `<EXTREMELY-IMPORTANT>` 块。如果你的 harness 没有 include 语法，就把内容内联到指令文件中。Gemini 不发布**任何**"已加载，请勿重新调用"的前言——对一个 `@`-include harness 而言，内容本身就是活动指令集，而非一个模型会重新加载的 skill。如果你发现你的 harness 确实会尝试重新调用，那就把那条说明作为字面行加到指令文件中（你没有其他方式添加它）。

### 步骤 4 — 编写工具映射

把动作词汇翻译成该 harness 的真实工具。覆盖以下每一个动作（仅省略确实不适用的）：

- 读一个文件
- 创建 / 编辑 / 删除一个文件（一个 `apply_patch` 风格的工具，还是分开的写/编辑？）
- 运行一个 shell 命令
- 搜索文件内容 / 按名称查找文件（grep、glob）
- 获取一个 URL / 网络搜索
- **派发一个 subagent**，包括如何传递 agent 类型——以及启用它所需的任何配置开关
- **创建 / 更新 todos**（把旧的 `TodoWrite` 引用当作这个动作）
- **调用一个 skill** —— 见步骤 5

**从 harness 获取真实工具名；绝不捏造。** 如果文档没列出，权威来源是 harness 本身：在一个活动会话中，让模型"列出你能调用的每个工具的精确机器名，每行一个"，并使用它报告的内容。

**harness 如何找到 `skills/` 目录本身也因 harness 而异**——要确认，不要假设。可能性：一个 manifest 的 `skills` 路径字段（Codex 的 `"skills": "./skills/"`）；一个 harness 自动扫描的*同目录* `skills/`（路径字段被**忽略**——一个真实 harness 只扫描与 `plugin.json` 同目录的 `skills/`）；一个 API/注册调用（OpenCode、pi）；或者你准备一个安装目录，把 manifest 与一个**指向仓库 `skills/` 的符号链接**配对，并让安装器指向该暂存目录（验证安装器*解引用*该符号链接并复制真实文件——在依赖它之前用 `agy plugin validate`/`install` 或等价物确认）。`skills` 路径字段*不可移植*。

映射放在哪里取决于形态：

- **Shape A：** 放在 `skills/using-superpowers/references/<harness>-tools.md`。agent 通过 bootstrap 到达它——`SKILL.md` 的"平台适配"小节链接了每个 harness 的参考文件。（Shape A 的 harness 没有指令文件；映射*不*内联到 hook 输出中。）
- **Shape B：** 映射通常内联到你注入的 bootstrap 字符串中（见 `superpowers.js` 中的 `toolMapping` 常量）。pi 把它放在*两个*地方——内联的 `piToolMapping()` **以及** `references/pi-tools.md`。如果你在两处维护，请都更新，否则移植只完成了一半。
- **Shape C：** 放在 `references/<harness>-tools.md`，并把它引入常驻的指令文件（例如 `GEMINI.md` `@`-include `gemini-tools.md`）。

你也可以在 `SKILL.md` 的"平台适配"小节加一行指针，指向你的 harness，这样读取 bootstrap 的 agent 就知道它的映射在哪里。这是一次移植可以对 `SKILL.md` 做的唯一编辑——而且仅因为那一节是指针列表，而非行为塑造内容。它不违反"不要编辑 skill 内容"的规则（第 1 部分）；不要触碰任何 skill 中的其他任何内容。（该列表是便利指针，而非详尽注册表——并非每个 harness 都被列出。）

### 步骤 5 — 处理没有原生 skill 工具的 harness

`using-superpowers/SKILL.md` 告诉模型*绝不使用文件工具手动读取 skill 文件——始终使用你所在平台的 skill 加载机制。*重点是"不要绕过机制"，而非"绝不使用文件读取"。什么算作"你所在平台的机制"取决于 harness——而对一个没有 skill 工具的 harness，文档记载的机制*就是*读取 `SKILL.md`。所以在那里读取它是遵循规则，而非破坏它。区分三种情况：

1. **原生 `Skill` 风格工具**（Claude Code、Copilot CLI、Gemini 的 `activate_skill`）：把映射指向该工具。
2. **原生 skill *发现*但没有 `Skill` 工具**（pi、Antigravity）：harness 能找到并列出 skills，但模型无法调用一个工具来加载某个 skill。把 skills 安装到 harness 扫描的位置（pi 通过 `resources_discover` → `skillPaths` 注册；OpenCode 通过其 `config` hook；`agy plugin install` 把它们复制进去），并告诉模型在某个 skill 适用时**用文件读取工具读取其 `SKILL.md`** 来加载它——这是此处受认可的机制，正如 `references/pi-tools.md` 所述。

   **对于 bootstrap 本身，优先使用一个声明的上下文文件（第 6 部分）。** 如果 harness 有一个 `contextFileName` 风格的 manifest 字段——正如 Antigravity 那样——通过安装器发布一个生成的上下文文件：它保证被加载，并携带 `using-superpowers` 内容和工具映射。那是强而有力的首选路径。

   **后备方案 —— 展示出来的 skill 索引。** 如果没有上下文文件字段，但 harness 在会话开始时展示每个已安装 skill 的 name + description，你*既不需要*构建的索引*也不需要*运行时列出指令——harness 就是索引，而 `using-superpowers` 自身被展示出来的描述就能成为触发模型加载它的东西。这比一个声明的上下文文件更软；相比上下文文件 / hook / 进程内注入器，它有两样**不**给你的东西——请为两者做好准备：
   - **它 bootstrap 的是*触发*，而非*工具映射*。** 一个注入器会在每个会话把 `<harness>-tools.md` 与 `using-superpowers` 一起前置。这里没有任何东西注入映射——模型只看到 skill *描述*，必须在需要工具名时*读取*你的 `references/<harness>-tools.md`。它能工作是因为 skills 命名动作（模型在行动时读取映射），但它比注入更软。确保映射能从模型加载的内容中到达——例如从 `SKILL.md` 的平台适配小节链接，并与 skills 一起安装——而不是只躺在仓库里。
   - **没有结构性保证触发会发生。** 没有 `<EXTREMELY_IMPORTANT>` 包裹，没有去重，没有压缩后的重新注入——是否触发取决于模型是否选择根据它在索引中看到的描述行动。这正是为什么此处验收测试是强制性的：它是*唯一*保证，所以要在你的用户实际会使用的模型上运行，而不仅是最强的那个。
3. **完全没有 skill 系统：** 没有什么需要注册，*唯一*机制是模型按需读取 `SKILL.md`。但模型无法读取它找不到的东西：`using-superpowers/SKILL.md` **不**枚举可用 skills，所以单靠它自己模型不会知道哪些 skills 存在或它们的触发条件。你必须提供一条发现路径。两个选项，且在耐久性上不同：(a) 生成一个 skill 索引（每个 `skills/*/SKILL.md` 的 `name` + `description` frontmatter），放在 `<EXTREMELY_IMPORTANT>` 包裹内、工具映射旁边（上面的 Shape B 配方），这样它就被去重保护覆盖——但构建时索引会随着 skills 增加而过时；或 (b) 指示模型在运行时列出 `skills/*/SKILL.md` 并读取它们的 frontmatter 以找到匹配——较慢但永不过时。除非你有理由不这样做，否则优先选 (b)。两者都没有的话，一个无 skill 系统的移植会加载 bootstrap，却静默地从不触发任何其他 skill。

在情况 2 和 3 中，在你的工具映射中明白地说读取 `SKILL.md` 是受认可路径，这样模型就不会以为自己在违反"绝不读取 skill 文件"的规则。不要去一个没有 skill 系统的 harness 里寻找 `skillPaths` 风格的注册 API——情况 3 没有这种东西。

### 步骤 6 — 添加测试

匹配现有的每个 harness 的测试风格：

- **Shape A：** 断言 hook 的 stdout 具有你的 harness 摄取的确切 JSON 形态，且包含 bootstrap。见 `tests/hooks/test-session-start.sh`，它校验每个 harness 的输出形态。
- **Shape B：** 一个单元测试，伪造 harness 的插件 API，并断言生命周期处理器被注册、bootstrap 只注入一次、去重保护工作，以及（如果相关）压缩后重新注入工作。见 `tests/pi/test-pi-extension.mjs`。以 `tests/opencode/` 的风格添加一个隔离安装的集成检查。
- 如果 bootstrap 被缓存，测试当文件缺失时缓存的行为（见 OpenCode 缓存测试）。

这些自动化测试覆盖接线；步骤 7 中的实时 tmux 运行才是证明集成真正触发 skills 的东西。

### 步骤 7 — 本地安装，然后驱动一个实时实例来验证

你无法通过阅读代码确认一个移植能工作。你必须用你进行中的移植加载 harness 并观察一个真实会话——这也是你产出 PR 所需会话记录的方式。

**本地安装。** 把 harness 的一个*本地*实例指向你的工作树，而非已发布的构建：

- **Shape A / C：** 从本仓库的本地路径安装插件/扩展（或把它的目录符号链接到 harness 查找的位置）。在其文档中找到该 harness 的"从本地目录 / git checkout 安装"路径。
- **Shape B：** 注册本地模块——例如一个 `opencode.json` 的 `plugin` 条目指向本地路径，或 pi 从仓库解析 `package.json` 字段。

每次变更后重新安装并重启 harness，因为 bootstrap 在启动时加载。

**用 tmux 驱动它。** 大多数 harness 是无法通过管道 stdin 驱动的交互式 REPL/TUI，所以在分离的 tmux 会话内运行 harness 并用 `send-keys` / `capture-pane` 控制。一个 harness 可能宣传一种非交互的"运行单个 prompt"模式（例如 `opencode run "..."`）——用它做快速冒烟检查，但**不要依赖它**：这些模式常常不稳定、受认证门控或受信任门控（一个真实 harness 的 `--print` 模式每次都挂起并超时、无任何输出）。准备好通过 tmux 做*一切*，包括冒烟检查。

**先清空门控，否则 tmux 会静默停滞。** 许多 harness 在首次运行的引导、一个"是否信任此目录？"提示、一个 sandbox 模式或一个权限门上阻塞——而一个分离的 tmux 会话会在等待时静悄悄地停在那里、不报任何错误。在运行之前，预先信任你的临时目录（在 harness 的设置/配置中）或准备好通过 `send-keys` 回答那些提示，并在你的第一次 `sleep` 中考虑 harness 的启动时间。

```bash
# 1. 以分离方式启动 harness，放在一个一次性项目目录中
mkdir -p /tmp/port-smoke
tmux new-session -d -s port-test -c /tmp/port-smoke '<harness-launch-command>'

# 2. 让它初始化 —— 真实的 TUI 比你想的更慢（带模型握手 10 秒以上）；
#    调整这个。然后在键入 prompt 之前捕获并清除任何阻塞模态框：首次运行引导和
#    "信任此目录？"是模态的，所以在它们期间发送的按键会选择菜单项而非键入你的 prompt。
sleep 12
tmux capture-pane -t port-test -p          # 引导 / 信任提示？先通过 send-keys 回答它
# （例如 tmux send-keys -t port-test Enter   # 接受信任提示 —— 先检查再假设）

# 3. 冒烟检查：模型知道自己有 superpowers 吗？
#    把文本和 Enter 作为分开的 send-keys、中间留一拍发送 ——
#    一起发送在某些 TUI 上会竞态（Enter 在文本落地之前到达）。
tmux send-keys -t port-test 'What are your superpowers?'; sleep 0.4; tmux send-keys -t port-test Enter
sleep 5
tmux capture-pane -t port-test -p          # 回复应显示它知道自己的 skills

# 4. 验收测试：精确 prompt（注意转义的撇号），全新会话
tmux send-keys -t port-test 'Let'\''s make a react todo list'; sleep 0.4; tmux send-keys -t port-test Enter
# 轮询直到这一轮结束 —— 每隔几秒重新捕获，不要只捕获一次
sleep 8
tmux capture-pane -t port-test -p          # 通过 = brainstorming 在任何代码之前触发

# 5. 为 PR 保存会话记录，然后清理
tmux capture-pane -t port-test -p > /tmp/port-smoke/transcript.txt
tmux kill-session -t port-test
```

此处会咬人的 tmux 陷阱：启动后等待片刻再做第一次捕获；把 prompt 文本和 `Enter` 作为*分开的* `send-keys` 调用、中间留一个短暂 `sleep`（一起发送在某些 TUI 上会竞态）；`Enter` 是键名而非 `\n`；agent 的这一轮需要时间，所以**循环轮询 `capture-pane`** 而非只捕获一次；`capture-pane` 只显示可见窗格，所以对长对话使用 harness 自身的会话记录/日志文件作为事实来源；完成时务必 `kill-session`。

如果冒烟检查显示模型*不*知道自己有 superpowers，说明 bootstrap 没有加载——在做验收测试之前先修复它。

---

## 第 6 部分 — 分发与发布

本仓库中一个可工作的集成，在真实用户能安装之前并不算可用。分发因 harness 生态而异——找到你的：

| 渠道 | 示例 | 你做什么 |
|---|---|---|
| 原生插件市场 | Claude Code | 在 `.claude-plugin/marketplace.json` 中注册；用户执行 `/plugin install`。外部的 `superpowers-marketplace` 仓库是用户安装的事实来源——见 `CLAUDE.md` 中的发布步骤。 |
| 外部市场 fork，由脚本同步 | Codex | `scripts/sync-to-codex-plugin.sh` 把被追踪的插件文件 rsync 到一个单独的 fork 仓库并开一个 PR。阅读它的 include/exclude 列表，这样你发布的目录树才正确（它有意丢弃仓库内部目录和其他 harness 的 dotdir）。 |
| Git-URL 扩展安装 | Gemini、Kimi Code、OpenCode | 用户从一个 git URL 安装（`gemini extensions install …`；Kimi Code `/plugins install …`；一个 `opencode.json` 的 `plugin` 数组条目）。记录确切的命令。 |
| 包 manifest 字段 | pi | 通过仓库根 `package.json` 的字段声明；用户通过 harness 的包命令安装。 |
| 本地安装器（plugin install） | Antigravity（`agy`） | 一个小型 `install.sh`，针对一个持有 manifest、skills 和生成的 `contextFileName` 上下文文件（即 bootstrap）的暂存目录运行 harness 自身的 `agy plugin install`。一切都通过安装机制到达——*而非*通过编辑用户的配置（见下文）。 |

然后：

- **一个插件安装器可能静默剥离*未声明的*文件——所以要让 bootstrap 成为安装器*认识*的文件，绝不能是用户配置编辑。** 一个 `plugin install` 通常只复制它认识的组件（skills/agents/commands/mcp/hooks/context）并丢弃其他一切，所以一个 manifest 没声明的上下文文件会在安装时消失。修复方法**不是**放弃并写入用户配置（**规则 2**）——而是把 bootstrap 声明为一个被认识的组件。按升级顺序：
  - **发布一个 manifest 声明的上下文文件。** 如果 harness 有一个 `contextFileName` 风格字段（一个它每个会话加载的、由扩展声明的文件），那是最强有力的干净 bootstrap：声明它，安装器既保留它*又*被 harness 加载它。在安装时从活动的 `using-superpowers/SKILL.md` + 工具映射（包裹在 `<EXTREMELY_IMPORTANT>` 中）生成它，这样已安装的 bootstrap 永不漂移。这正是 `.antigravity-plugin/install.sh` 所做的——`agy plugin install` 报告 `✔ context : ANTIGRAVITY.md`，而一个干净会话读取 `using-superpowers` 的 SKILL.md、加载 `brainstorming`、在任何代码之前进入 brainstorming 流程。**用标记验证**安装器保留该文件且 harness 加载它：一个移植者错误地断定它做不到，因为他们在发布文件时*没有*声明 `contextFileName`，于是它作为未识别内容被剥离。
  - **否则依赖已安装的 `using-superpowers` skill 本身。** 如果 harness 在会话开始时展示每个已安装 skill 的 name + description，那么 `using-superpowers` 的描述（"Use when starting any conversation…"）就能提示模型加载它——安装该 skill *就是* bootstrap。更软（没有有保证的包裹；它携带的是触发而非工具映射——见步骤 5），所以当可用时优先使用声明的上下文文件。
  - 如果两者都不行，该 harness 还不能被干净地支持——**直说**并提出来，而不是手工编辑用户的配置。
- **编写安装文档。** 一个 `docs/README.<harness>.md` 和/或一个 `.<harness>/INSTALL.md`（见 `docs/README.opencode.md` 和 `.opencode/INSTALL.md`），加上顶层 `README.md` 的一个安装小节。唯一受支持的安装动作是**运行 harness 自身的安装命令**（`agy plugin install`、`gemini extensions install`、`/plugin install` 等）。手工复制 skill 文件和编辑用户的全局/个人配置*都*是禁区（规则 2 / PR 规则）。如果 harness 根本没有安装命令——它的唯一入口是一个用户拥有的配置文件——那么它违反了"通过安装机制交付"的规则，你应该提出来，而不是发布一个编辑用户文件的安装器。
- **注册版本。** 如果你的 harness 引入一个*新的*带版本 manifest，把它的路径和版本字段添加到 `.version-bump.json`，这样 `scripts/bump-version.sh` 能保持它同步（阅读该文件看当前追踪了什么）。一个未在那里注册的新 manifest 会发布过时版本。如果你的 harness 搭载一个已被追踪的文件——pi 在仓库根 `package.json` 中声明自己，而它已被列出——则没有什么新东西要添加。
- **如果没有现成渠道合适，你就是在搭建一个新的。** 四行可能都不匹配你的 harness。如果它需要 Codex 风格的外部 fork 同步，`scripts/sync-to-codex-plugin.sh` 是要克隆的模板（注意它锚定的 include/exclude 列表和它的 PR 自动化）。而且每当你添加一个新的每个 harness 的目录时，把它加入*其他* harness 的同步排除项（例如 `sync-to-codex-plugin.sh` 中的 EXCLUDES 列表），这样你的 dotdir 不会泄漏到它们的分发中。

---

## 第 7 部分 — 跨平台 / Windows

仅与 shell-hook 形态相关。`hooks/run-hook.cmd` 是一个 polyglot：一个既作为 Windows 批处理脚本有效、又作为 Unix shell 脚本有效的单一文件。在 Windows 上，`cmd.exe` 运行批处理部分，它定位 `bash`（先 Git for Windows，再 `PATH` 上的 `bash`）并运行被命名的 hook 脚本；如果没找到 bash，它干净退出，这样 harness 仍能工作，只是没有注入。在 Unix 上，开头的 `:` 使批处理块成为无操作，shell 直接运行脚本。

它强制执行的两条规则，你必须遵守：

- **Hook 脚本是无扩展名的**（`session-start`，而非 `session-start.sh`）。Claude Code 在 Windows 上的处理会对任何包含 `.sh` 的命令前置 `bash`，这会双重调用。给你的 hook 脚本命名时不带扩展名。
- 不要为 hook 脚本写按操作系统区分的变体。一个无扩展名的 bash 脚本加上 polyglot 包装器就覆盖了全部三个平台。

`hooks/run-hook.cmd` 本身是权威实现——阅读它。dispatcher 模式背后的背景和理由见 `docs/windows/polyglot-hooks.md`。

---

## 第 8 部分 — 提交 PR

- 目标分支为 **`dev`**。一个 PR 一个 harness。
- 填写 PR 模板的 **"New harness support"** 小节，并粘贴完整的验收测试会话记录（展示 `brainstorming` 自动触发的那个 "Let's make a react todo list" 会话）。没有此证明的 PR 会被关闭。
- Superpowers 是零依赖插件。不要添加第三方运行时依赖。添加一个新 harness 是贡献者规则允许的唯一豁免，即便如此，也只限于集成严格需要的——纯类型导入会被编译掉，没问题；运行时包不行。
- 不要触碰 skill 内容（第 1 部分）。如果你发现自己为了移植工作而编辑 `SKILL.md`，修复方案应放在你的工具映射里。

---

## 附录 A — 参考集成（当前）

把它当作活的索引用；有疑问时读文件，不要读这张表。

| Harness | 入口点 | Bootstrap 机制 | 工具映射 | 测试 | 分发 |
|---|---|---|---|---|---|
| Claude Code | `.claude-plugin/plugin.json` + `hooks/hooks.json` | shell hook → `hooks/session-start`（`hookSpecificOutput.additionalContext`） | 原生 `Skill` 工具；`references/claude-code-tools.md` | `tests/hooks/` | marketplace |
| Codex | `.codex-plugin/plugin.json` + `hooks/hooks-codex.json` | shell hook → `hooks/session-start-codex` | `references/codex-tools.md` | `tests/codex-plugin-sync/`、`tests/hooks/` | fork 同步（`scripts/sync-to-codex-plugin.sh`） |
| Cursor | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json` | shell hook → `hooks/session-start`（`additional_context`） | `references/claude-code-tools.md` | `tests/hooks/` | 手工编写 |
| Copilot CLI | （共享 Claude Code hook 路径；`COPILOT_CLI` env） | shell hook → `hooks/session-start`（`additionalContext`） | `references/copilot-tools.md` | `tests/hooks/` | — |
| Gemini CLI | `gemini-extension.json` + `GEMINI.md` | 指令文件 `@`-include bootstrap + 映射 | `references/gemini-tools.md` | — | `gemini extensions install` |
| Kimi Code | `.kimi-plugin/plugin.json` | manifest 的 `sessionStart.skill` 加载 `using-superpowers` | manifest 内联的 `skillInstructions` | `tests/kimi/` | marketplace 或 `/plugins install` GitHub URL |
| OpenCode | `.opencode/plugins/superpowers.js`（通过根 `package.json` 的 `main` 声明） | 进程内：`config` hook 注册 skills 目录；`experimental.chat.messages.transform` 注入 user 消息 | 在 `superpowers.js` 内联 | `tests/opencode/` | `opencode.json` plugin git URL |
| pi | `.pi/extensions/superpowers.ts` | 进程内：`resources_discover` 注册 skills；`context` 事件注入 user 消息；生命周期标志 + 感知压缩 | 内联的 `piToolMapping()` **以及** `references/pi-tools.md` | `tests/pi/` | 仓库根 `package.json` 字段 |

## 附录 B — 咬过移植者的陷阱

- **逐个会话地启用不算移植。** 如果你的人类伙伴每个会话都必须做点什么才能用上 Superpowers，验收测试会失败。重读第 2 部分。
- **错误的 JSON 字段 → 静默失败或双重注入。** 仅 Shape A。确认确切字段/嵌套；Claude Code 不去重地读取两个字段。
- **Hook 配置 schema 因 harness 而异。** Shape A。Cursor 的 `hooks-cursor.json` 看起来一点也不像 Claude/Codex 的那个（`version`、小写 `sessionStart`、相对命令、没有 `matcher`/`type`/`async`）。匹配最接近的现有文件。
- **Plugin-root 环境变量因 harness 而异。** Shape A。hook 命令用 `${CLAUDE_PLUGIN_ROOT}`（Claude）、`${PLUGIN_ROOT}`（Codex）或相对路径（Cursor）。用你的 harness 导出的；脚本自己重新推导根目录。
- **System 消息注入。** Shape B 故意注入一条 *user* 消息（#750、#894）。不要把它"修复"成 system 消息。
- **按步骤 vs 按轮回调。** OpenCode 每步触发（按调用去重保护）；pi 每轮触发（生命周期标志 + `agent_end` 重置）。把一个 harness 的去重策略照搬到另一个的回调频率上会破坏注入。
- **消息对象形态因 harness 而异。** Shape B。pi 和 OpenCode 用不兼容的形态；发现你自己的，不要复制参考的对象字面量。
- **寻找一个并不存在的 skill 注册 API。** 一个没有 skill 系统（不只是没有 `Skill` 工具）的 harness 没什么可注册的——模型按需读取 `SKILL.md`。不要假设存在一个 `skillPaths` 等价物。
- **映射放在两处。** 对进程内插件，映射可能既内联又在 `references/` 文件中（pi）。两者都更新。
- **"绝不读取 skill 文件"那句话。** 它的意思是"不要绕过你所在平台的 skill 加载机制"，而非"绝不使用文件读取"。在一个没有 skill 工具的 harness 上，那个机制*就是*读取 `SKILL.md`——在映射中明白地说出来（第 5 部分）。
- **Windows 上的 `.sh`。** 保持 hook 脚本无扩展名（第 7 部分）。
- **未注册的版本。** 一个未添加到 `.version-bump.json` 的新 manifest 会发布过时版本（第 6 部分）。
- **为了让适配 harness 而编辑 skills。** 绝不要。修复方案应放在工具映射中。
