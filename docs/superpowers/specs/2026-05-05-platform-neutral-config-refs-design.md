# 平台中立的配置文件引用 — Phase B 设计

## 背景

Phase A（见 `2026-05-05-platform-neutral-prose-design.md`）已将泛化的第三人称"Claude"叙述替换为 agent 中立的表述形式。本阶段处理下一个类别：skills 内部对各平台指令文件（CLAUDE.md、AGENTS.md、GEMINI.md）的引用。

该 plugin 可运行在多个 harness 上，每个 harness 读取各自的指令文件。当某个 skill 把 CLAUDE.md 当作唯一文件来指代时，这是一种以 Claude Code 为中心的假设，在 Codex / Gemini CLI / OpenCode 上并不成立。

## 范围内

活动 skills 中的两处具体行：

1. **`skills/writing-skills/SKILL.md:58`** — `Project-specific conventions (put in CLAUDE.md)`
2. **`skills/receiving-code-review/SKILL.md:30`** — `"You're absolutely right!" (explicit CLAUDE.md violation)`

## 范围外

- **`skills/using-superpowers/SKILL.md:22, 26`** — 指令优先级列表。该列表已经将三者（CLAUDE.md、GEMINI.md、AGENTS.md）一并列出，这是正确的：这一节是在对多平台 plugin 上的*什么才算作用户指令*做出真实断言。无需改动。
- **历史 / 示例产物**：
  - `skills/systematic-debugging/CREATION-LOG.md` — 归因路径（`~/.claude/CLAUDE.md`）是一个历史事实。
  - `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` — 整个文件就是一个测试 CLAUDE.md 内容变体的实例演示。文件名、正文以及来自 `testing-skills-with-subagents.md` 的引用都保留；对其进行规整化反而会破坏这个示例的意义。
- **平台工具引用** — Phase D 的候选：
  - `skills/using-superpowers/SKILL.md:40`（关于 GEMINI.md 的 Gemini CLI 工具映射说明）
  - `skills/using-superpowers/references/gemini-tools.md`（`save_memory` 会持久化到 GEMINI.md）

## 替换规则

两处不同的修改，分别对应范围内的一行。

### 规则 1："项目特定约定放在哪里"

`writing-skills/SKILL.md:58`：

- **修改前：** `Project-specific conventions (put in CLAUDE.md)`
- **修改后：** `Project-specific conventions (put in your instructions file)`

使用一个泛化短语，而不是选定某个具体文件名。不同的 harness 读取不同的文件（CLAUDE.md、AGENTS.md、GEMINI.md 等），skill 不应假定某一个。platform-tools 参考文档（`references/{codex,copilot,gemini}-tools.md`）才是标明各平台首选文件的合适位置。

### 规则 2："(explicit CLAUDE.md violation)" 括注

`receiving-code-review/SKILL.md:30`：

- **修改前：** `"You're absolutely right!" (explicit CLAUDE.md violation)`
- **修改后：** `"You're absolutely right!" (explicit instruction-file violation)`

这个括注承担着实际作用——它标示出这句话不仅仅是风格不佳，而是主动违反了许多用户写进自己指令文件中的规则。"Instruction file" 是覆盖 AGENTS.md / CLAUDE.md / GEMINI.md 的天然跨平台术语，既能保留原始信号，又不必选定某个文件名，也不会软化为 "common"。

## 提交计划

原子提交，按顺序：

1. **`writing-skills/SKILL.md`** — 在"项目约定放在哪里"那一行，将 CLAUDE.md 改为 "your instructions file"
2. **`receiving-code-review/SKILL.md`** — 在违规括注中将 CLAUDE.md 改为 instruction-file
3. **平台工具参考文档** — 在每个 `references/{codex,copilot,gemini}-tools.md` 中添加该平台首选的指令文件名（CLAUDE.md、AGENTS.md、GEMINI.md 等），以便读者能将 "your instructions file" 解析到具体文件名。

每个提交信息都要标明 "Phase B" 以及对应的工作切片。

## 验证

每次提交之后：

- 阅读该行所在的上下文段落，确认语法和含义仍然通顺。
- `grep -n "CLAUDE\.md" <touched-file>` — 在活动叙述文本中不应再有命中（例外项已记录在案）。

两次提交都完成后：

- `grep -rn "CLAUDE\.md" skills/` 应当只返回已记录的例外项（CREATION-LOG、CLAUDE_MD_TESTING 及其入站引用、using-superpowers 中的优先级列表）。

## 非目标

- 不要改动 `using-superpowers/SKILL.md` 中的优先级列表顺序。重新排序 CLAUDE.md / GEMINI.md / AGENTS.md 是一种审美改动，不是替换，不属本次范围。
- 不要重命名 `examples/CLAUDE_MD_TESTING.md` 或修改其内容。
- 不要修改 Gemini CLI 特有的工具引用（Phase D 的候选）。

## 实现说明

本 spec 所述的 Phase B 涵盖三次提交以及三个非 Claude Code 平台的 platform-tools 引用文档。实现时还多做了一步：在提交 `8505703` 中新增了第四个引用 `references/claude-code-tools.md`，以保持对称性，让 Claude Code 的指令文件约定和工具名清单与其余平台并列存放，而非隐含地散落在周边 skill 的叙述文本中。这一新增在本 spec 中并未预见，但与其意图一致。
