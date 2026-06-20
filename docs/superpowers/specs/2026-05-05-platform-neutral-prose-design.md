# 平台中立的叙述文本 — Phase A 设计

## 背景

Superpowers 会发布到多个 agent 运行时（Claude Code、Codex、Cursor、OpenCode、Copilot CLI、Gemini CLI）。skill 内容和配套文档最初是为 Claude Code 编写的，在若干位置使用了 "Claude" 一词，而这些地方其实适用于任意运行时的 agent。OpenAI 的 vendored fork（openai/plugins#217）曾尝试过一次整体改写，但其中存在明显错误——重写了历史归因路径、模型名称和平台特定的安装说明——我们希望在去除纯属偶然的平台中心化叙述的同时，避免重蹈覆辙。

整个工作按引用类别拆分为多个阶段。**本 spec 仅涵盖 Phase A：** 在非平台特定语境中以泛化第三人称叙述提到 "Claude" 的情形。后续阶段（配置文件引用、营销文案、工具名引用）不在本次范围内，会有各自的 spec。

## 范围内

泛化叙述文本中提到 "Claude" 的位置，涉及：

- `skills/*/SKILL.md` 以及活动 skill 目录下的配套 `.md` 文件
- `skills/writing-skills/anthropic-best-practices.md`
- `README.md`（仅在提法属于泛化叙述、而非平台营销时）

外加一个术语重命名：**Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**，位于 `skills/writing-skills/SKILL.md`。

## 范围外

- **平台/运行时相关陈述** — "In Claude Code:"、安装说明、工具映射引用。（Phase D 候选。）
- **配置文件引用** — CLAUDE.md、AGENTS.md、GEMINI.md 的优先级列表，以及"项目约定放在哪里"的提示。（Phase B。）
- **工具名引用** — `Skill`、`Bash`、`Read`、`Task`、`TodoWrite`。skills 是以 Claude Code 的工具词汇表书写的；既有的 `references/{codex,copilot,gemini}-tools.md` 文件对它们做了映射。（编写本 spec 时，计划是推迟或跳过这些。Phase E 最终完成了这一项——在所有活动 skills 中用动作语言替换工具名，并围绕同一词汇表统一平台工具引用。）
- **README 中的营销文案** — "Superpowers for Claude Code"、以平台命名的安装章节。（Phase C。）
- **历史产物** — `docs/plans/*.md`、`docs/superpowers/specs/*.md`、`CREATION-LOG.md`。这些是带日期的、时间点性文档；改写它们等于改写历史。
- **模型标识符** — Claude Haiku / Sonnet / Opus。这些是真实的产品名。
- **文件名 / URL 引用** — `CLAUDE.md`、`claude.com`、`claude-plugin/`、`~/.claude/` 下的路径。
- **`anthropic-best-practices.md` 文件名** — 即便我们重写其中的叙述文本，文件名仍按其来源命名。

## 替换风格

使用在英文中读起来自然的混合形式：

- **第二人称——"your agent"**，用于就 skill 作者*自己的*运行时对其进行称谓
  - "your agent reads the description"
- **第三人称——"the agent" / "agents" / "an agent"**，用于泛化地描述系统行为
  - "Future agents find your skills"
  - "Use words an agent would search for"
  - "Agents read SKILL.md only when the skill becomes relevant"

选择最贴合上下文句子的形式；不要为了强行统一而牺牲表达流畅。在自然的情况下使用复数（"future agents"、"agents read"），而不是一律说 "the agent"。

### 保留为 "Claude" 的例外

- 模型名：Claude Haiku、Claude Sonnet、Claude Opus
- 文件名和 URL：`CLAUDE.md`、`claude.com`、`~/.claude/`
- 品牌平台名 "Claude Code"——当其作为运行时本身被指代时（在后续阶段处理）

### 术语重命名

- **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**
  - 出现在 `skills/writing-skills/SKILL.md` 中，作为章节标题及附近叙述。需重命名标题、缩写，以及文件内的所有交叉引用。

## 受影响文件

以下计数基于一次过滤掉例外项后的 `grep`：

| 文件 | 泛化叙述提及数 |
|------|------------------------|
| `skills/writing-skills/SKILL.md` | ~12（含 CSO 标题 + 正文） |
| `skills/writing-skills/anthropic-best-practices.md` | ~30 |
| `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` | ~1——文件名保留（它是 CLAUDE.md 测试产物）；"Variant C: Claude.AI Emphatic Style" 标题也保留（它是为某种特定风格命名的标签） |
| `README.md` | ~1 |

最终清单在实现时通过重新运行过滤后的 grep 予以确认。

## 提交计划

四次原子提交，按顺序：

1. **在 `skills/writing-skills/SKILL.md` 中将 CSO 重命名为 SDO**。机械式、隔离、便于在我们改变对术语看法时回退。
2. **活动 skills 叙述文本** — 在 `skills/*/SKILL.md` 及配套 `.md` 中将泛化 "Claude" 替换为 "agent" 形式，排除 `anthropic-best-practices.md`。
3. **`anthropic-best-practices.md` 叙述文本** — 同样的替换规则。单独提交，因为该文件是一份外部文档的 vendored 改编；隔离改动能让未来与上游的对照更容易阅读。
4. **README.md 叙述文本** *（仅当过滤后仍有泛化叙述提及剩余时）*。若为空则跳过。

每个提交信息都要标明阶段（"Phase A"）以及切片（"rename CSO to SDO"、"agent prose in active skills" 等），使整个系列自解释。

## 验证

每次提交之后：

- `grep -rn "Claude" <touched-paths>` — 每一处剩余命中都必须落入已记录的例外范畴（模型名、文件名、URL、"Claude Code" 平台名、历史产物）。
- 端到端阅读改动后的文件——替换不应破坏句意连贯、代词一致性或列表平行结构。
- 无需运行测试；本次仅为叙述文本改动。

最后一次提交之后：

- 在真实会话中粗略浏览每个改动过的 skill，确认没有任何地方读起来生硬。

## 非目标

- 不要改变行为、结构、标题（CSO→SDO 除外）、示例、代码块或 YAML frontmatter。
- 不要引入新章节、提示框或兼容性说明。
- 不要在编辑时对替换之外的叙述文本进行"改进"。
