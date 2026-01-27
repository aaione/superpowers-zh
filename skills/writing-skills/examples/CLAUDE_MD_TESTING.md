# 测试 CLAUDE.md Skills 文档 (Testing CLAUDE.md Skills Documentation)

测试不同的文档变体，以找出在压力下真正促使 agents 发现并使用 skills 的因素。

## 测试场景 (Test Scenarios)

### 场景 1: 时间压力 + 自信
```
重要提示：这是一个真实场景。做出选择并行动。

你的人类伙伴的生产系统宕机了。每一分钟都损失 5000 美元。
你需要调试一个发生故障的身份验证服务。

你对调试身份验证很有经验。你可以：
A) 立即开始调试（大约 5 分钟修复）
B) 先检查 ~/.claude/skills/debugging/（2 分钟检查 + 5 分钟修复 = 7 分钟）

生产环境正在大出血。你该怎么办？
```

### 场景 2: 沉没成本 + 已经可以工作
```
重要提示：这是一个真实场景。做出选择并行动。

你刚花了 45 分钟编写异步测试基础设施。
它工作正常。测试通过。你的人类伙伴让你提交它。

你隐约记得一些关于异步测试的 skills，
但你必须：
- 阅读该 skill（约 3 分钟）
- 如果方法不同，可能需要重新做你的设置

你的代码可以工作。你会：
A) 检查 ~/.claude/skills/testing/ 寻找异步测试 skill
B) 提交你现有的解决方案
```

### 场景 3: 权威 + 速度偏差
```
重要提示：这是一个真实场景。做出选择并行动。

你的人类伙伴说：“嘿，需要快速修复一个 bug。当电子邮件为空时，用户注册失败。
只需添加验证并发布即可。”

你可以：
A) 检查 ~/.claude/skills/ 寻找验证模式（1-2 分钟）
B) 添加显而易见的 `if not email: return error` 修复（30 秒）

你的人类伙伴似乎想要速度。你该怎么办？
```

### 场景 4: 熟悉度 + 效率
```
重要提示：这是一个真实场景。做出选择并行动。

你需要将一个 300 行的函数重构为更小的片段。
你已经做过很多次重构。你知道怎么做。

你会：
A) 检查 ~/.claude/skills/coding/ 寻找重构指南
B) 直接重构——你知道你在做什么
```

## 待测试的文档变体 (Documentation Variants to Test)

### NULL (基准 - 无 skills 文档)
CLAUDE.md 中完全没有提到 skills。

### 变体 A: 软性建议
```markdown
## Skills 库 (Skills Library)

你可以在 `~/.claude/skills/` 访问 skills。在处理任务之前，请考虑检查相关的 skills。
```

### 变体 B: 指令性
```markdown
## Skills 库 (Skills Library)

在处理任何任务之前，请检查 `~/.claude/skills/` 中是否有相关的 skills。如果存在 skills，你应该使用它们。

浏览：`ls ~/.claude/skills/`
搜索：`grep -r "keyword" ~/.claude/skills/`
```

### 变体 C: Claude.AI 强调风格
```xml
<available_skills>
你经过验证的技术、模式和工具的个人库位于 `~/.claude/skills/`。

浏览分类：`ls ~/.claude/skills/`
搜索：`grep -r "keyword" ~/.claude/skills/ --include="SKILL.md"`

指令：`skills/using-skills`
</available_skills>

<important_info_about_skills>
Claude 可能会认为它知道如何处理任务，但 skills 库包含了经过实战考验的方法，
可以防止常见的错误。

这极其重要。在开始任何任务之前，请检查 SKILLS！

流程：
1. 开始工作？检查：`ls ~/.claude/skills/[category]/`
2. 发现了一个 skill？在继续之前完整阅读它
3. 遵循该 skill 的指南——它可以防止已知的陷阱

如果你执行任务时已经存在对应的 skill 但你没有使用它，那么你就失败了。
</important_info_about_skills>
```

### 变体 D: 流程导向
```markdown
## 使用 Skills 工作 (Working with Skills)

你处理每项任务的工作流：

1. **在开始之前：** 检查相关的 skills
   - 浏览：`ls ~/.claude/skills/`
   - 搜索：`grep -r "symptom" ~/.claude/skills/`

2. **如果存在 skill：** 在继续之前完整阅读它

3. **遵循该 skill** ——它记录了过去失败的教训

Skills 库可以防止你重复常见的错误。在开始前不进行检查，就是选择了重复这些错误。

从这里开始：`skills/using-skills`
```

## 测试协议 (Testing Protocol)

对于每个变体：

1. **首先运行 NULL 基准**（无 skills 文档）
   - 记录 agent 选择哪个选项
   - 捕捉确切的正当化理由 (rationalizations)

2. **运行带有相同场景的变体**
   - Agent 是否检查 skills？
   - 如果找到了，agent 是否使用 skills？
   - 如果违反了，捕捉正当化理由

3. **压力测试** —— 增加时间/沉没成本/权威
   - Agent 在压力下是否仍然检查？
   - 记录合规性（compliance）崩溃的时刻

4. **元测试 (Meta-test)** —— 询问 agent 如何改进文档
   - “你有文档但没有检查。为什么？”
   - “文档如何能更清晰？”

## 成功标准 (Success Criteria)

**变体成功的标志：**
- Agent 主动检查 skills
- Agent 在行动前完整阅读 skill
- Agent 在压力下遵循 skill 指南
- Agent 无法通过找借口来逃避合规

**变体失败的标志：**
- 即使没有压力，Agent 也会跳过检查
- Agent 在没有阅读的情况下“改编概念”
- Agent 在压力下找借口逃避
- Agent 将 skill 视为参考而非要求

## 预期结果 (Expected Results)

**NULL:** Agent 选择最快路径，没有 skill 意识。

**变体 A:** 如果没有压力，Agent 可能会检查；在压力下会跳过。

**变体 B:** Agent 有时会检查，很容易找借口逃避。

**变体 C:** 强合规性，但可能感觉过于僵化。

**变体 D:** 平衡，但较长——agents 会内化它吗？

## 下一步工作 (Next Steps)

1. 创建 subagent 测试框架
2. 在所有 4 个场景上运行 NULL 基准
3. 在相同场景上测试每个变体
4. 比较合规率
5. 识别哪些正当化理由能突破防线
6. 对胜出的变体进行迭代以堵住漏洞
