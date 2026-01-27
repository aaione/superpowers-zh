# Implementer Subagent Prompt Template

当 dispatch 一个 implementer subagent 时使用此模板。

```
Task tool (general-purpose):
  description: "Implement Task N: [task name]"
  prompt: |
    你正在实施 Task N: [task name]

    ## 任务描述 (Task Description)

    [FULL TEXT of task from plan - paste it here, don't make subagent read file]

    ## 上下文 (Context)

    [场景设置: 它的位置, 依赖关系, 架构上下文]

    ## 开始之前 (Before You Begin)

    如果你有关于以下方面的问题:
    - 需求或验收标准
    - 方法或实施策略
    - 依赖关系或假设
    - 任务描述中任何不清楚的地方

    **现在问。** 在开始工作之前提出任何疑虑。

    ## 你的工作 (Your Job)

    一旦你明确了需求:
    1. 准确实施任务指定的内容
    2. 编写测试 (如果任务要求，遵循 TDD)
    3. 验证实施有效
    4. Commit 你的工作
    5. 自我审查 (见下文)
    6. 报告

    Work from: [directory]

    **当你工作时:** 如果你遇到意想不到或不清楚的事情，**提问**。
    暂停并澄清总是可以的。不要猜测或做出假设。

    ## 报告之前: 自我审查 (Self-Review)

    用全新的眼光审查你的工作。问你自己:

    **完整性 (Completeness):**
    - 我是否完全实施了 spec 中的所有内容？
    - 我是否遗漏了任何需求？
    - 是否有我没处理的边界情况？

    **质量 (Quality):**
    - 这是我最好的工作吗？
    - 命名是否清晰准确（匹配做了什么，而不是怎么做）？
    - 代码是否整洁且可维护？

    **纪律 (Discipline):**
    - 我是否避免了过度构建 (YAGNI)？
    - 我是否只构建了请求的内容？
    - 我是否遵循了 codebase 中的现有模式？

    **测试 (Testing):**
    - 测试实际上验证了行为吗（不仅仅是模拟行为）？
    - 如果需要，我是否遵循了 TDD？
    - 测试是否全面？

    如果你在自我审查期间发现问题，请在报告之前立即修复它们。

    ## 报告格式 (Report Format)

    完成后，报告:
    - 你实施了什么
    - 你测试了什么和测试结果
    - 更改的文件
    - 自我审查发现 (如果有)
    - 任何问题或疑虑
```
