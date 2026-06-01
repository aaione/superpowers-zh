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

---

## v5.1.0 (2026-04-30)

### Removals

- **移除旧版 slash commands** — `/brainstorm`、`/execute-plan` 和 `/write-plan` 已被移除。它们是已弃用的存根，仅告诉用户调用对应的 skill。改为直接调用 `superpowers:brainstorming`、`superpowers:executing-plans` 和 `superpowers:writing-plans`。(#1188)
- **移除 `superpowers:code-reviewer` 命名 agent** — 该 agent 是插件唯一的命名 agent，仅被两个 skills 使用，而仓库中所有其他 reviewer/implementer subagent 都是使用 `general-purpose` 配合提示模板来分派的。该 agent 的角色和清单已合并到 `skills/requesting-code-review/code-reviewer.md` 中，作为一个独立的 Task 分派模板。任何分派 `Task (superpowers:code-reviewer)` 的地方都应改为使用 `Task (general-purpose)` 配合提示模板。(PR #1299)
- **从 skills 中移除 Integration 部分** — 这些是 agents 没有原生 skills 系统时代的遗留内容，对引导行为没有帮助。

### Worktree Skills 重写

`using-git-worktrees` 和 `finishing-a-development-branch` 现在能够检测 agent 是否已在隔离的 worktree 中运行，并优先使用 harness 的原生 worktree 控制功能，然后才回退到 `git worktree`。行为经过 TDD 验证，并在五个 harness 上进行了跨平台检查。(PRI-974, PR #1121)

- **环境检测** — 两个 skill 都在执行任何操作前检查 `GIT_DIR != GIT_COMMON`；如果已在链接的 worktree 中，则完全跳过创建。子模块防护可防止误检测。
- **创建 worktree 前征得同意** — `using-git-worktrees` 不再隐式创建 worktree；skill 会先询问用户。修复 #991（subagent-driven-development 曾在未经同意的情况下自动创建 worktree）。
- **原生工具优先 (Step 1a)** — 当 harness 暴露自己的 worktree 工具（如 Codex）时，skill 会优先使用它。用户表达的偏好会得到尊重。
- **基于来源的清理** — `finishing-a-development-branch` 仅清理 `.worktrees/` 内的 worktree（由 superpowers 创建）；其他位置的 worktree 不受影响。修复 #940（选项 2 曾错误清理 worktree）、#999（合并后移除的顺序问题）和 #238（在 `git worktree remove` 前需要 `cd` 到 repo 根目录）。
- **Detached HEAD 处理** — 当没有可合并的分支时，finishing 菜单折叠为两个选项。
- **硬编码的 `/Users/jesse` 路径** — skill 示例中的路径已替换为通用占位符。(#858, PR #1122)

### AI Agent 贡献者指南

`CLAUDE.md` 顶部新增了两个部分（符号链接到 `AGENTS.md`），直接面向 AI agents。对该仓库最近 100 个已关闭 PR 的审计显示，94% 的拒绝率是由 AI 生成的垃圾内容驱动的：agents 不阅读 PR 模板、打开重复 PR、编造问题描述，或推送 fork 或特定领域的更改到上游。

- **提交前清单** — 阅读 PR 模板、搜索现有 PR、验证真实问题存在、确认更改属于核心、在提交前向人类合作伙伴展示完整 diff。
- **我们不会接受的内容** — 第三方依赖、skill 内容的"合规性"重写、项目特定配置、批量 PR、投机性修复、特定领域的 skills、fork 特定更改、编造内容以及捆绑的不相关更改。
- **新 harness PR 需要会话记录** — 过去大多数新 harness 集成只是复制 skill 文件或使用 `npx skills` 包装，而不是在会话开始时加载 `using-superpowers` bootstrap。现在需要验收测试（"Let's make a react todo list" 必须在全新会话中自动触发 `brainstorming`）和完整的会话记录。

### Codex 插件镜像工具

新的 `sync-to-codex-plugin` 脚本将 superpowers 镜像到 OpenAI Codex 插件市场，名称为 `prime-radiant-inc/openai-codex-plugins`。路径无关、用户无关，任何团队成员都可以运行。(PR #1165)

- 每次运行将 fork 克隆到临时目录，内联重新生成覆盖层，并打开 PR；从脚本自身位置自动检测上游，并预检 `rsync`/`git`/`gh auth`/`python3`。
- `--bootstrap` 标志用于首次设置；`EXCLUDES` 模式锚定到源根目录；`assets/` 被排除。
- 镜像 `CODE_OF_CONDUCT.md`；丢弃 `agents/openai.yaml` 覆盖层。
- 在镜像的 `plugin.json` 中植入 `interface.defaultPrompt`。(PR #1180 by @arittr)
- Codex 插件文件已提交到源仓库，因此同步脚本使用规范版本；Codex 市场元数据被保留。

### OpenCode

- **Bootstrap 内容在模块级缓存** — `getBootstrapContent()` 曾在每次 agent 步骤调用 `fs.existsSync` + `fs.readFileSync` + frontmatter 正则表达式（`experimental.chat.messages.transform` hook 在 OpenCode 的 agent 循环中每个步骤都会触发）。现在只读取一次，在会话生命周期内缓存，对文件缺失情况使用 null 哨兵值。15 个回归测试覆盖了缓存行为、fs 调用计数、注入防护、文件缺失哨兵和缓存重置。(修复 #1202)
- **集成测试现代化**。
- **README 中的安装注意事项已澄清**。

### Code Review 整合

`requesting-code-review` 现在是自包含的：角色、清单和分派模板位于 `skills/requesting-code-review/code-reviewer.md`，skill 直接分派 `Task (general-purpose)`。(PR #1299)

- **单一事实来源** — 以前存在于 `agents/code-reviewer.md` 和 skill 占位模板中（且独立漂移）的角色/清单现在合并为一个文件。
- **`subagent-driven-development` 跟进** — 其 `code-quality-reviewer-prompt.md` 现在分派 `Task (general-purpose)` 而不是命名 agent。
- **添加行为测试** — `tests/claude-code/test-requesting-code-review.sh` 在一个小型项目中植入真实 bug（SQL 注入、明文密码处理、凭据日志记录），并断言分派的 reviewer 在 Critical/Important 严重级别标记每个植入的问题并拒绝批准 diff。
- **Codex 和 Copilot 变通文档精简** — `references/codex-tools.md` 和 `references/copilot-tools.md` 中的 "Named agent dispatch" 部分记录了如何将命名 agent 展平为通用分派。由于不再发布命名 agent，变通方案已无必要；两个部分均已移除。

### Subagent-Driven Development

- **不再每 3 个任务暂停** — `requesting-code-review` 中 "每批审查（3 个任务）" 的节奏（最初用于 `executing-plans`）泄漏到了 `subagent-driven-development` 中。替换为 "每个任务或在自然检查点" 加上显式的持续执行指令。
- **SDD 集成测试现在实际运行断言** — 三个独立 bug 导致测试在打印任何验证结果之前静默退出：工作目录路径中未解析的 `..` 段、`set -euo pipefail` 与 `find | sort | head -1` 的交互（生产者端的 SIGPIPE 杀死了脚本），以及 `claude -p` 调用中缺少 `--plugin-dir`（导致测试加载已安装的插件而不是工作树）。三个问题全部修复；六个验证测试现在针对真实的端到端 SDD 运行实际执行。

### Cursor

- **Windows SessionStart hook** — 通过 `run-hook.cmd` 路由，而不是直接调用无扩展名的 `session-start` 脚本。修复 Windows 将文件在编辑器中打开而不是执行的问题。还移除了 `hooks-cursor.json` 中意外的 UTF-8 BOM。

### Gemini CLI

- **Subagent 分派映射** — Gemini 的 `Task` 分派现在映射到 `@agent-name` / `@generalist`，并记录了独立任务的并行 subagent 分派。

### Skills

- **术语清理** — 在 skill 内容中进行。

### 文档与安装

- **Factory Droid 安装说明** 添加到 README。
- **README 中的快速开始安装链接**。(PR #1293 by @arittr)
- **Codex 插件安装指南** 更新。(PR #1288 by @arittr)
- **Codex `wait` 映射修正** 为工具参考中的 `wait_agent`。
- **安装顺序重新组织**；Codex 安装说明清理。
- **移除残留的 `CHANGELOG.md`**，改为以 `RELEASE-NOTES.md` 作为单一来源。(PR #1163 by @shaanmajid)
- **Discord 邀请链接** 修复；社区部分添加了版本发布公告链接和详细的 Discord 描述。

### 社区

- @shaanmajid — 移除残留的 `CHANGELOG.md` (PR #1163)
- @arittr — README 快速开始安装链接 (#1293)、Codex 插件安装指南 (#1288)、`sync-to-codex-plugin` `interface.defaultPrompt` 种子 (#1180)

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 支持

- **SessionStart 上下文注入** — Copilot CLI v1.0.11 在 sessionStart hook 输出中添加了对 `additionalContext` 的支持。Session-start hook 现在检测 `COPILOT_CLI` 环境变量并发出 SDK 标准的 `{ "additionalContext": "..." }` 格式，使 Copilot CLI 用户在会话开始时获得完整的 superpowers bootstrap。(原始修复由 @culinablaz 在 PR #910 中完成)
- **工具映射** — 添加了 `references/copilot-tools.md`，包含完整的 Claude Code 到 Copilot CLI 工具等价表
- **Skill 和 README 更新** — 将 Copilot CLI 添加到 `using-superpowers` skill 的平台说明和 README 安装部分

### OpenCode 修复

- **Skills 路径一致性** — bootstrap 文本不再宣传与运行时路径不匹配的 `configDir/skills/superpowers/` 路径。Agent 应使用原生 `skill` 工具，而不是通过路径导航文件。测试现在使用从单一事实来源派生的一致路径。(#847, #916)
- **Bootstrap 作为用户消息** — 将 bootstrap 注入从 `experimental.chat.system.transform` 移至 `experimental.chat.messages.transform`，前置到第一个用户消息而不是添加系统消息。避免了每轮重复系统消息导致的 token 膨胀 (#750)，并修复了与 Qwen 等在多个系统消息上会出问题的模型的兼容性 (#894)。

## v5.0.6 (2026-03-24)

### 内联自审查替代 Subagent 审查循环

Subagent 审查循环（分派新 agent 审查计划/规范）使执行时间翻倍（约 25 分钟开销），但未显著提高计划质量。跨越 5 个版本、每个版本 5 次试验的回归测试显示，无论是否运行审查循环，质量分数完全相同。

- **brainstorming** — 将 Spec Review Loop（subagent 分派 + 3 次迭代上限）替换为内联 Spec Self-Review 清单：占位符扫描、内部一致性、范围检查、歧义检查
- **writing-plans** — 将 Plan Review Loop（subagent 分派 + 3 次迭代上限）替换为内联 Self-Review 清单：规范覆盖、占位符扫描、类型一致性
- **writing-plans** — 添加了显式的 "No Placeholders" 部分，定义了计划失败的情况（TBD、模糊描述、未定义引用、"similar to Task N"）
- 自审查在大约 30 秒内每次运行捕获 3-5 个真实 bug，而不是约 25 分钟，缺陷率与 subagent 方式相当

### Brainstorm 服务器

- **会话目录重组** — brainstorm 服务器会话目录现在包含两个同级子目录：`content/`（向浏览器提供的 HTML 文件）和 `state/`（事件、server-info、pid、日志）。以前，服务器状态和用户交互数据与提供的内容一起存储，使得它们可以通过 HTTP 访问。`screen_dir` 和 `state_dir` 路径都包含在服务器启动的 JSON 中。(由吉田仁报告)

### Bug 修复

- **Owner-PID 生命周期修复** — brainstorm 服务器的 owner-PID 监控有两个 bug 导致在 60 秒内误关闭：(1) 来自跨用户 PID（Tailscale SSH 等）的 EPERM 被视为"进程已死"，(2) 在 WSL 上，grandparent PID 解析为在第一次生命周期检查之前退出的短生命期子进程。修复方式为将 EPERM 视为"存活"，并在启动时验证 owner PID — 如果已死，则禁用监控，服务器依赖 30 分钟空闲超时。这也移除了 `start-server.sh` 中 Windows/MSYS2 特定的特殊处理，因为服务器现在通用处理它。(#879)
- **writing-skills** — 修正了 SKILL.md frontmatter 仅支持"两个字段"的错误声明；现在说"两个必需字段"并链接到 agentskills.io 规范以获取所有支持的字段 (PR #882 by @arittr)

### Codex App 兼容性

- **codex-tools** — 添加了命名 agent 分派映射，记录如何将 Claude Code 的命名 agent 类型转换为 Codex 的 `spawn_agent` 和 worker 角色 (PR #647 by @arittr)
- **codex-tools** — 为 worktree 感知的 skills 添加了环境检测和 Codex App finishing 部分 (by @arittr)
- **设计规范** — 添加了 Codex App 兼容性设计规范 (PRI-823)，涵盖只读环境检测、worktree 安全的 skill 行为和沙盒回退模式 (by @arittr)

## v5.0.5 (2026-03-17)

### Bug 修复

- **Brainstorm 服务器 ESM 修复** — 将 `server.js` 重命名为 `server.cjs`，使 brainstorming 服务器在 Node.js 22+ 上正确启动，因为根 `package.json` 的 `"type": "module"` 导致 `require()` 失败。(PR #784 by @sarbojitrana，修复 #774, #780, #783)
- **Windows 上的 Brainstorm owner-PID** — 在 Windows/MSYS2 上跳过 PID 生命周期监控，因为 PID 命名空间对 Node.js 不可见，防止服务器在 60 秒后自终止。(#770，文档来自 PR #768 by @lucasyhzlu-debug)
- **stop-server.sh 可靠性** — 在报告成功之前验证服务器进程确实已终止。SIGTERM + 2 秒等待 + SIGKILL 回退。(#723)

### 变更

- **执行交接** — 在计划编写后恢复用户在 subagent-driven 和 inline 执行之间的选择。推荐使用 Subagent-driven 但不再是强制的。

## v5.0.4 (2026-03-16)

### 审查循环优化

通过消除不必要的审查轮次和收紧审查者焦点，大幅减少 token 使用量并加速规范和计划审查。

- **单次全计划审查** — 计划审查者现在一次性审查完整计划，而不是逐块审查。移除了所有块相关概念（`## Chunk N:` 标题、1000 行块限制、逐块分派）。
- **提高了阻塞问题的门槛** — 规范和计划审查者提示现在包含 "Calibration" 部分：仅标记会在实施过程中导致实际问题的问题。措辞细微差异、风格偏好和格式挑剔不应阻止批准。
- **减少最大审查迭代次数** — 规范和计划审查循环从 5 次减少到 3 次。如果审查者校准正确，3 轮已足够。
- **精简审查者清单** — 规范审查者从 7 个类别精简到 5 个；计划审查者从 7 个精简到 4 个。移除了以格式为重点的检查（任务语法、块大小），转而关注实质内容（可构建性、规范对齐）。

### OpenCode

- **一行插件安装** — OpenCode 插件现在通过 `config` hook 自动注册 skills 目录。不需要符号链接或 `skills.paths` 配置。安装只需在 `opencode.json` 中添加一行。(PR #753)
- **添加了 `package.json`**，使 OpenCode 可以从 git 安装 superpowers 作为 npm 包。

### Bug 修复

- **验证服务器确实已停止** — `stop-server.sh` 现在在报告成功之前确认进程已终止。SIGTERM + 2 秒等待 + SIGKILL 回退。如果进程仍然存活则报告失败。(PR #751)
- **通用 agent 语言** — brainstorm companion 等待页面现在说 "the agent" 而不是 "Claude"。

## v5.0.3 (2026-03-15)

### Cursor 支持

- **Cursor hooks** — 添加了 `hooks/hooks-cursor.json`，使用 Cursor 的 camelCase 格式（`sessionStart`、`version: 1`），并更新了 `.cursor-plugin/plugin.json` 以引用它。修复了 `session-start` 中的平台检测，优先检查 `CURSOR_PLUGIN_ROOT`（Cursor 也可能设置 `CLAUDE_PLUGIN_ROOT`）。(基于 PR #709)

### Bug 修复

- **停止在 `--resume` 时触发 SessionStart hook** — 启动 hook 在恢复的会话上重新注入上下文，而这些会话的对话历史中已经有了上下文。Hook 现在仅在 `startup`、`clear` 和 `compact` 时触发。
- **Bash 5.3+ hook 挂起** — 在 `hooks/session-start` 中用 `printf` 替换了 heredoc（`cat <<EOF`）。修复了 macOS 上 Homebrew bash 5.3+ 因 bash 在 heredoc 中大变量扩展的回归问题导致的无限挂起。(#572, #571)
- **POSIX 安全的 hook 脚本** — 在 `hooks/session-start` 中用 `$0` 替换了 `${BASH_SOURCE[0]:-$0}`。修复了在 `/bin/sh` 为 dash 的 Ubuntu/Debian 上的 "Bad substitution" 错误。(#553)
- **可移植的 shebangs** — 在所有 shell 脚本中用 `#!/usr/bin/env bash` 替换了 `#!/bin/bash`。修复了在 NixOS、FreeBSD 和安装了 Homebrew bash 的 macOS 上 `/bin/bash` 过旧或缺失的执行问题。(#700)
- **Windows 上的 Brainstorm 服务器** — 自动检测 Windows/Git Bash（`OSTYPE=msys*`、`MSYSTEM`）并切换到前台模式，修复了由 `nohup`/`disown` 进程回收导致的静默服务器失败。(#737)
- **Codex 文档修复** — 在 Codex 文档中将已弃用的 `collab` 标志替换为 `multi_agent`。(PR #749)

## v5.0.2 (2026-03-11)

### 零依赖 Brainstorm 服务器

**移除了所有 vendored node_modules — server.js 现在完全自包含**

- 用使用内置 `http`、`fs` 和 `crypto` 模块的零依赖 Node.js 服务器替换了 Express/Chokidar/WebSocket 依赖
- 移除了约 1,200 行 vendored 的 `node_modules/`、`package.json` 和 `package-lock.json`
- 自定义 WebSocket 协议实现（RFC 6455 帧、ping/pong、正确的关闭握手）
- 原生 `fs.watch()` 文件监控替代 Chokidar
- 完整测试套件：HTTP 服务、WebSocket 协议、文件监控和集成测试

### Brainstorm 服务器可靠性

- **空闲 30 分钟后自动退出** — 当没有客户端连接时服务器关闭，防止孤立进程
- **Owner 进程追踪** — 服务器监控父 harness PID 并在拥有会话终止时退出
- **存活性检查** — skill 在重用现有实例之前验证服务器是否响应
- **编码修复** — 在提供的 HTML 页面上使用正确的 `<meta charset="utf-8">`

### Subagent 上下文隔离

- 所有委派 skills（brainstorming、dispatching-parallel-agents、requesting-code-review、subagent-driven-development、writing-plans）现在包含上下文隔离原则
- Subagents 仅接收所需的上下文，防止上下文窗口污染

## v5.0.1 (2026-03-10)

### Agentskills 合规

**Brainstorm-server 移入 skill 目录**

- 根据 [agentskills.io](https://agentskills.io) 规范，将 `lib/brainstorm-server/` 移至 `skills/brainstorming/scripts/`
- 所有 `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/` 引用替换为相对 `scripts/` 路径
- Skills 现在可以跨平台完全移植 — 不需要平台特定的环境变量来定位脚本
- `lib/` 目录已移除（曾是最后残留的内容）

### 新功能

**Gemini CLI 扩展**

- 通过仓库根目录的 `gemini-extension.json` 和 `GEMINI.md` 实现原生 Gemini CLI 扩展支持
- `GEMINI.md` 在会话开始时 @imports `using-superpowers` skill 和工具映射表
- Gemini CLI 工具映射参考 (`skills/using-superpowers/references/gemini-tools.md`) — 将 Claude Code 工具名称（Read、Write、Edit、Bash 等）翻译为 Gemini CLI 等效项（read_file、write_file、replace 等）
- 记录了 Gemini CLI 限制：不支持 subagent，skills 回退到 `executing-plans`
- 扩展根目录位于仓库根目录以实现跨平台兼容性（避免 Windows 符号链接问题）
- 安装说明已添加到 README

### 改进

**多平台 Brainstorm 服务器启动**

- `visual-companion.md` 中提供了各平台启动说明：Claude Code（默认模式）、Codex（通过 `CODEX_CI` 自动前台）、Gemini CLI（`--foreground` 配合 `is_background`）和其他环境的回退
- 服务器现在将启动 JSON 写入 `$SCREEN_DIR/.server-info`，以便 agents 即使在后台执行隐藏 stdout 时也能找到 URL 和端口

**Brainstorm 服务器依赖捆绑**

- `node_modules` 已 vendor 到仓库中，使得 brainstorm 服务器在全新插件安装时无需在运行时 `npm install` 即可立即工作
- 从捆绑的依赖中移除了 `fsevents`（仅 macOS 的原生二进制文件；chokidar 在没有它的情况下优雅回退）
- 如果缺少 `node_modules` 则通过 `npm install` 自动回退安装

**OpenCode 工具映射修复**

- `TodoWrite` → `todowrite`（之前错误映射为 `update_plan`）；已对照 OpenCode 源代码验证

### Bug 修复

**Windows/Linux: 单引号破坏 SessionStart hook** (#577, #529, #644, PR #585)

- hooks.json 中 `${CLAUDE_PLUGIN_ROOT}` 周围的单引号在 Windows 上失败（cmd.exe 不将单引号识别为路径分隔符），在 Linux 上也失败（单引号阻止变量展开）
- 修复：将单引号替换为转义的双引号 — 适用于 macOS bash、Windows cmd.exe、Windows Git Bash 和 Linux，无论路径中是否包含空格
- 已在 Windows 11 (NT 10.0.26200.0) 上使用 Claude Code 2.1.72 和 Git for Windows 验证

**Brainstorming 规范审查循环被跳过** (#677)

- 规范审查循环（分派 spec-document-reviewer subagent，迭代直到批准）存在于散文 "After the Design" 部分中，但在清单和流程图中缺失
- 由于 agents 更可靠地遵循流程图和清单而不是散文，规范审查步骤被完全跳过
- 在清单中添加了步骤 7（规范审查循环），并在 dot 图中添加了相应节点
- 使用 `claude --plugin-dir` 和 `claude-session-driver` 测试：worker 现在正确分派审查者

**Cursor 安装命令** (PR #676)

- 修复了 README 中的 Cursor 安装命令：`/plugin-add` → `/add-plugin`（通过 Cursor 2.5 发布公告确认）

**Brainstorming 中的用户审查门** (#565)

- 在规范完成和 writing-plans 交接之间添加了显式的用户审查步骤
- 用户必须在实施计划开始之前批准规范
- 清单、流程和散文已使用新门更新

**Session-start hook 每个平台仅发出一次上下文**

- Hook 现在检测它是在 Claude Code 还是其他平台中运行
- 对 Claude Code 发出 `hookSpecificOutput`，对其他平台发出 `additional_context` — 防止双重上下文注入

**Token 分析脚本中的 linting 修复**

- `tests/claude-code/analyze-token-usage.py` 中的 `except:` → `except Exception:`

### 维护

**移除死代码**

- 删除了 `lib/skills-core.js` 及其测试（`tests/opencode/test-skills-core.js`）— 自 2026 年 2 月以来未使用
- 从 `tests/opencode/test-plugin-loading.sh` 中移除了 skills-core 存在性检查

### 社区

- @karuturi — Claude Code 官方市场安装说明 (PR #610)
- @mvanhorn — session-start hook 双重发出修复，OpenCode 工具映射修复
- @daniel-graham — 裸 except 的 linting 修复
- PR #585 作者 — Windows/Linux hooks 引号修复

---

## v5.0.0 (2026-03-09)

### Breaking Changes

**Specs 和 plans 目录重组**

- Specs（brainstorming 输出）现在保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- Plans（writing-plans 输出）现在保存到 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- 用户对 spec/plan 位置的偏好覆盖这些默认值
- 所有内部 skill 引用、测试文件和示例路径已更新以匹配
- 迁移：如有需要，将现有文件从 `docs/plans/` 移至新位置

**在支持 subagent 的 harness 上强制使用 Subagent-driven development**

Writing-plans 不再提供 subagent-driven 和 executing-plans 之间的选择。在支持 subagent 的 harness（Claude Code、Codex）上，subagent-driven-development 是必须的。Executing-plans 保留给不支持 subagent 的 harness，并告知用户 Superpowers 在支持 subagent 的平台上效果更好。

**Executing-plans 不再批量执行**

移除了 "执行 3 个任务后停止审查" 模式。计划现在连续执行，仅在遇到阻塞问题时停止。

**Slash commands 已弃用**

`/brainstorm`、`/write-plan` 和 `/execute-plan` 现在显示弃用通知，引导用户使用对应的 skills。命令将在下一个主要版本中移除。

### 新功能

**可视化 Brainstorming Companion**

Brainstorming 会话的可选浏览器 companion。当某个主题可能受益于视觉辅助时，brainstorming skill 提议在浏览器窗口中与终端对话并排显示模型、图表、比较和其他内容。

- `lib/brainstorm-server/` — WebSocket 服务器，带有浏览器辅助库、会话管理脚本和暗/亮主题框架模板（"Superpowers Brainstorming" 附 GitHub 链接）
- `skills/brainstorming/visual-companion.md` — 服务器工作流、屏幕编写和反馈收集的渐进式披露指南
- Brainstorming skill 在其流程中添加了可视化 companion 决策点：在探索项目上下文之后，skill 评估即将到来的问题是否涉及视觉内容，并在自己的消息中提供 companion
- 每个问题的决策：即使在接受后，每个问题都会评估浏览器或终端哪个更合适
- `tests/brainstorm-server/` 中的集成测试

**文档审查系统**

使用 subagent 分派的规范和计划文档自动审查循环：

- `skills/brainstorming/spec-document-reviewer-prompt.md` — 审查者检查完整性、一致性、架构和 YAGNI
- `skills/writing-plans/plan-document-reviewer-prompt.md` — 审查者检查规范对齐、任务分解、文件结构和文件大小
- Brainstorming 在编写设计文档后分派规范审查者
- Writing-plans 在每个部分后包含基于块的计划审查循环
- 审查循环重复直到批准或在 5 次迭代后升级
- `tests/claude-code/test-document-review-system.sh` 中的端到端测试
- `docs/superpowers/` 中的设计规范和实施计划

**Skill 管道中的架构指导**

Design-for-isolation 和 file-size-awareness 指导已添加到 brainstorming、writing-plans 和 subagent-driven-development：

- **Brainstorming** — 新部分："Design for isolation and clarity"（清晰的边界、定义良好的接口、独立可测试的单元）和 "Working in existing codebases"（遵循现有模式、仅做针对性改进）
- **Writing-plans** — 新 "File Structure" 部分：在定义任务之前规划文件和职责。新 "Scope Check" 兜底：捕获应在 brainstorming 阶段分解的多子系统规范
- **SDD implementer** — 新 "Code Organization" 部分（遵循计划的文件结构、报告文件膨胀的担忧）和 "When You're in Over Your Head" 升级指导
- **SDD code quality reviewer** — 现在检查架构、单元分解、计划符合性和文件增长
- **Spec/plan reviewers** — 架构和文件大小已添加到审查标准
- **范围评估** — Brainstorming 现在评估项目是否对单个规范来说太大。多子系统请求会及早标记并分解为子项目，每个子项目有自己的 spec → plan → implementation 循环

**Subagent-driven development 改进**

- **模型选择** — 根据任务类型选择模型能力的指导：廉价模型用于机械性实施、标准模型用于集成、高能力模型用于架构和审查
- **Implementer 状态协议** — Subagents 现在报告 DONE、DONE_WITH_CONCERNS、BLOCKED 或 NEEDS_CONTEXT。Controller 适当处理每种状态：使用更多上下文重新分派、升级模型能力、拆分任务或升级给人类

### 改进

**指令优先级层次**

在 using-superpowers 中添加了显式的优先级排序：

1. 用户的显式指令（CLAUDE.md、AGENTS.md、直接请求）— 最高优先级
2. Superpowers skills — 覆盖默认系统行为
3. 默认系统提示 — 最低优先级

如果 CLAUDE.md 或 AGENTS.md 说 "不要使用 TDD"，而 skill 说 "始终使用 TDD"，用户指令优先。

**SUBAGENT-STOP 门**

在 using-superpowers 中添加了 `<SUBAGENT-STOP>` 块。为特定任务分派的 subagents 现在跳过 skill，而不是激活 1% 规则并调用完整的 skill 工作流。

**多平台改进**

- Codex 工具映射移至渐进式披露参考文件 (`references/codex-tools.md`)
- 添加了平台适配指针，以便非 Claude Code 平台可以找到工具等效项
- 计划标题现在称呼 "agentic workers" 而不是特指 "Claude"
- `docs/README.codex.md` 中记录了 Collab 功能要求

**Writing-plans 模板更新**

- 计划步骤现在使用复选框语法（`- [ ] **Step N:**`）进行进度追踪
- 计划标题引用了 subagent-driven-development 和 executing-plans 并带有平台感知路由

---

## v4.3.1 (2026-02-21)

### 新增

**Cursor 支持**

Superpowers 现在可与 Cursor 的插件系统配合使用。包含 `.cursor-plugin/plugin.json` 清单和 README 中的 Cursor 特定安装说明。SessionStart hook 输出现在包含 `additional_context` 字段以及现有的 `hookSpecificOutput.additionalContext`，以兼容 Cursor hook。

### 修复

**Windows: 恢复 polyglot wrapper 以实现可靠的 hook 执行 (#518, #504, #491, #487, #466, #440)**

Claude Code 在 Windows 上的 `.sh` 自动检测会在 hook 命令前预置 `bash`，破坏了执行。修复方式：

- 将 `session-start.sh` 重命名为 `session-start`（无扩展名），使自动检测不干扰
- 恢复了 `run-hook.cmd` polyglot wrapper，具有多位置 bash 发现（标准 Git for Windows 路径，然后 PATH 回退）
- 如果未找到 bash 则静默退出而不是报错
- 在 Unix 上，wrapper 通过 `exec bash` 直接运行脚本
- 使用 POSIX 安全的 `dirname "$0"` 路径解析（适用于 dash/sh，不仅仅是 bash）

这修复了 Windows 上路径包含空格、缺少 WSL、MSYS 上 `set -euo pipefail` 脆弱性以及反斜杠损坏导致的 SessionStart 失败。

## v4.3.0 (2026-02-12)

此修复应显著提高 superpowers skills 合规性，并减少 Claude 无意间进入其原生计划模式的可能性。

### 变更

**Brainstorming skill 现在强制执行其工作流而不是描述它**

模型跳过设计阶段直接进入 frontend-design 等实施 skills，或将整个 brainstorming 过程折叠为单个文本块。skill 现在使用硬门控、强制性清单和 Graphviz 流程来强制合规：

- `<HARD-GATE>`：在设计呈现并获得用户批准之前，不得使用实施 skills、编写代码或搭建脚手架
- 显式清单（6 项），必须作为任务创建并按顺序完成
- Graphviz 流程以 `writing-plans` 作为唯一有效终止状态
- 针对 "这太简单了不需要设计" 的反模式标注 — 正是模型用来跳过流程的合理化借口
- 设计部分大小基于部分复杂度，而非项目复杂度

**Using-superpowers 工作流图拦截 EnterPlanMode**

在 skill 流程图中添加了 `EnterPlanMode` 拦截器。当模型即将进入 Claude 的原生计划模式时，它会检查 brainstorming 是否已经发生，并路由到 brainstorming skill。计划模式永远不会被进入。

### 修复

**SessionStart hook 现在同步运行**

将 hooks.json 中的 `async: true` 更改为 `async: false`。当异步时，hook 可能在模型第一轮之前无法完成，意味着 using-superpowers 指令不在第一条消息的上下文中。

## v4.2.0 (2026-02-05)

### Breaking Changes

**Codex: 用原生 skill 发现替换 bootstrap CLI**

`superpowers-codex` bootstrap CLI、Windows `.cmd` wrapper 和相关的 bootstrap 内容文件已被移除。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生 skill 发现，因此旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只需 clone + symlink（在 INSTALL.md 中记录）。不需要 Node.js 依赖。旧的 `~/.codex/skills/` 路径已弃用。

### 修复

**Windows: 修复 Claude Code 2.1.x hook 执行 (#331)**

Claude Code 2.1.x 更改了 hooks 在 Windows 上的执行方式：它现在自动检测命令中的 `.sh` 文件并预置 `bash`。这破坏了 polyglot wrapper 模式，因为 `bash "run-hook.cmd" session-start.sh` 试图将 `.cmd` 文件作为 bash 脚本执行。

修复：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 自动处理 bash 调用。还添加了 .gitattributes 以强制 shell 脚本使用 LF 行尾（修复 Windows checkout 上的 CRLF 问题）。

**Windows: SessionStart hook 异步运行以防止终端冻结 (#404, #413, #414, #419)**

同步的 SessionStart hook 阻止 TUI 在 Windows 上进入 raw 模式，冻结所有键盘输入。异步运行 hook 可防止冻结，同时仍注入 superpowers 上下文。

**Windows: 修复 O(n^2) `escape_for_json` 性能**

使用 `${input:$i:1}` 的逐字符循环由于子字符串复制开销在 bash 中是 O(n^2) 的。在 Windows Git Bash 上这需要 60+ 秒。替换为 bash 参数替换（`${s//old/new}`），每个模式作为单次 C 级别传递运行 — 在 macOS 上快 7 倍，在 Windows 上显著更快。

**Codex: 修复 Windows/PowerShell 调用 (#285, #243)**

- Windows 不遵循 shebangs，因此直接调用无扩展名的 `superpowers-codex` 脚本会触发 "Open with" 对话框。所有调用现在都带有 `node` 前缀。
- 修复了 Windows 上的 `~/` 路径展开 — PowerShell 在将 `~` 作为参数传递给 `node` 时不展开。改为 `$HOME`，在 bash 和 PowerShell 中都能正确展开。

**Codex: 修复安装程序中的路径解析**

使用 `fileURLToPath()` 而不是手动 URL 路径名解析，以正确处理所有平台上包含空格和特殊字符的路径。

**Codex: 修复 writing-skills 中的过时 skills 路径**

将 `~/.codex/skills/` 引用（已弃用）更新为 `~/.agents/skills/` 以支持原生发现。

### 改进

**实施前现在需要 Worktree 隔离**

为 `subagent-driven-development` 和 `executing-plans` 都添加了 `using-git-worktrees` 作为必需 skill。实施工作流现在明确要求在开始工作之前设置隔离的 worktree，防止意外直接在 main 上工作。

**Main 分支保护软化为需要显式同意**

Skills 不再完全禁止 main 分支工作，而是允许在用户明确同意的情况下进行。更灵活的同时仍确保用户意识到其影响。

**简化安装验证**

从验证步骤中移除了 `/help` 命令检查和特定的 slash command 列表。Skills 主要通过描述你想做什么来调用，而不是通过运行特定命令。

**Codex: 阐明 bootstrap 中的 subagent 工具映射**

改进了 Codex 工具如何映射到 Claude Code 等效项用于 subagent 工作流的文档。

### 测试

- 添加了 subagent-driven-development 的 worktree 需求测试
- 添加了 main 分支红旗警告测试
- 修复了 skill 识别测试断言中的大小写敏感性问题
