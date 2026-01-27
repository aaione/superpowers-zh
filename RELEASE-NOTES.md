# Superpowers Release Notes

## v4.1.1 (2026-01-23)

### 修复 (Fixes)

**OpenCode: 按照官方文档标准化 `plugins/` 目录 (#343)**

OpenCode 的官方文档使用 `~/.config/opencode/plugins/` (复数)。我们的文档以前使用 `plugin/` (单数)。虽然 OpenCode 接受这两种形式，但我们已统一使用官方惯例以避免混淆。

变更:
- 在 repo 结构中重命名 `.opencode/plugin/` 为 `.opencode/plugins/`
- 更新所有平台上的所有安装文档 (INSTALL.md, README.opencode.md)
- 更新测试脚本以匹配

**OpenCode: 修复 symlink 说明 (#339, #342)**

- 在 `ln -s` 之前添加显式的 `rm` (修复重新安装时的 "file already exists" 错误)
- 添加了以前 INSTALL.md 中缺少的 skills symlink 步骤
- 从已弃用的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0 (2026-01-23)

### 重大变更 (Breaking Changes)

**OpenCode: 切换到原生 skills 系统**

Superpowers for OpenCode 现在使用 OpenCode 的原生 `skill` 工具，而不是自定义的 `use_skill`/`find_skills` 工具。这是一个更清晰的集成，可与 OpenCode 的内置 skill 发现一起使用。

**需要迁移:** Skills 必须被 symlinked 到 `~/.config/opencode/skills/superpowers/` (参见更新的安装文档)。

### 修复 (Fixes)

**OpenCode: 修复会话开始时的 agent 重置 (#226)**

以前使用 `session.prompt({ noReply: true })` 的 bootstrap 注入方法导致 OpenCode 在第一条消息时将选定的 agent 重置为 "build"。现在使用 `experimental.chat.system.transform` hook，它直接修改 system prompt 而没有副作用。

**OpenCode: 修复 Windows 安装 (#232)**

- 删除了对 `skills-core.js` 的依赖 (消除了当文件被复制而不是 symlinked 时损坏的相对导入)
- 为 cmd.exe, PowerShell, 和 Git Bash 添加了全面的 Windows 安装文档
- 记录了每个平台的正确 symlink vs junction 用法

**Claude Code: 修复 Claude Code 2.1.x 的 Windows hook 执行**

Claude Code 2.1.x 更改了 hooks 在 Windows 上的执行方式: 它现在自动检测命令中的 `.sh` 文件并预置 `bash `。这破坏了 polyglot wrapper 模式，因为 `bash "run-hook.cmd" session-start.sh` 试图将 .cmd 文件作为 bash 脚本执行。

修复: hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制 shell 脚本使用 LF 行尾 (修复 Windows checkout 上的 CRLF 问题)。

---

## v4.0.3 (2025-12-26)

### 改进 (Improvements)

**加强 using-superpowers skill 以进行显式 skill 请求**

解决了 Claude 即使在用户按名称显式请求时 (例如 "subagent-driven-development, please") 也会跳过调用 skill 的故障模式。Claude 会认为 "我知道那通过什么"，然后直接开始工作而不是加载 skill。

变更:
- 更新 "The Rule" 为 "调用相关或请求的 skills" 而不是 "检查 skills" - 强调主动调用而非被动检查
- 添加 "在任何响应或行动之前" - 原文只提到 "响应"，但 Claude 有时会在没有响应的情况下采取行动
- 添加了调用错误的 skill 也没关系的保证 - 减少犹豫
- 添加新的红旗: "I know what that means" → 知道概念 ≠ 使用 skill

**添加显式 skill 请求测试**

`tests/explicit-skill-requests/` 中的新测试套件验证 Claude 在用户按名称请求时正确调用 skills。包括单轮和多轮测试场景。

## v4.0.2 (2025-12-23)

### 修复 (Fixes)

**Slash commands 现在仅限用户使用**

向所有三个 slash commands (`/brainstorm`, `/execute-plan`, `/write-plan`) 添加了 `disable-model-invocation: true`。Claude 不再能够通过 Skill 工具调用这些命令——它们仅限于手动用户调用。

底层 skills (`superpowers:brainstorming`, `superpowers:executing-plans`, `superpowers:writing-plans`) 仍然可供 Claude 自主调用。此更改防止了当 Claude 调用仅重定向到 skill 的命令时产生的混淆。

## v4.0.1 (2025-12-23)

### 修复 (Fixes)

**阐明如何在 Claude Code 中访问 skills**

修复了一个令人困惑的模式，即 Claude 会通过 Skill 工具调用一个 skill，然后尝试单独读取 skill 文件。`using-superpowers` skill 现在明确指出 Skill 工具直接加载 skill 内容——不需要读取文件。

- 向 `using-superpowers` 添加 "How to Access Skills" 部分
- 说明中将 "read the skill" 更改为 "invoke the skill"
- 更新 slash commands 以使用完全限定的 skill 名称 (例如 `superpowers:brainstorming`)

**向 receiving-code-review 添加 GitHub 线程回复指南** (h/t @ralphbean)

添加了关于在原始线程中回复行内审查评论的说明，而不是作为顶层 PR 评论。

**向 writing-skills 添加自动化优于文档指南** (h/t @EthanJStark)

添加了指南，即机械约束应该自动化，而不是文档化——将 skills 留给判断调用。

## v4.0.0 (2025-12-17)

### 新功能 (New Features)

**subagent-driven-development 中的两阶段代码审查**

Subagent 工作流现在在每个任务后使用两个单独的审查阶段:

1.  **规范合规性审查** - 怀疑的审查者验证实施是否完全符合规范。捕获缺失的需求和过度构建。不信任实施者的报告——阅读实际代码。

2.  **代码质量审查** - 仅在规范合规性通过后运行。审查代码的清晰度、测试覆盖率、可维护性。

这捕获了常见的故障模式，即代码写得很好但不符合要求。审查是循环的，不是一次性的: 如果审查者发现问题，实施者修复它们，然后审查者再次检查。

其他 subagent 工作流改进:
- Controller 向 workers 提供完整的任务文本 (而不是文件引用)
- Workers 可以在工作之前和期间提出澄清问题
- 报告完成前的自我审查清单
- 计划在开始时读取一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新提示模板:
- `implementer-prompt.md` - 包括自我审查清单，鼓励提问
- `spec-reviewer-prompt.md` - 针对需求的怀疑验证
- `code-quality-reviewer-prompt.md` - 标准代码审查

**集成了工具的 Debugging 技术**

`systematic-debugging` 现在捆绑了支持技术和工具:
- `root-cause-tracing.md` - 通过调用栈向后追踪 bugs
- `defense-in-depth.md` - 在多层添加验证
- `condition-based-waiting.md` - 用条件轮询替换任意超时
- `find-polluter.sh` - 二分脚本以查找哪个测试产生污染
- `condition-based-waiting-example.ts` - 来自实际调试会话的完整实现

**测试反模式参考**

`test-driven-development` now includes `testing-anti-patterns.md` covering:
- 测试 mock 行为而不是真实行为
- 向生产类添加仅用于测试的方法
- 在不了解依赖关系的情况下 Mocking
- 隐藏结构假设的不完整 mocks

**Skill 测试基础设施**

用于验证 skill 行为的三个新测试框架:

`tests/skill-triggering/` - 验证 skills 从朴素提示触发而不需要显式命名。测试 6 个 skills 以确保仅描述就足够了。

`tests/claude-code/` - 使用 `claude -p` 进行 headless 测试的集成测试。通过会话记录 (JSONL) 分析验证 skill 使用。包括用于成本跟踪的 `analyze-token-usage.py`。

`tests/subagent-driven-dev/` - 具有两个完整测试项目的端到端工作流验证:
- `go-fractals/` - 带有 Sierpinski/Mandelbrot 的 CLI 工具 (10 tasks)
- `svelte-todo/` - 带有 localStorage 和 Playwright 的 CRUD app (12 tasks)

### 重大变更 (Major Changes)

**DOT 流程图作为可执行规范**

使用 DOT/GraphViz 流程图重写了关键 skills 作为权威的流程定义。散文成为支持内容。

**描述陷阱 (The Description Trap)** (文档化于 `writing-skills`): 发现当描述包含工作流摘要时，skill 描述会覆盖流程图内容。Claude 遵循简短描述而不是阅读详细流程图。修复: 描述必须仅用于触发 ("Use when X") 而没有流程细节。

**using-superpowers 中的 Skill 优先级**

当多个 skills 适用时，流程 skills (brainstorming, debugging) 现在明确先于实施 skills。"Build X" 首先触发 brainstorming，然后是领域 skills。

**brainstorming 触发加强**

描述更改为祈使句: "You MUST use this before any creative work—creating features, building components, adding functionality, or modifying behavior."

### 破坏性变更 (Breaking Changes)

**Skill 合并** - 六个独立 skills 合并:
- `root-cause-tracing`, `defense-in-depth`, `condition-based-waiting` → 捆绑在 `systematic-debugging/`
- `testing-skills-with-subagents` → 捆绑在 `writing-skills/`
- `testing-anti-patterns` → 捆绑在 `test-driven-development/`
- `sharing-skills` 移除 (已过时)

### 其他改进 (Other Improvements)

- **render-graphs.js** - 从 skills 中提取 DOT 图表并渲染为 SVG 的工具
- **Rationalizations table** in using-superpowers - 可扫描格式，包括新条目: "I need more context first", "Let me explore first", "This feels productive"
- **docs/testing.md** - 使用 Claude Code 集成测试测试 skills 的指南

---

## v3.6.2 (2025-12-03)

### 修复 (Fixed)

- **Linux 兼容性**: 修复 polyglot hook wrapper (`run-hook.cmd`) 以使用符合 POSIX 的语法
  - 将第 16 行的 bash 特有的 `${BASH_SOURCE[0]:-$0}` 替换为标准的 `$0`
  - 解决 `/bin/sh` 为 dash 的 Ubuntu/Debian 系统上的 "Bad substitution" 错误
  - 修复 #141

---

## v3.5.1 (2025-11-24)

### 变更 (Changed)

- **OpenCode Bootstrap 重构**: 从 `chat.message` hook 切换到 `session.created` 事件进行 bootstrap 注入
  - Bootstrap 现在在会话创建时通过带有 `noReply: true` 的 `session.prompt()` 注入
  - 显式告诉模型 using-superpowers 已加载，以防止冗余 skill 加载
  - 将 bootstrap 内容生成合并到共享的 `getBootstrapContent()` 辅助函数中
  - 更清晰的单一实现方法 (删除了回退模式)

---

## v3.5.0 (2025-11-23)

### 新增 (Added)

- **OpenCode 支持**: 用于 OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具: `use_skill` 和 `find_skills`
  - 消息插入模式用于 skill 在上下文压缩后的持久性
  - 通过 chat.message hook 自动注入上下文
  - 在 session.compacted 事件上自动重新注入
  - 三层 skill 优先级: project > personal > superpowers
  - 项目本地 skills 支持 (`.opencode/skills/`)
  - 共享核心模块 (`lib/skills-core.js`) 用于与 Codex 的代码重用
  - 具有适当隔离的自动化测试套件 (`tests/opencode/`)
  - 特定平台的文档 (`docs/README.opencode.md`, `docs/README.codex.md`)

### 变更 (Changed)

- **重构 Codex 实现**: 现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - skill 发现和解析的单一事实来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进文档**: 重写 README 以清楚地解释问题/解决方案
  - 删除了重复部分和冲突信息
  - 添加了完整的工作流描述 (brainstorm → plan → execute → finish)
  - 简化的平台安装说明
  - 强调 skill 检查协议胜过自动激活声明

---

## v3.4.1 (2025-10-31)

### 改进 (Improvements)

- 优化 superpowers bootstrap 以消除冗余 skill 执行。`using-superpowers` skill 内容现在直接在会话上下文中提供，并将明确指导仅对其他 skills 使用 Skill 工具。这减少了开销并防止了代理即使在会话开始时已有内容却仍手动执行 `using-superpowers` 的混乱循环。

## v3.4.0 (2025-10-30)

### 改进 (Improvements)

- 简化 `brainstorming` skill 以回归原始的对话愿景。删除了带有正式清单的重量级 6 阶段流程，转而支持自然对话: 一次问一个问题，然后以 200-300 字的部分与验证呈现设计。保留文档和实施移交功能。

## v3.3.1 (2025-10-28)

### 改进 (Improvements)

- 更新 `brainstorming` skill 以要求在提问前进行自主侦察，鼓励推荐驱动的决策，并防止代理将优先级委派回人类。
- 遵循 Strunk 的 "Elements of Style" 原则 (省略不必要的词，将否定形式转换为肯定形式，改进平行结构) 对 `brainstorming` skill 应用写作清晰度改进。

### Bug 修复

- 阐明 `writing-skills` 指南，使其指向正确的特定于代理的个人 skill 目录 (Claude Code 为 `~/.claude/skills`，Codex 为 `~/.codex/skills`)。

## v3.3.0 (2025-10-28)

### 新功能 (New Features)

**实验性 Codex 支持**
- 添加统一的 `superpowers-codex` 脚本，带有 bootstrap/use-skill/find-skills 命令
- Cross-platform Node.js implementation (在 Windows, macOS, Linux 上工作)
- 命名空间 skills: `superpowers:skill-name` 用于 superpowers skills, `skill-name` 用于 personal
- 当名称匹配时，Personal skills 覆盖 superpowers skills
- 清晰的 skill 显示: 显示名称/描述而没有原始 frontmatter
- 有用的上下文: 显示每个 skill 的支持文件目录
- Codex 的工具映射: TodoWrite→update_plan, subagents→manual fallback, etc.
- 与最小 AGENTS.md 集成的 Bootstrap 用于自动启动
- 特定于 Codex 的完整安装指南和 bootstrap 说明

**与 Claude Code 集成的主要区别:**
- 单一统一脚本而不是单独的工具
- Codex 特定等效项的工具替换系统
- 简化的 subagent 处理 (手动工作而不是委派)
- 更新的术语: "Superpowers skills" 而不是 "Core skills"

### 添加的文件 (Files Added)
- `.codex/INSTALL.md` - Codex 用户安装指南
- `.codex/superpowers-bootstrap.md` - 带有 Codex 适配的 Bootstrap 说明
- `.codex/superpowers-codex` - 具有所有功能的统一 Node.js 可执行文件

**注意:** Codex 支持是实验性的。该集成提供核心 superpowers 功能，但可能需要根据用户反馈进行改进。

## v3.2.3 (2025-10-23)

### 改进 (Improvements)

**更新 using-superpowers skill 以使用 Skill 工具而不是 Read 工具**
- 将 skill 调用说明从 Read 工具更改为 Skill 工具
- 更新描述: "using Read tool" → "using Skill tool"
- 更新步骤 3: "Use the Read tool" → "Use the Skill tool to read and run"
- 更新合理化列表: "Read the current version" → "Run the current version"


The Skill tool is the proper mechanism for invoking skills in Claude Code. This update corrects the bootstrap instructions to guide agents toward the correct tool.

### 文件变更 (Files Changed)
- 更新: `skills/using-superpowers/SKILL.md` - 将工具引用从 Read 更改为 Skill

## v3.2.2 (2025-10-21)

### 改进 (Improvements)

**加强 using-superpowers skill 以抵制代理合理化**
- 添加了 EXTREMELY-IMPORTANT 块，使用绝对语言关于强制性 skill 检查
  - "If even 1% chance a skill applies, you MUST read it"
  - "You do not have a choice. You cannot rationalize your way out."
- 添加 MANDATORY FIRST RESPONSE PROTOCOL 清单
  - 代理必须在任何响应之前完成的 5 步流程
  - 明确的 "responding without this = failure" 后果
- 添加带有 8 个特定逃避模式的 Common Rationalizations 部分
  - "This is just a simple question" → WRONG
  - "I can check files quickly" → WRONG
  - "Let me gather information first" → WRONG
  - Plus 5 more common patterns observed in agent behavior

这些更改解决了观察到的代理行为，即尽管有明确的指示，他们仍围绕 skill 使用进行合理化。强有力的语言和先发制人的反驳旨在使不合规变得更加困难。

### 文件变更 (Files Changed)
- 更新: `skills/using-superpowers/SKILL.md` - 添加三层强制措施以防止 skill 跳过合理化

## v3.2.1 (2025-10-20)

### 新功能 (New Features)

**Code reviewer agent 现在包含在插件中**
- 添加 `superpowers:code-reviewer` agent 到插件的 `agents/` 目录
- Agent 提供针对计划和编码标准的系统代码审查
- 以前要求用户拥有个人 agent 配置
- 所有 skill 引用更新为使用命名空间的 `superpowers:code-reviewer`
- 修复 #55

### 文件变更 (Files Changed)
- 新增: `agents/code-reviewer.md` - 带有审查清单和输出格式的 Agent 定义
- 更新: `skills/requesting-code-review/SKILL.md` - 引用 `superpowers:code-reviewer`
- 更新: `skills/subagent-driven-development/SKILL.md` - 引用 `superpowers:code-reviewer`

## v3.2.0 (2025-10-18)

### 新功能 (New Features)

**Brainstorming 工作流中的设计文档**
- 添加 Phase 4: Design Documentation 到 brainstorming skill
- 设计文档现在在实施之前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复了 skill 转换期间丢失的原始 brainstorming 命令的功能
- 在 worktree 设置和实施计划之前写入文档
- 用 subagent 测试以验证在时间压力下的合规性

### 重大变更 (Breaking Changes)

**Skill 引用命名空间标准化**
- 所有内部 skill 引用现在使用 `superpowers:` 命名空间前缀
- 更新格式: `superpowers:test-driven-development` (以前只是 `test-driven-development`)
- 影响所有 REQUIRED SUB-SKILL, RECOMMENDED SUB-SKILL, 和 REQUIRED BACKGROUND 引用
- 与使用 Skill 工具调用 skills 的方式保持一致
- 文件更新: brainstorming, executing-plans, subagent-driven-development, systematic-debugging, testing-skills-with-subagents, writing-plans, writing-skills

### 改进 (Improvements)

**设计 vs 实施计划命名**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实施计划继续使用现有的 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，具有清晰的命名区别

## v3.1.1 (2025-10-17)

### Bug 修复

- **修复 README 中的命令语法** (#44) - 更新所有命令引用以使用正确的命名空间语法 (`/superpowers:brainstorm` 而不是 `/brainstorm`)。插件提供的命令由 Claude Code 自动命名空间，以避免插件之间的冲突。

## v3.1.0 (2025-10-17)

### 重大变更 (Breaking Changes)

**Skill 名称标准化为小写**
- 所有 skill frontmatter `name:` 字段现在使用与目录名匹配的小写 kebab-case
- 示例: `brainstorming`, `test-driven-development`, `using-git-worktrees`
- 所有 skill 公告和交叉引用更新为小写格式
- 这确保了目录名、frontmatter 和文档之间的一致命名

### 新功能 (New Features)

**增强的 brainstorming skill**
- 添加显示阶段、活动和工具用法的快速参考表
- 添加用于跟踪进度的可复制工作流清单
- 为何时重新审视早期阶段添加决策流程图
- 添加用于跟踪进度的可解析 Dot 图表
- 添加带有具体示例的全面 AskUserQuestion 工具指南
- 添加 "Question Patterns" 部分解释何时使用结构化与开放式问题
- 将关键原则重组为可扫描表

**Anthropic 最佳实践集成**
- 添加 `skills/writing-skills/anthropic-best-practices.md` - 官方 Anthropic skill 作者指南
- 在 writing-skills SKILL.md 中引用以获得全面指导
- 提供用于渐进式披露、工作流和评估的模式

### 改进 (Improvements)

**Skill 交叉引用清晰度**
- 所有 skill 引用现在使用显式要求标记:
  - `**REQUIRED BACKGROUND:**` - 必须了解的先决条件
  - `**REQUIRED SUB-SKILL:**` - 必须在工作流中使用的 Skills
  - `**Complementary skills:**` - 可选但有用的相关 skills
- 删除了旧路径格式 (`skills/collaboration/X` → just `X`)
- 更新了带有分类关系 (Required vs Complementary) 的集成部分
- 更新了带有最佳实践的交叉引用文档

**与 Anthropic 最佳实践保持一致**
- 修复描述语法和语态 (完全第三人称)
- 添加用于扫描的快速参考表
- 添加 Claude 可以复制和跟踪的工作流清单
- 适当使用流程图用于非显而易见的决策点
- 改进的可扫描表格式
- 所有 skills 远低于 500 行建议

### Bug 修复

- **重新添加丢失的命令重定向** - 恢复了在 v3.0 迁移中意外删除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复 `defense-in-depth` 名称不匹配 (was `Defense-in-Depth-Validation`)
- 修复 `receiving-code-review` 名称不匹配 (was `Code-Review-Reception`)
- 修复 `commands/brainstorm.md` 引用正确的 skill 名称
- 删除了对不存在的相关 skills 的引用

### 文档 (Documentation)

**writing-skills 改进**
- 更新带有显式要求标记的交叉引用指南
- 添加对 Anthropic 官方最佳实践的引用
- 改进显示正确 skill 引用格式的示例

## v3.0.1 (2025-10-16)

### 变更 (Changes)

我们现在使用 Anthropic 的第一方 skills 系统！

## v2.0.2 (2025-10-12)

### Bug 修复

- **修复当本地 skills repo 领先于上游时的错误警告** - 当本地仓库有领先于上游的提交时，初始化脚本错误地警告 "New skills available from upstream"。该逻辑现在正确区分三种 git 状态: 本地落后 (应更新), 本地领先 (无警告), 和偏离 (由于应警告)。

## v2.0.1 (2025-10-12)

### Bug 修复

- **修复插件上下文中的 session-start hook 执行** (#8, PR #9) - 该 hook 以前因 "Plugin hook error" 静默失败，阻止了 skills 上下文加载。修复方法:
  - 当 BASH_SOURCE 在 Claude Code 的执行上下文中未绑定时使用 `${BASH_SOURCE[0]:-$0}` 回退
  - 在过滤状态标志时添加 `|| true` 以优雅地处理空的 grep 结果

---

## v2.0.0 (2025-10-10)

### 重大变更 (Breaking Changes)

**Skills 仓库分离**
Skills 不再位于插件中。它们已被移至 [obra/superpowers-skills](https://github.com/obra/superpowers-skills) 的单独仓库。

**这对你意味着什么:**
- **首次安装:** 插件自动克隆 skills 到 `~/.config/superpowers/skills/`
- **Forking:** 在设置期间，如果你安装了 (`gh`)，将提供 fork skills repo 的选项
- **更新:** Skills 在会话开始时自动更新 (尽可能快进)
- **贡献:** 在分支上工作，本地提交，提交 PR 到上游
- **不再有 shadowing:** 旧的两层系统 (personal/core) 被单仓库分支工作流取代

**迁移:**
如果你有现有的安装:
1. 你的旧 `~/.config/superpowers/.git` 将备份到 `~/.config/superpowers/.git.bak`
2. 旧 skills 将备份到 `~/.config/superpowers/skills.bak`
3. obra/superpowers-skills 的新克隆将创建在 `~/.config/superpowers/skills/`

### 移除的功能 (Removed Features)
- **Personal superpowers overlay system** - 被 git 分支工作流取代
- **setup-personal-superpowers hook** - 被 initialize-skills.sh 取代

### 新功能 (New Features)

**Skills 仓库基础设施**
- **自动 Clone & Setup** (`lib/initialize-skills.sh`): 首次运行时克隆 obra/superpowers-skills
- **自动更新 (Auto-Update)**: 每次会话开始时从 tracking remote 获取

**新 Skills**
- **问题解决 Skills** (`skills/problem-solving/`): collision-zone-thinking, inversion-exercise, meta-pattern-recognition, scale-game, simplification-cascades, when-stuck
- **研究 Skills** (`skills/research/`): tracing-knowledge-lineages
- **架构 Skills** (`skills/architecture/`): preserving-productive-tensions

**Skills 改进**
- **using-skills (formerly getting-started)**: 用祈使语气完全重写，前置关键规则
- **writing-skills**: 改进 CSO (Claude Search Optimization) 指南
- **sharing-skills**: 更新为新分支和 PR 工作流

**工具改进**
- **find-skills**: 现在输出带有 /SKILL.md 后缀的完整路径，使路径可直接与 Read 工具一起使用

### 文档 (Documentation)

**README**
- 更新为新 skills 仓库架构
- 突出链接到 superpowers-skills repo
- 更新自动更新说明
- 修复 skill 名称和引用
- 更新 Meta skills 列表

**新增 (Added):**
- `lib/initialize-skills.sh` - Skills repo 初始化和自动更新
- `docs/TESTING-CHECKLIST.md` - 手动测试场景
- `.claude-plugin/marketplace.json` - 本地测试配置

**移除 (Removed):**
- `skills/` 目录 (82 个文件) - 现在在 obra/superpowers-skills 中
- `scripts/` 目录 - 现在在 obra/superpowers-skills/skills/using-skills/ 中
- `hooks/setup-personal-superpowers.sh` - 已过时

**修改 (Modified):**
- `hooks/session-start.sh` - 使用来自 ~/.config/superpowers/skills 的 skills
- `commands/brainstorm.md` - 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 更新路径到 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 针对新架构的完整重写

### 提交历史 (Commit History)

本版本包括:
- 20 多个用于 skills 仓库分离的提交
- PR #1: 受 Amplifier 启发的解决问题和研究 skills
- PR #2: 个人 superpowers 叠加系统 (后来被取代)
- 多个 skill 改进和文档提升

## 升级指南 (Upgrade Instructions)

### 全新安装 (Fresh Install)

```bash
# In Claude Code
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件会自动处理一切。

### 从 v1.x 升级

1. **备份你的个人 skills** (如果你有的话):
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件:**
   ```bash
   /plugin update superpowers
   ```

3. **在下次会话开始时:**
   - 旧安装将自动备份
   - 将克隆新的 skills repo
   - 如果你有 GitHub CLI，你将被提供 fork 的选项

4. **迁移个人 skills** (如果你有的话):
   - 在本地 skills repo 中创建一个分支
   - 从备份中复制你的个人 skills
   - 提交并推送到你的 fork
   - 考虑通过 PR 贡献回上游

## 下一步工作 (What's Next)

### 对于用户 (For Users)

- 探索新的解决问题 skills
- 尝试基于分支的工作流来改进 skill
- 将 skills 贡献回社区

### 对于贡献者 (For Contributors)

- Skills 仓库现在位于 https://github.com/obra/superpowers-skills
- Fork → Branch → PR 工作流
- 参见 skills/meta/writing-skills/SKILL.md 了解文档的 TDD 方法

## 已知问题 (Known Issues)

目前没有已知问题。

## 致谢 (Credits)

- 受 Amplifier 模式启发的解决问题 skills
- 社区贡献和反馈
- 对 skill 有效性的广泛测试和迭代

---

**完整变更日志:** https://github.com/obra/superpowers/compare/dd013f6...main
**Skills 仓库:** https://github.com/obra/superpowers-skills
**问题反馈:** https://github.com/obra/superpowers/issues
