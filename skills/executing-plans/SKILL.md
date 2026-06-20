---
name: executing-plans
description: 当你有一个书面实现计划，需要在带有 review 检查点的独立会话中执行时使用
---

# 执行计划 (Executing Plans)

## 概述 (Overview)

加载计划，批判性地审查，执行所有任务，完成后报告。

**开始时宣布：** "我正在使用 executing-plans skill 来实现这个计划。"

**注意：** 告诉你的 human partner，Superpowers 在可以访问 subagent 的情况下工作得好得多。如果在支持 subagent 的平台上运行（Claude Code、Codex CLI、Codex App、Copilot CLI 和 Gemini CLI 都符合条件；见 `../using-superpowers/references/` 中的各平台工具参考），其工作质量会显著更高。如果 subagent 可用，使用 superpowers:subagent-driven-development 而非本 skill。

## 流程 (The Process)

### 步骤 1：加载并审查计划
1. 读取计划文件
2. 批判性审查——识别关于计划的任何问题或疑虑
3. 如有疑虑：在开始之前向你的 human partner 提出
4. 如无疑虑：为计划项创建 todos 并继续

### 步骤 2：执行任务

对每个任务：
1. 标记为 in_progress
2. 严格遵循每一步（计划有拆分好的小步骤）
3. 按指定运行验证
4. 标记为 completed

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣布："我正在使用 finishing-a-development-branch skill 来完成这项工作。"
- **必需的子 skill：** 使用 superpowers:finishing-a-development-branch
- 遵循该 skill 验证测试、呈现选项、执行选择

## 何时停下并寻求帮助 (When to Stop and Ask for Help)

**立即停止执行，当：**
- 遇到阻碍（缺少依赖、测试失败、指令不清）
- 计划有关键缺口阻碍开始
- 你不理解某条指令
- 验证反复失败

**请寻求澄清，而非猜测。**

## 何时回到更早步骤 (When to Revisit Earlier Steps)

**回到审查（步骤 1），当：**
- 伙伴根据你的反馈更新了计划
- 根本方法需要重新思考

**不要硬闯阻碍** —— 停下并询问。

## 记住 (Remember)

- 先批判性审查计划
- 严格遵循计划步骤
- 不要跳过验证
- 计划要求时引用 skills
- 受阻时停下，不要猜测
- 没有用户明确同意，绝不擅自在 main/master 分支上开始实现

## 集成 (Integration)

**必需的工作流 skills：**
- **superpowers:using-git-worktrees** - 确保隔离工作区（创建一个或验证已存在）
- **superpowers:writing-plans** - 创建本 skill 执行的计划
- **superpowers:finishing-a-development-branch** - 在所有任务之后完成开发
