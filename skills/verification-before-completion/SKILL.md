---
name: verification-before-completion
description: 当准备声称工作已完成、修复或通过时使用，在 commit 或创建 PR 之前 - 需要运行验证命令并在做出任何成功声明之前确认输出；永远是先证据后断言
---

# Verification Before Completion (完成前验证)

## 概述 (Overview)

未经验证就声称工作完成是不诚实，而不是效率。

**核心原则:** 证据先于声明，永远。

**违反此规则的字面意思就是违反此规则的精神。**

## 铁律 (The Iron Law)

```
没有新鲜的验证证据就没有完成声明 (NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE)
```

如果你没有在此消息中运行验证命令，你不能声称它通过。

## Gate Function (把关函数)

```
在声称任何状态或表达满意之前:

1. 识别: 什么命令证明此声明？
2. 运行: 执行完整命令 (fresh, complete)
3. 阅读: 完整输出，检查退出代码，计算失败
4. 验证: 输出是否确认声明？
   - 如果 NO: 带着证据陈述实际状态
   - 如果 YES: 带着证据陈述声明
5. 仅在此时: 做出声明

跳过任何步骤 = 撒谎，不是验证
```

## 常见失败 (Common Failures)

| 声明 | 需要 | 不足以 |
|-------|----------|----------------|
| Tests pass | Test command output: 0 failures | 之前的运行, "should pass" |
| Linter clean | Linter output: 0 errors | 部分检查, 推断 |
| Build succeeds | Build command: exit 0 | Linter passing, logs look good |
| Bug fixed | Test original symptom: passes | 代码已更改, 假定已修复 |
| Regression test works | Red-green cycle verified | Test passes once |
| Agent completed | VCS diff shows changes | Agent 报告 "success" |
| Requirements met | Line-by-line checklist | Tests passing |

## 危险信号 - 停止

- 使用 "should", "probably", "seems to"
- 在验证之前表达满意 ("Great!", "Perfect!", "Done!", etc.)
- 准备 commit/push/PR 而未经验证
- 信任 agent 成功报告
- 依赖部分验证
- 想着 "just this once"
- 累了想结束工作
- **任何暗示成功而没有运行验证的措辞**

## 合理化预防 (Rationalization Prevention)

| 借口 | 现实 |
|--------|---------|
| "现在应该能用了" | **运行**验证 |
| "我很自信" | 自信 ≠ 证据 |
| "就这一次" | 无例外 |
| "Linter 通过了" | Linter ≠ 编译器 |
| "Agent 说成功了" | 独立验证 |
| "我累了" | 疲惫 ≠ 借口 |
| "部分检查足够了" | 部分证明不了什么 |
| "措辞不同所以规则不适用" | 精神高于字面 |

## 关键模式 (Key Patterns)

**Tests:**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**Regression tests (TDD Red-Green):**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**Build:**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**Requirements:**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**Agent delegation:**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## 为什么这很重要

来自 24 个失败记忆:
- 你的人类伙伴说 "I don't believe you" - 信任破裂
- 发布的未定义函数 - 会崩溃
- 发布的缺失需求 - 不完整的功能
- 浪费在虚假完成上的时间 → 重定向 → 返工
- 违反: "Honesty is a core value. If you lie, you'll be replaced."

## 何时应用 (When To Apply)

**总是 (ALWAYS) 在之前:**
- 成功/完成声明的任何变体
- 满意的任何表达
- 关于工作状态的任何积极陈述
- Committing, PR creation, task completion
- 移动到下一个任务
- 委托给 agents

**规则适用于:**
- 确切的短语
- 释义和同义词
- 成功的暗示
- 任何暗示完成/正确性的沟通

## 总结 (The Bottom Line)

**验证没有捷径。**

运行命令。阅读输出。**然后**声称结果。

这是不可谈判的。
