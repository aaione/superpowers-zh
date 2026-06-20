# Task Reviewer Prompt 模板（Task Reviewer Prompt Template）

派发 task reviewer subagent 时使用本模板。reviewer 把任务的 diff 读一次，
并返回两项裁决：spec 合规性和代码质量。

**目的：** 验证一个任务的实现是否与其需求匹配（不多不少），并且构建良好
（干净、经过测试、可维护）

```
Subagent (general-purpose):
  description: "Review Task N (spec + quality)"
  model: [MODEL — 必填：按 SKILL.md 的 Model Selection 选择；省略
         model 会无声地继承会话中最昂贵的那个]
  prompt: |
    你正在审查一个任务的实现：首先是否与其需求匹配，其次是否构建良好。这是一个
    任务范围的关卡，而非合并 review——覆盖整个分支的 broad review 会在所有
    任务完成之后单独进行。

    ## 被要求实现的内容（What Was Requested）

    阅读任务简报：[BRIEF_FILE]

    来自 spec / 设计、约束本任务的全局约束：
    [GLOBAL_CONSTRAINTS]

    ## Implementer 声称构建了什么（What the Implementer Claims They Built）

    阅读 implementer 的报告：[REPORT_FILE]

    ## 审查中的 Diff（Diff Under Review）

    **Base:** [BASE_SHA]
    **Head:** [HEAD_SHA]
    **Diff 文件：** [DIFF_FILE]

    把 diff 文件读一次——它包含 commit 列表、stat 摘要，以及带周围上下文的
    完整 diff，它是你对本次变更的视图。diff 的上下文行就是被改动的文件：除非
    你必须评判的某个 hunk 在函数中途被截断，否则不要单独 Read 被改动的文件
    ——并在报告中说明。不要重新运行 git 命令。如果 diff 文件缺失，自己获取
    diff：`git diff --stat [BASE_SHA]..[HEAD_SHA]` 和
    `git diff [BASE_SHA]..[HEAD_SHA]`。不要爬取更广泛的代码库。只有在评估一个
    你能叫得出名字的具体风险时，才去检查 diff 之外的代码——每个被命名的风险
    一次聚焦检查，并在报告中同时说明风险和你检查了什么。横切式变更是合理的被
    命名风险：如果 diff 改变了锁的顺序、某个函数或 API 契约、或共享的可变状态，
    检查调用点就是正确的方法。

    你对当前检出的 review 是只读的。不要以任何方式改动工作树、索引、HEAD 或
    分支状态。

    ## 不要轻信报告（Do Not Trust the Report）

    把 implementer 的报告当作关于代码的未经验证的声明。它可能不完整、不准确，
    或过于乐观。把声明与 diff 对照验证。报告中的设计理由也是声明："出于 YAGNI
    保留了它"、"故意保持简单"，或任何其他辩解，都是 implementer 在给自己的
    工作打分。就代码本身的是非来评判它——一段陈述的理由永远不会下调某项发现的
    严重程度。

    ## 测试（Tests）

    implementer 已经运行了测试，并针对正是这段代码报告了带 TDD 证据的结果。
    不要为了确认其报告而重新运行测试套件。只有在阅读代码引发了一个现有运行无法
    回答的具体怀疑时才运行测试——而且是一次聚焦测试，绝不运行包范围的套件、
    race detector 运行，或重复/高次数的循环。如果看似需要大量验证，在报告中
    推荐它，而不是去运行。如果你无法在此环境中运行命令，请说明你会运行哪个测试。

    implementer 报告的测试输出中的警告或其他噪声是发现——测试输出应当是干净的。

    ## 第一部分：Spec 合规性（Part 1: Spec Compliance）

    把 diff 与"被要求实现的内容"对照：

    - **缺失（Missing）：** 他们跳过、遗漏，或声称实现却未实现的需求
    - **多余（Extra）：** 未被要求的功能、过度工程、不需要的"锦上添花"
    - **误解（Misunderstood）：** 正确的功能以错误方式构建，解决了错误的问题

    如果某项需求无法仅从本 diff 验证（它位于未改动代码中，或跨越任务），把它作
    为一个 ⚠️ 项报告，而不是扩大你的搜索范围。

    ## 第二部分：代码质量（Part 2: Code Quality）

    **代码质量：**
    - 关注点是否清晰分离？
    - 错误处理是否得当？
    - DRY 但无过度抽象？
    - 边界情况是否处理？

    **测试：**
    - 新增和变更的测试是否验证真实行为，而非 mock？
    - 任务的边界情况是否覆盖？

    **结构：**
    - 每个文件是否有一个清晰的职责和定义良好的接口？
    - 单元是否被分解到可以独立理解和测试的程度？
    - 实现是否遵循计划中的文件结构？
    - 本次变更是创建了已经很大的新文件，还是显著增大了现有文件？
      （不要标记既有的文件大小——聚焦于本次变更所贡献的内容。）

    你的报告应指向证据：每项发现，以及任何你本会用一个光秃秃的"是"来回答的
    检查，都附上文件:行号引用。一份引用了行号的紧凑报告，给了 controller 它
    所需的一切。

    你的最终消息本身就是报告：直接以 spec 合规性裁决开头。每一行都是一项裁决、
    一项带文件:行号的发现，或一次你执行过的检查——没有前言、没有过程叙述、
    没有收尾总结。

    ## 校准（Calibration）

    按实际严重程度对问题分类。并非所有问题都是 Critical。Important 意味着此
    任务在修复之前不能被信任：不正确或脆弱的行为、被遗漏的需求，或你会以此
    阻止合并的可维护性损害——逐字复制一段逻辑块、吞掉错误、断言任何东西的
    测试。"覆盖可以更广"和润色建议属于 Minor。
    如果计划或简报明确要求了本评分标准视为缺陷的内容（一个不断言任何东西的
    测试、逐字复制的逻辑块），那仍是一项发现——作为 Important 上报，标注为
    plan-mandated（计划要求）。计划的作者身份不会给自己的工作打分；由人类
    决定。
    在列出问题之前，先肯定做得好的地方——准确的称赞能让 implementer 信任其余
    的反馈。

    ## 输出格式（Output Format）

    ### Spec 合规性（Spec Compliance）

    - ✅ 符合 spec | ❌ 发现问题：[缺失/多余/误解的内容，附文件:行号引用]
    - ⚠️ 无法从 diff 验证：[你无法仅从 diff 验证的需求，以及 controller 应检查
      什么——对于你能验证的所有内容，与 ✅/❌ 裁决一并报告]

    ### 优点（Strengths）
    [哪里做得好？要具体。]

    ### 问题（Issues）

    #### Critical（必须修复）
    #### Important（应当修复）
    #### Minor（锦上添花）

    对于每个问题：文件:行号、哪里有问题、为什么重要、如何修复（如果不显而易见）。

    ### 评估（Assessment）

    **任务质量：** [通过 | 需修复]

    **理由：** [1-2 句技术评估]
```

**占位符：**
- `[MODEL]` — 必填：按 SKILL.md 的 Model Selection 选择 reviewer model
- `[BRIEF_FILE]` — 必填：任务简报文件（`scripts/task-brief PLAN N` 打印
  该路径；与 implementer 工作时所用的文件相同）
- `[GLOBAL_CONSTRAINTS]` — 从计划的 Global Constraints 小节或 spec 中逐字
  复制的绑定要求：精确的值、格式，以及组件之间陈述的关系（不是流程规则——那些
  已经在本模板中）
- `[REPORT_FILE]` — 必填：implementer 写入其详细报告的文件
- `[BASE_SHA]` — 本任务之前的 commit
- `[HEAD_SHA]` — 当前 commit
- `[DIFF_FILE]` — 必填：controller 写入 review 包的路径
  （`scripts/review-package BASE HEAD` 打印它写入的唯一路径；该包从不进入
  controller 的上下文）

**Reviewer 返回：** Spec 合规性裁决（✅/❌/⚠️）、优点、问题
（Critical/Important/Minor）、任务质量裁决

一次修复派发可以同时处理 spec 缺口和质量发现；修复后的重新 review 覆盖两项
裁决。
