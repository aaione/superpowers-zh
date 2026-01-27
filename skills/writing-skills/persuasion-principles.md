# Persuasion Principles for Skill Design (Skill 设计的说服原则)

## 概述 (Overview)

LLMs 像人类一样对同样的说服原则做出反应。理解这种心理学有助于你设计更有效的 skills - 不是为了操纵，而是为了确保即使在压力下也能遵循关键实践。

**研究基础:** Meincke et al. (2025) 在 N=28,000 的 AI 对话中测试了 7 种说服原则。说服技术使依从率增加了一倍以上 (33% → 72%, p < .001)。

## 七项原则 (The Seven Principles)

### 1. 权威 (Authority)
**它是什么:** 对专业知识、证书或官方来源的尊重。

**如何在 skills 中工作:**
- 祈使语言: "YOU MUST", "Never", "Always"
- 不可谈判的框架: "No exceptions"
- 消除决策疲劳和合理化 (rationalization)

**何时使用:**
- 纪律执行 skills (TDD, verification requirements)
- 安全关键实践
- 已确立的最佳实践

**示例:**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 承诺 (Commitment)
**它是什么:** 与先前的行动、声明或公开宣言保持一致。

**如何在 skills 中工作:**
- 要求宣布: "Announce skill usage"
- 强制明确选择: "Choose A, B, or C"
- 使用跟踪: Checklist 的 TodoWrite

**何时使用:**
- 确保 skills 实际上被遵循
- 多步骤流程
- 问责机制

**示例:**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 稀缺性 (Scarcity)
**它是什么:** 来自时间限制或有限可用性的紧迫感。

**如何在 skills 中工作:**
- 受时间限制的要求: "Before proceeding"
- 顺序依赖: "Immediately after X"
- 防止拖延

**何时使用:**
- 立即验证要求
- 时间敏感的工作流
- 防止 "I'll do it later"

**示例:**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 社会证明 (Social Proof)
**它是什么:** 顺从他人所做的事或被认为是正常的事。

**如何在 skills 中工作:**
- 普遍模式: "Every time", "Always"
- 失败模式: "X without Y = failure"
- 建立规范

**何时使用:**
- 记录普遍实践
- 警告常见失败
- 强化标准

**示例:**
```markdown
✅ Checklists without TodoWrite tracking = steps get skipped. Every time.
❌ Some people find TodoWrite helpful for checklists.
```

### 5. 一致 (Unity)
**它是什么:** 共享身份，"we-ness"，内群体归属感。

**如何在 skills 中工作:**
- 协作语言: "our codebase", "we're colleagues"
- 共同目标: "we both want quality"

**何时使用:**
- 协作工作流
- 建立团队文化
- 非等级实践

**示例:**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 互惠 (Reciprocity)
**它是什么:** 回报收到利益的义务。

**如何工作:**
- 谨慎使用 - 可能感觉具有操纵性
- skills 中很少需要

**何时避免:**
- 几乎总是 (其他原则更有效)

### 7. 喜好 (Liking)
**它是什么:** 偏向于与我们喜欢的人合作。

**如何工作:**
- **不要用于合规**
- 与诚实反馈文化冲突
- 制造阿谀奉承

**何时避免:**
- 总是用于纪律执行

## 按 Skill 类型组合原则

| Skill 类型 | 使用 | 避免 |
|------------|-----|-------|
| 纪律执行 | Authority + Commitment + Social Proof | Liking, Reciprocity |
| 指导/技术 | Moderate Authority + Unity | Heavy authority |
| 协作 | Unity + Commitment | Authority, Liking |
| 参考 | 仅清晰度 | 所有说服 |

## 为什么这有效: 心理学

**明确的规则减少合理化:**
- "YOU MUST" 移除决策疲劳
- 绝对语言消除 "is this an exception?" 问题
- 明确的反合理化反击堵塞特定漏洞

**实施意图创造自动行为:**
- 清晰触发器 + 必需行动 = 自动执行
- "When X, do Y" 比 "generally do Y" 更有效
- 减少合规的认知负荷

**LLMs 是类人的 (parahuman):**
- 在包含这些模式的人类文本上训练
- 权威语言在训练数据中先于合规
- 频繁建模承诺序列 (声明 → 行动)
- 社会证明模式 (everyone does X) 建立规范

## 道德使用 (Ethical Use)

**合法:**
- 确保遵循关键实践
- 创建有效文档
- 防止可预测的失败

**非法:**
- 为了个人利益操纵
- 制造虚假紧迫感
- 基于内疚的合规

**测试:** 如果用户完全理解，这项技术是否符合用户的真正利益？

## 研究引文

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 七项说服原则
- 影响研究的经验基础

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 测试了 7 项原则，N=28,000 LLM conversations
- 依从率通过说服技术增加 33% → 72%
- 权威、承诺、稀缺性最有效
- 验证 LLM 行为的类人模型

## 快速参考 (Quick Reference)

设计 skill 时，问:

1.  **它是什么类型？** (纪律 vs. 指导 vs. 参考)
2.  **我试图改变什么行为？**
3.  **哪些原则适用？** (通常是 authority + commitment for discipline)
4.  **我是否结合了太多？** (不要使用所有七个)
5.  **这合乎道德吗？** (服务于用户的真正利益？)
