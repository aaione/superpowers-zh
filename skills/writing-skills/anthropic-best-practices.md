# Skill 编写最佳实践（Skill authoring best practices）

> 学习如何编写有效的 Skill，让 agent 能发现并成功使用。

好的 Skill 简洁、结构良好，并经过真实使用测试。本指南提供实用的编写决策，帮助你写出 agent 能发现并有效使用的 Skill。

关于 Skill 工作原理的概念背景，请参见 [Skills 概览](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是一种公共资源。你的 Skill 与 agent 需要知道的一切共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他 Skill 的元数据
* 你的实际请求

你的 Skill 中并非每个 token 都会立即产生成本。在启动时，只有来自所有 Skill 的元数据（name 和 description）会被预加载。Agent 仅在 Skill 变得相关时才读取 SKILL.md，并按需读取额外文件。不过，在 SKILL.md 中保持简洁仍然重要：一旦 agent 加载了它，每个 token 都在与对话历史和其他上下文竞争。

**默认假设**：Agent 已经非常聪明

只添加 agent 还不知道的上下文。对每一条信息提出质疑：

* "Agent 真的需要这个解释吗？"
* "我能假设 agent 知道这个吗？"
* "这一段值得它的 token 成本吗？"

**好的示例：简洁**（约 50 token）：

````markdown  theme={null}
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏的示例：太冗长**（约 150 token）：

```markdown  theme={null}
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available for PDF processing, but we
recommend pdfplumber because it's easy to use and handles most cases well.
First, you'll need to install it using pip. Then you can use the code below...
```

简洁版本假设 agent 知道 PDF 是什么，以及库是如何工作的。

### 设定合适的自由度

把具体性水平与任务的脆弱性和可变性相匹配。

**高自由度**（基于文本的指令）：

适用场景：

* 多种方法都有效
* 决策取决于上下文
* 启发式方法指导方法

示例：

```markdown  theme={null}
## Code review process

1. Analyze the code structure and organization
2. Check for potential bugs or edge cases
3. Suggest improvements for readability and maintainability
4. Verify adherence to project conventions
```

**中等自由度**（带参数的伪代码或脚本）：

适用场景：

* 存在首选模式
* 某些变化是可接受的
* 配置影响行为

示例：

````markdown  theme={null}
## Generate report

Use this template and customize as needed:

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
    # Optionally include visualizations
```
````

**低自由度**（特定脚本，很少或没有参数）：

适用场景：

* 操作脆弱且易错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## Database migration

Run exactly this script:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

**类比**：把 agent 想象成一个在路径上探索的机器人：

* **两侧都是悬崖的窄桥**：只有一条安全的前进道路。提供具体的护栏和精确的指令（低自由度）。示例：必须按确切顺序运行的数据库迁移。
* **没有危险的旷野**：许多路径都能通向成功。给出大致方向，信任 agent 找到最佳路线（高自由度）。示例：由上下文决定最佳方法的 code review。

### 用所有你计划使用的模型测试

Skill 充当对模型的补充，因此有效性取决于底层模型。用所有计划使用的模型测试你的 Skill。

**按模型的测试考量**：

* **Claude Haiku**（快速、经济）：Skill 是否提供了足够的指导？
* **Claude Sonnet**（均衡）：Skill 是否清晰且高效？
* **Claude Opus**（强大推理）：Skill 是否避免过度解释？

对 Opus 完美适用的内容，对 Haiku 可能需要更多细节。如果你计划在多个模型上使用 Skill，目标是让指令在所有模型上都运作良好。

## Skill 结构

<Note>
  **YAML Frontmatter**：SKILL.md 的 frontmatter 需要两个字段：

  * `name` - Skill 的人类可读名称（最多 64 字符）
  * `description` - 关于 Skill 做什么以及何时使用的一行描述（最多 1024 字符）

  完整的 Skill 结构详情，见 [Skills 概览](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式，让 Skill 更容易引用和讨论。我们建议对 Skill 名称使用**动名词形式**（动词 + -ing），因为这清楚地描述了 Skill 提供的活动或能力。

**好的命名示例（动名词形式）**：

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代**：

* 名词短语："PDF Processing"、"Spreadsheet Analysis"
* 动作导向："Process PDFs"、"Analyze Spreadsheets"

**避免**：

* 含糊的名称："Helper"、"Utils"、"Tools"
* 过于通用的名称："Documents"、"Data"、"Files"
* 你的 skill 集合中不一致的模式

一致的命名让你更容易：

* 在文档和对话中引用 Skill
* 一眼理解一个 Skill 做什么
* 组织和搜索多个 Skill
* 维护一个专业、内聚的 skill 库

### 编写有效的 description

`description` 字段支持 Skill 发现，应同时包含 Skill 做什么以及何时使用。

<Warning>
  **始终用第三人称撰写**。description 会被注入到系统提示中，不一致的视角会导致发现问题。

  * **好：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**要具体并包含关键术语**。既包含 Skill 做什么，也包含何时使用它的具体触发器/上下文。

每个 Skill 恰好有一个 description 字段。description 对 skill 选择至关重要：agent 用它从可能 100 多个可用 Skill 中选择正确的 Skill。你的 description 必须提供足够的细节让 agent 知道何时选择此 Skill，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF Processing skill：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel Analysis skill：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免像下面这样含糊的 description：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 渐进式披露模式

SKILL.md 充当概览，按需把 agent 指向详细材料，就像入职指南中的目录。关于渐进式披露如何工作的说明，见概览中的 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指南：**

* 保持 SKILL.md 正文在 500 行以内以获得最佳性能
* 当接近此限制时把内容拆分到单独文件
* 使用下面的模式来有效地组织指令、代码和资源

#### 可视化概览：从简单到复杂

一个基本的 Skill 只从一个包含元数据和指令的 SKILL.md 文件开始：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着你的 Skill 增长，你可以捆绑 agent 仅在需要时才加载的额外内容：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的 Skill 目录结构可能像这样：

```
pdf/
├── SKILL.md              # Main instructions (loaded when triggered)
├── FORMS.md              # Form-filling guide (loaded as needed)
├── reference.md          # API reference (loaded as needed)
├── examples.md           # Usage examples (loaded as needed)
└── scripts/
    ├── analyze_form.py   # Utility script (executed, not loaded)
    ├── fill_form.py      # Form filling script
    └── validate.py       # Validation script
```

#### 模式 1：带参考的高级指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns
````

Agent 仅在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：按领域组织

对于有多个领域的 Skill，按领域组织内容以避免加载无关上下文。当用户询问销售指标时，agent 只需要阅读销售相关的 schema，而不是财务或营销数据。这保持低 token 用量并聚焦上下文。

```
bigquery-skill/
├── SKILL.md (overview and navigation)
└── reference/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    ├── product.md (API usage, features)
    └── marketing.md (campaigns, attribution)
```

````markdown SKILL.md theme={null}
# BigQuery Data Analysis

## Available datasets

**Finance**: Revenue, ARR, billing → See [reference/finance.md](reference/finance.md)
**Sales**: Opportunities, pipeline, accounts → See [reference/sales.md](reference/sales.md)
**Product**: API usage, features, adoption → See [reference/product.md](reference/product.md)
**Marketing**: Campaigns, attribution, email → See [reference/marketing.md](reference/marketing.md)

## Quick search

Find specific metrics using grep:

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件性细节

展示基本内容，链接到高级内容：

```markdown  theme={null}
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

Agent 仅在用户需要那些功能时才阅读 REDLINING.md 或 OOXML.md。

### 避免深度嵌套的引用

当文件从其他被引用的文件中再次被引用时，agent 可能只部分读取文件。遇到嵌套引用时，agent 可能使用 `head -100` 等命令预览内容，而不是阅读整个文件，导致信息不完整。

**保持引用从 SKILL.md 起只深一层**。所有引用文件都应直接从 SKILL.md 链接，以确保 agent 在需要时阅读完整文件。

**坏的示例：太深**：

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here's the actual information...
```

**好的示例：一层深**：

```markdown  theme={null}
# SKILL.md

**Basic usage**: [instructions in SKILL.md]
**Advanced features**: See [advanced.md](advanced.md)
**API reference**: See [reference.md](reference.md)
**Examples**: See [examples.md](examples.md)
```

### 用目录组织较长的参考文件

对于超过 100 行的参考文件，在顶部包含一个目录。这确保 agent 即使在部分预览读取时，也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API Reference

## Contents
- Authentication and setup
- Core methods (create, read, update, delete)
- Advanced features (batch operations, webhooks)
- Error handling patterns
- Code examples

## Authentication and setup
...

## Core methods
...
```

Agent 之后可以阅读完整文件，或按需跳到特定小节。

关于这种基于文件系统的架构如何支持渐进式披露的细节，见下面进阶小节中的 [Runtime environment](#runtime-environment) 小节。

## 工作流和反馈循环

### 为复杂任务使用工作流

把复杂操作拆分为清晰的顺序步骤。对于特别复杂的工作流，提供一个检查清单，agent 可以把它复制到自己的回复中，并随着推进逐项勾选。

**示例 1：研究综合工作流**（适用于无代码的 Skill）：

````markdown  theme={null}
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1: Read all source documents**

Review each document in the `sources/` directory. Note the main arguments and supporting evidence.

**Step 2: Identify key themes**

Look for patterns across sources. What themes appear repeatedly? Where do sources agree or disagree?

**Step 3: Cross-reference claims**

For each major claim, verify it appears in the source material. Note which source supports each point.

**Step 4: Create structured summary**

Organize findings by theme. Include:
- Main claim
- Supporting evidence from sources
- Conflicting viewpoints (if any)

**Step 5: Verify citations**

Check that every claim references the correct source document. If citations are incomplete, return to Step 3.
````

这个示例展示了工作流如何适用于不需要代码的分析任务。检查清单模式适用于任何复杂的多步骤流程。

**示例 2：PDF 表单填写工作流**（适用于带代码的 Skill）：

````markdown  theme={null}
## PDF form filling workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1: Analyze the form**

Run: `python scripts/analyze_form.py input.pdf`

This extracts form fields and their locations, saving to `fields.json`.

**Step 2: Create field mapping**

Edit `fields.json` to add values for each field.

**Step 3: Validate mapping**

Run: `python scripts/validate_fields.py fields.json`

Fix any validation errors before continuing.

**Step 4: Fill the form**

Run: `python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5: Verify output**

Run: `python scripts/verify_output.py output.pdf`

If verification fails, return to Step 2.
````

清晰的步骤防止 agent 跳过关键的校验。检查清单帮助你和你和 agent 追踪多步骤工作流的进度。

### 实现反馈循环

**常见模式**：运行校验器 → 修复错误 → 重复

这个模式极大提升输出质量。

**示例 1：风格指南合规**（适用于无代码的 Skill）：

```markdown  theme={null}
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

这展示了使用参考文档而非脚本的校验循环模式。"校验器"是 STYLE\_GUIDE.md，agent 通过阅读和比较来执行检查。

**示例 2：文档编辑流程**（适用于带代码的 Skill）：

```markdown  theme={null}
## Document editing process

1. Make your edits to `word/document.xml`
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails:
   - Review the error message carefully
   - Fix the issues in the XML
   - Run validation again
4. **Only proceed when validation passes**
5. Rebuild: `python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. Test the output document
```

校验循环及早捕获错误。

## 内容指南

### 避免时效性信息

不要包含会过时的信息：

**坏的示例：时效性**（会变错）：

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**好的示例**（使用"旧模式"小节）：

```markdown  theme={null}
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

旧模式小节提供历史背景而不让主内容变得杂乱。

### 使用一致的术语

选择一个术语并在整个 Skill 中使用：

**好 - 一致**：

* 始终用 "API endpoint"
* 始终用 "field"
* 始终用 "extract"

**坏 - 不一致**：

* 混用 "API endpoint"、"URL"、"API route"、"path"
* 混用 "field"、"box"、"element"、"control"
* 混用 "extract"、"pull"、"get"、"retrieve"

一致性帮助 agent 理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。把严格程度水平与你的需求相匹配。

**用于严格要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## Report structure

ALWAYS use this exact template structure:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

**用于灵活指导**（当改编有用时）：

````markdown  theme={null}
## Report structure

Here is a sensible default format, but use your best judgment based on the analysis:

```markdown
# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Tailor to the specific context]
```

Adjust sections as needed for the specific analysis type.
````

### 示例模式

对于输出质量取决于看到示例的 Skill，像在常规提示中一样提供输入/输出对：

````markdown  theme={null}
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output:
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**Example 3:**
Input: Updated dependencies and refactored error handling
Output:
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

Follow this style: type(scope): brief description, then detailed explanation.
````

示例帮助 agent 比仅凭描述更清楚地理解期望的风格和细节程度。

### 条件工作流模式

引导 agent 通过决策点：

```markdown  theme={null}
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

<Tip>
  如果工作流变得庞大或复杂、步骤很多，考虑把它们推到单独的文件中，并告诉 agent 根据手头任务阅读合适的文件。
</Tip>

## 评估与迭代

### 先构建评估

**在编写大量文档之前创建评估。** 这确保你的 Skill 解决的是真实问题，而不是为想象的问题写文档。

**评估驱动的开发：**

1. **识别空白**：在没有 Skill 的情况下，在代表性任务上运行你的 agent。记录具体的失败或缺失的上下文
2. **创建评估**：构建三个测试这些空白的场景
3. **建立基线**：在没有 Skill 的情况下衡量 agent 的表现
4. **编写最少的指令**：创建刚好够填补空白并通过评估的内容
5. **迭代**：执行评估，与基线比较，并改进

这种方法确保你在解决实际问题，而不是在预测可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  这个示例展示了一个带简单测试评分标准的数据驱动评估。我们目前不提供运行这些评估的内置方式。用户可以创建自己的评估系统。评估是你衡量 Skill 有效性的真理来源。
</Note>

### 与 agent 迭代开发 Skills

最有效的 Skill 开发流程涉及 agent 自身。与一个实例（"Agent A"）一起创建将被其他实例（"Agent B"）使用的 Skill。Agent A 帮助你设计和改进指令，而 Agent B 在真实任务中测试它们。这之所以有效，是因为底层模型既理解如何编写有效的 agent 指令，也理解 agent 需要什么信息。

**创建新 Skill：**

1. **在没有 Skill 的情况下完成任务**：用普通提示与 Agent A 一起解决一个问题。随着你推进，你会自然地提供上下文、解释偏好、分享流程知识。注意你反复提供哪些信息。

2. **识别可复用模式**：完成任务后，识别你提供的哪些上下文对类似的未来任务有用。

   **示例**：如果你完成了一次 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"），以及常见查询模式。

3. **请 Agent A 创建一个 Skill**："创建一个 Skill，捕获我们刚刚使用的这个 BigQuery 分析模式。包含表 schema、命名约定，以及过滤测试账户的规则。"

   <Tip>
     现代 agent 原生理解 Skill 格式和结构。你不需要特殊的系统提示或"writing skills" skill 来获得创建 Skill 的帮助。只需请 agent 创建一个 Skill，它就会生成结构恰当、带合适 frontmatter 和正文的 SKILL.md 内容。
   </Tip>

4. **审查简洁性**：检查 Agent A 是否添加了不必要的解释。例如问："删掉关于 win rate 含义的解释——agent 已经知道那个了。"

5. **改进信息架构**：请 Agent A 更有效地组织内容。例如："把这个重新组织，让表 schema 在单独的参考文件中。我们以后可能会加更多表。"

6. **在相似任务上测试**：把 Skill 用于 Agent B（一个加载了该 Skill 的新实例），在相关用例上使用。观察 Agent B 是否找到正确信息、正确应用规则，并成功完成任务。

7. **基于观察迭代**：如果 Agent B 挣扎或遗漏了什么，带着具体细节回到 Agent A："当 agent 使用这个 Skill 时，它忘了为 Q4 按日期过滤。我们是不是该加一节关于日期过滤模式的内容？"

**迭代现有 Skills：**

在改进 Skills 时，同样的层级模式继续。你在以下之间交替：

* **与 Agent A 合作**（帮助改进 Skill 的专家）
* **用 Agent B 测试**（使用 Skill 执行真实工作的 agent）
* **观察 Agent B 的行为**并把洞见带回给 Agent A

1. **在真实工作流中使用 Skill**：给 Agent B（加载了 Skill）真实任务，而不是测试场景

2. **观察 Agent B 的行为**：注意它在哪里挣扎、成功或做出意外选择

   **示例观察**："当我向 Agent B 要一份区域销售报告时，它写了查询但忘了过滤掉测试账户，尽管 Skill 提到了这条规则。"

3. **回到 Agent A 寻求改进**：分享当前的 SKILL.md 并描述你观察到的。问："我注意到当我要求区域报告时，Agent B 忘了过滤测试账户。Skill 提到了过滤，但也许它不够突出？"

4. **审查 Agent A 的建议**：Agent A 可能建议重新组织让规则更突出，使用更强语言如"MUST filter"代替"always filter"，或重构工作流小节。

5. **应用并测试改动**：用 Agent A 的改进更新 Skill，然后用 Agent B 在相似请求上再次测试

6. **基于使用情况重复**：随着你遇到新场景，继续这个观察-改进-测试循环。每次迭代都基于真实 agent 行为而非假设改进 Skill。

**收集团队反馈：**

1. 与队友分享 Skills 并观察他们的使用
2. 问：Skill 是否如期激活？指令是否清晰？缺什么？
3. 整合反馈以解决你自己使用模式中的盲点

**这种方法为何有效**：Agent A 理解 agent 需求，你提供领域专业知识，Agent B 通过真实使用揭示空白，而迭代改进基于观察到的行为而非假设来改进 Skills。

### 观察 agent 如何导航 Skills

在你迭代 Skills 时，注意 agent 在实践中如何实际使用它们。注意：

* **意外的探索路径**：agent 是否以你没预期的顺序读取文件？这可能表明你的结构不像你以为的那样直观
* **遗漏的连接**：agent 是否未能跟随对重要文件的引用？你的链接可能需要更明确或更突出
* **过度依赖某些小节**：如果 agent 反复阅读同一文件，考虑那些内容是否应放在主 SKILL.md 中
* **被忽略的内容**：如果 agent 从不访问某个捆绑文件，它可能是不必要的，或在主指令中信号不足

基于这些观察而非假设来迭代。你 Skill 元数据中的 'name' 和 'description' 尤其关键。agent 在决定是否响应当前任务触发 Skill 时使用它们。确保它们清楚描述 Skill 做什么以及何时应被使用。

## 要避免的反模式

### 避免 Windows 风格路径

始终在文件路径中使用正斜杠，即使在 Windows 上：

* ✓ **好**：`scripts/helper.py`、`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

Unix 风格路径跨所有平台工作，而 Windows 风格路径在 Unix 系统上会导致错误。

### 避免提供太多选项

除非必要，不要呈现多种方法：

````markdown  theme={null}
**Bad example: Too many choices** (confusing):
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**Good example: Provide a default** (with escape hatch):
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 进阶：带可执行代码的 Skills

下面的小节聚焦于包含可执行脚本的 Skills。如果你的 Skill 只使用 markdown 指令，跳到 [有效 Skills 的检查清单](#checklist-for-effective-skills)。

### 解决，而不是推诿

在为 Skills 编写脚本时，处理错误条件，而不是推给 agent。

**好的示例：显式处理错误**：

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**坏的示例：推诿给 agent**：

```python  theme={null}
def process_file(path):
    # Just fail and let the agent figure it out
    return open(path).read()
```

配置参数也应有理有据并文档化，以避免"魔法常数"（Ousterhout 定律）。如果你不知道正确的值，agent 又怎么确定？

**好的示例：自文档化**：

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**坏的示例：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### 提供实用脚本

即使你的 agent 能写脚本，预制的脚本也有优势：

**实用脚本的好处**：

* 比生成的代码更可靠
* 节省 token（无需在上下文中包含代码）
* 节省时间（无需生成代码）
* 确保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e040774f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上图展示了可执行脚本如何与指令文件协同工作。指令文件（forms.md）引用脚本，agent 可以在不把脚本内容加载到上下文中的情况下执行它。

**重要区分**：在你的指令中明确 agent 应该：

* **执行脚本**（最常见）："Run `analyze_form.py` to extract fields"
* **作为参考阅读**（用于复杂逻辑）："See `analyze_form.py` for the field extraction algorithm"

对于大多数实用脚本，执行是首选，因为它更可靠且高效。关于脚本执行如何工作的细节，见下面的 [Runtime environment](#runtime-environment) 小节。

**示例**：

````markdown  theme={null}
## Utility scripts

**analyze_form.py**: Extract all form fields from PDF

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

Output format:
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**: Check for overlapping bounding boxes

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**: Apply field values to PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 agent 分析它们：

````markdown  theme={null}
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. The agent can see field locations and types visually
````

<Note>
  在这个示例中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

Agent 视觉能力帮助理解布局和结构。

### 创建可验证的中间产物

当 agent 执行复杂的、开放式任务时，它们可能犯错。"规划-校验-执行"模式通过让 agent 先以结构化格式创建计划，再用脚本校验该计划，然后执行，从而及早捕获错误。

**示例**：想象让 agent 根据电子表格更新 PDF 中的 50 个表单字段。没有校验，它可能引用不存在的字段、创建冲突的值、遗漏必需字段，或错误地应用更新。

**解决方案**：使用上面展示的工作流模式（PDF 表单填写），但添加一个中间的 `changes.json` 文件，在应用改动之前校验它。工作流变成：分析 → **创建计划文件** → **校验计划** → 执行 → 验证。

**此模式为何有效：**

* **及早捕获错误**：校验在改动应用之前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆的规划**：agent 可以在不触碰原件的情况下迭代计划
* **清晰的调试**：错误信息指向具体问题

**何时使用**：批量操作、破坏性改动、复杂校验规则、高风险操作。

**实现提示**：让校验脚本详尽，带具体错误信息，如"Field 'signature\_date' not found. Available fields: customer\_name, order\_total, signature\_date\_signed"，以帮助 agent 修复问题。

### 包依赖

Skills 在代码执行环境中运行，有平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，并从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问，没有运行时包安装

在你的 SKILL.md 中列出所需包，并验证它们在[代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)中可用。

### Runtime environment

Skills 运行在具有文件系统访问、bash 命令和代码执行能力的代码执行环境中。关于此架构的概念解释，见概览中的 [The Skills architecture](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这如何影响你的编写：**

**Agent 如何访问 Skills：**

1. **元数据预加载**：启动时，所有 Skill 的 YAML frontmatter 中的 name 和 description 被加载到系统提示中
2. **按需读取文件**：Agent 在需要时使用文件读取工具从文件系统访问 SKILL.md 和其他文件
3. **高效执行脚本**：实用脚本可以通过 bash 执行，而无需把完整内容加载到上下文。只有脚本的输出消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在实际被读取之前不消耗上下文 token

* **文件路径很重要**：Agent 像导航文件系统一样导航你的 skill 目录。使用正斜杠（`reference/guide.md`），不要用反斜杠
* **描述性地命名文件**：使用能指示内容的名称：`form_validation_rules.md`，而不是 `doc2.md`
* **为发现而组织**：按领域或特性构建目录结构
  * 好：`reference/finance.md`、`reference/sales.md`
  * 坏：`docs/file1.md`、`docs/file2.md`
* **捆绑全面的资源**：包含完整 API 文档、大量示例、大型数据集；在被访问前没有上下文惩罚
* **对确定性操作优先使用脚本**：编写 `validate_form.py`，而不是让 agent 生成校验代码
* **让执行意图清晰**：
  * "Run `analyze_form.py` to extract fields"（执行）
  * "See `analyze_form.py` for the extraction algorithm"（作为参考阅读）
* **测试文件访问模式**：通过用真实请求测试来验证 agent 能导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md (overview, points to reference files)
└── reference/
    ├── finance.md (revenue metrics)
    ├── sales.md (pipeline data)
    └── product.md (usage analytics)
```

当用户询问收入时，agent 阅读 SKILL.md，看到对 `reference/finance.md` 的引用，并调用 bash 仅读取那个文件。sales.md 和 product.md 文件保留在文件系统中，在被需要之前消耗零上下文 token。这种基于文件系统的模型正是渐进式披露的基础。Agent 可以导航并选择性地加载每个任务确切需要的内容。

关于技术架构的完整细节，见 Skills 概览中的 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的 Skill 使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名，以避免"tool not found"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名
* `bigquery_schema` 和 `create_issue` 是那些服务器中的工具名

没有服务器前缀，agent 可能无法定位工具，尤其是当有多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包是可用的：

````markdown  theme={null}
**Bad example: Assumes installation**:
"Use the pdf library to process the file."

**Good example: Explicit about dependencies**:
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最多 64 字符）和 `description`（最多 1024 字符）字段。完整的结构详情见 [Skills 概览](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

保持 SKILL.md 正文在 500 行以内以获得最佳性能。如果你的内容超过此限制，使用前面描述的渐进式披露模式拆分到单独文件。关于架构细节，见 [Skills 概览](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效 Skills 的检查清单

在分享 Skill 之前，验证：

### 核心质量

* [ ] description 具体并包含关键术语
* [ ] description 同时包含 Skill 做什么以及何时使用
* [ ] SKILL.md 正文在 500 行以内
* [ ] 额外细节在单独文件中（如需要）
* [ ] 没有时效性信息（或在"旧模式"小节中）
* [ ] 全文术语一致
* [ ] 示例具体，不抽象
* [ ] 文件引用只深一层
* [ ] 恰当地使用渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而不是推诿给 agent
* [ ] 错误处理显式且有帮助
* [ ] 没有"魔法常数"（所有值有理有据）
* [ ] 所需包在指令中列出并验证为可用
* [ ] 脚本有清晰的文档
* [ ] 没有 Windows 风格路径（全部用正斜杠）
* [ ] 关键操作有校验/验证步骤
* [ ] 为质量关键任务包含反馈循环

### 测试

* [ ] 至少创建三个评估
* [ ] 用 Haiku、Sonnet 和 Opus 测试过
* [ ] 用真实使用场景测试过
* [ ] 整合了团队反馈（如适用）

## 后续步骤

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart">
    Create your first Skill
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="https://code.claude.com/docs/en/skills">
    Create and manage Skills in Claude Code
  </Card>

  <Card title="Use Skills with the API" icon="code" href="https://platform.claude.com/docs/en/build-with-claude/skills-guide">
    Upload and use Skills programmatically
  </Card>
</CardGroup>
