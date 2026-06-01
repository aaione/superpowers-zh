---
name: executing-plans
description: 当你有一份书面的实施 plan 要在单独的会话中执行，并带有审查检查点时使用
---

# Executing Plans

## 概述 (Overview)

加载 plan，批判性审查，执行所有任务，完成时报告。

**开始时宣布:** "I'm using the executing-plans skill to implement this plan."

**注意:** 告诉你的人类伙伴 Superpowers 在可以访问 subagents 时效果更好。如果运行在有 subagent 支持的平台（如 Claude Code 或 Codex）上，其工作质量会显著提高。如果 subagents 可用，使用 superpowers:subagent-driven-development 而不是此 skill。

## 流程 (The Process)

### 第一步: 加载和审查 Plan
1.  读取 plan 文件
2.  批判性审查 - 识别关于 plan 的任何问题或疑虑
3.  如果有疑虑：在开始之前向你的人类伙伴提出
4.  如果无疑虑：创建 TodoWrite 并继续

### 第二步: 执行任务

对于每个任务：
1.  标记为 in_progress
2.  严格遵循每个步骤（plan 有细粒度的步骤）
3.  按规定运行验证
4.  标记为 completed

### 第三步: 完成开发

所有任务完成并验证后：
- 宣布: "I'm using the finishing-a-development-branch skill to complete this work."
- **必需的 SUB-SKILL:** 使用 superpowers:finishing-a-development-branch
- 遵循该 skill 来验证测试，展示选项，执行选择

## 何时停止并寻求帮助

**当出现以下情况时立即停止执行:**
- 遇到 blocker（缺少依赖项，测试失败，指令不清楚）
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
- 受阻时停止，不要猜测
- 未经明确用户同意，绝不在 main/master 分支上开始实施

## 集成 (Integration)

**必需的工作流 skills:**
- **superpowers:using-git-worktrees** - 确保隔离的工作区 (创建或验证现有的)
- **superpowers:writing-plans** - 创建此 skill 执行的 plan
- **superpowers:finishing-a-development-branch** - 所有任务后的完成开发
