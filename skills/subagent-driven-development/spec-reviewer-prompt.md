# Spec Compliance Reviewer Prompt Template

当 dispatch 一个 spec compliance reviewer subagent 时使用此模板。

**目的:** 验证 implementer 构建了请求的内容（没有更多，也没有更少）

```
Task tool (general-purpose):
  description: "Review spec compliance for Task N"
  prompt: |
    你正在审查实施是否符合其规范。

    ## 要求的如内容 (What Was Requested)

    [FULL TEXT of task requirements]

    ## Implementer 声称构建的内容 (What Implementer Claims They Built)

    [From implementer's report]

    ## 关键: 不要相信报告 (CRITICAL: Do Not Trust the Report)

    Implementer 完成得可疑地快。他们的报告可能是不完整的、不准确的或乐观的。你必须独立验证一切。

    **不要 (DO NOT):**
    - 听信他们实施的内容
    - 相信他们关于完整性的说法
    - 接受他们对需求的解释

    **做 (DO):**
    - 阅读他们编写的实际代码
    - 逐行将实际实施与需求进行比较
    - 检查他们声称实施了但遗漏的部分
    - 寻找他们没有提到的额外功能

    ## 你的工作 (Your Job)

    阅读实施代码并验证:

    **遗漏的需求 (Missing requirements):**
    - 他们是否实施了请求的所有内容？
    - 是否有他们跳过或遗漏的需求？
    - 他们是否声称某样东西有效但实际上并未实施？

    **额外/不需要的工作 (Extra/unneeded work):**
    - 他们是否构建了未请求的东西？
    - 他们是否过度设计或添加了不必要的功能？
    - 他们是否添加了 spec 中没有的 "nice to haves"？

    **误解 (Misunderstandings):**
    - 他们对需求的解释是否与预期不同？
    - 他们是否解决了错误的问题？
    - 他们是否实施了正确的功能但方式错误？

    **通过阅读代码来验证，而不是通过信任报告。**

    Report:
    - ✅ Spec compliant (如果在代码检查后一切匹配)
    - ❌ Issues found: [具体列出遗漏或额外的内容，带有文件:行引用]
```
