---
name: executing-plans
description: 当你有一份书面的实施 plan 要在单独的会话中执行，并带有审查检查点时使用
---

# Executing Plans

## 概述 (Overview)

加载 plan，批判性审查，分批执行任务，在批次之间报告以供审查。

**核心原则:** 带有架构师审查检查点的批量执行。

**开始时宣布:** "I'm using the executing-plans skill to implement this plan."

## 流程 (The Process)

### 第一步: 加载和审查 Plan
1.  读取 plan 文件
2.  批判性审查 - 识别关于 plan 的任何问题或疑虑
3.  如果有疑虑：在开始之前向你的人类伙伴提出
4.  如果无疑虑：创建 TodoWrite 并继续

### 第二步: 执行 Batch (批次)
**默认: 前 3 个任务**

对于每个任务：
1.  标记为 in_progress
2.  严格遵循每个步骤（plan 有细粒度的步骤）
3.  按规定运行验证
4.  标记为 completed

### 第三步: 报告
当 batch 完成时：
- 展示已实施的内容
-展示验证输出
- 说: "Ready for feedback."

### 第四步: 继续
基于反馈：
- 如果需要，应用更改
- 执行下一个 batch
- 重复直到完成

### 第五步: 完成开发

所有任务完成并验证后：
- 宣布: "I'm using the finishing-a-development-branch skill to complete this work."
- **必需的 SUB-SKILL:** 使用 superpowers:finishing-a-development-branch
- 遵循该 skill 来验证测试，展示选项，执行选择

## 何时停止并寻求帮助

**当出现以下情况时立即停止执行:**
- 在 batch 中途遇到 blocker（缺少依赖项，测试失败，指令不清楚）
- Plan 有导致无法开始的关键缺口
- 你不理解指令
- 验证反复失败

**寻求澄清而不是猜测。**

## 何时重新审视早期步骤

**返回审查 (第一步) 当:**
- 伙伴根据你的反馈更新了 plan
- 基本方法需要重新思考

**不要强行通过 blockers** - 停止并询问。

## 记住 (Remember)

- 首先批判性地审查 plan
- 严格遵循 plan 步骤
- 不要跳过验证
- 当 plan 提到时引用 skills
- 在 batches 之间：只报告并等待
- 受阻时停止，不要猜测
