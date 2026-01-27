# Creation Log: Systematic Debugging Skill

参考示例：提取、构建和使关键 skill 无懈可击。

## 来源材料 (Source Material)

从 `/Users/jesse/.claude/CLAUDE.md` 提取的调试框架:
- 4 阶段系统流程 (Investigation → Pattern Analysis → Hypothesis → Implementation)
- 核心指令: ALWAYS 找到根本原因, NEVER 修复症状
- 旨在抵制时间压力和合理化的规则

## 提取决策 (Extraction Decisions)

**包含什么:**
- 带有所有规则的完整 4 阶段框架
- 反捷径 ("NEVER fix symptom", "STOP and re-analyze")
- 抗压力语言 ("even if faster", "even if I seem in a hurry")
- 每个阶段的具体步骤

**排除什么:**
- 项目特定的上下文
- 同一规则的重复变体
- 叙述性解释 (压缩为原则)

## 遵循 skill-creation/SKILL.md 的结构

1.  **Rich when_to_use** - 包含症状和反模式
2.  **Type: technique** - 带有步骤的具体流程
3.  **Keywords** - "root cause", "symptom", "workaround", "debugging", "investigation"
4.  **Flowchart** - "fix failed" → re-analyze vs add more fixes 的决策点
5.  **Phase-by-phase breakdown** - 可扫描的清单格式
6.  **Anti-patterns section** - 什么**不**该做 (对这个 skill 至关重要)

## 无懈可击元素 (Bulletproofing Elements)

框架旨在在压力下抵制合理化:

### 语言选择
- "ALWAYS" / "NEVER" (not "should" / "try to")
- "even if faster" / "even if I seem in a hurry"
- "STOP and re-analyze" (explicit pause)
- "Don't skip past" (catches the actual behavior)

### 结构性防御
- **Phase 1 required** - 无法跳到实施
- **Single hypothesis rule** - 强制思考，防止乱枪打鸟式修复
- **Explicit failure mode** - "IF your first fix doesn't work" 带有强制行动
- **Anti-patterns section** - 准确显示捷径看起来是什么样子的

### 冗余
- 根本原因指令在 overview + when_to_use + Phase 1 + implementation rules 中
- "NEVER fix symptom" 在不同上下文中出现 4 次
- 每个阶段都有明确的 "don't skip" 指导

## 测试方法 (Testing Approach)

遵循 skills/meta/testing-skills-with-subagents 创建了 4 个验证测试:

### Test 1: Academic Context (No Pressure)
- Simple bug, no time pressure
- **Result:** 完全合规，完整调查

### Test 2: Time Pressure + Obvious Quick Fix
- User "in a hurry", symptom fix looks easy
- **Result:** 抵制捷径，遵循完整流程，找到真正根本原因

### Test 3: Complex System + Uncertainty
- Multi-layer failure, unclear if can find root cause
- **Result:** 系统调查，追踪所有层，找到源头

### Test 4: Failed First Fix
- Hypothesis doesn't work, temptation to add more fixes
- **Result:** 停止，重新分析，形成新假设 (no shotgun)

**所有测试通过。** 未发现合理化。

## 迭代 (Iterations)

### Initial Version
- 完整 4 阶段框架
- 反模式部分
- "fix failed" 决策的流程图

### Enhancement 1: TDD Reference
- 添加到 skills/testing/test-driven-development 的链接
- 说明解释 TDD 的 "simplest code" ≠ debugging 的 "root cause"
- 防止方法论之间的混淆

## 最终结果 (Final Outcome)

无懈可击的 skill:
- ✅ 明确强制根本原因调查
- ✅ 抵制时间压力合理化
- ✅ 为每个阶段提供具体步骤
- ✅ 明确显示反模式
- ✅ 在多种压力场景下测试
- ✅ 阐明与 TDD 的关系
- ✅ 准备好使用

## 关键见解 (Key Insight)

**最重要的无懈可击:** 反模式部分显示了当时感觉合理的捷径。当 Claude 认为 "I'll just add this one quick fix" 时，看到那个确切模式被列为错误会产生认知摩擦。

## 用法示例

当遇到 bug 时:
1. Load skill: skills/debugging/systematic-debugging
2. Read overview (10 sec) - reminded of mandate
3. Follow Phase 1 checklist - forced investigation
4. If tempted to skip - see anti-pattern, stop
5. Complete all phases - root cause found

**时间投资:** 5-10 minutes
**节省时间:** Hours of symptom-whack-a-mole

---

*Created: 2025-10-03*
*Purpose: Reference example for skill extraction and bulletproofing*
