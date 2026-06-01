# 文档审查系统设计

## 概述

在 superpowers 工作流中添加两个新的审查阶段：

1. **Spec 文档审查** - 在 brainstorming 之后，writing-plans 之前
2. **Plan 文档审查** - 在 writing-plans 之后，实施之前

两者都遵循实施审查所使用的迭代循环模式。

## Spec 文档审查者

**目的：** 验证 spec 完整、一致，并准备好进行实施规划。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 要查找的内容 |
|----------|------------------|
| 完整性 | TODO、占位符、"TBD"、不完整的章节 |
| 覆盖度 | 缺失的错误处理、边界情况、集成点 |
| 一致性 | 内部矛盾、冲突的需求 |
| 清晰度 | 模糊的需求 |
| YAGNI | 未被要求的功能、过度工程 |

**输出格式：**
```
## Spec 审查

**状态：** 通过 | 发现问题

**问题（如有）：**
- [章节 X]: [问题] - [为什么重要]

**建议（参考性质）：**
- [不阻碍通过的建议]
```

**审查循环：** 发现问题 -> brainstorming agent 修复 -> 重新审查 -> 重复直到通过。

**调度机制：** 使用 Task tool 配合 `subagent_type: general-purpose`。审查者 prompt 模板提供完整的 prompt。brainstorming skill 的 controller 调度审查者。

## Plan 文档审查者

**目的：** 验证 plan 完整、与 spec 匹配，并具有适当的任务分解。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 要查找的内容 |
|----------|------------------|
| 完整性 | TODO、占位符、不完整的任务 |
| Spec 对齐 | Plan 覆盖 spec 需求，没有范围蔓延 |
| 任务分解 | 任务原子化、边界清晰 |
| 任务语法 | 任务和步骤使用复选框语法 |
| Chunk 大小 | 每个 chunk 不超过 1000 行 |

**Chunk 定义：** chunk 是 plan 文档内按 `## Chunk N: <name>` 标题划分的任务逻辑分组。writing-plans skill 基于逻辑阶段（例如"基础"、"核心功能"、"集成"）创建这些边界。每个 chunk 应足够自包含，可以独立审查。

**Spec 对齐验证：** 审查者接收：
1. Plan 文档（或当前 chunk）
2. Spec 文档路径供参考

审查者阅读两者并比较需求覆盖情况。

**输出格式：** 与 spec 审查者相同，但限定在当前 chunk 范围内。

**审查流程（逐 chunk）：**
1. writing-plans 创建 chunk N
2. Controller 调度 plan-document-reviewer 并传入 chunk N 内容和 spec 路径
3. 审查者阅读 chunk 和 spec，返回结论
4. 如果有问题：writing-plans agent 修复 chunk N，回到步骤 2
5. 如果通过：继续处理 chunk N+1
6. 重复直到所有 chunk 通过

**调度机制：** 与 spec 审查者相同 - 使用 Task tool 配合 `subagent_type: general-purpose`。

## 更新后的工作流

```
brainstorming -> spec -> SPEC 审查循环 -> writing-plans -> plan -> PLAN 审查循环 -> 实施
```

**Spec 审查循环：**
1. Spec 完成
2. 调度审查者
3. 如果有问题：修复 -> 回到 2
4. 如果通过：继续

**Plan 审查循环：**
1. Chunk N 完成
2. 为 chunk N 调度审查者
3. 如果有问题：修复 -> 回到 2
4. 如果通过：下一个 chunk 或进入实施

## Markdown 任务语法

任务和步骤使用复选框语法：

```markdown
- [ ] ### Task 1: 名称

- [ ] **Step 1:** 描述
  - 文件：path
  - 命令：cmd
```

## 错误处理

**审查循环终止：**
- 没有硬性的迭代限制 - 循环继续直到审查者通过
- 如果循环超过 5 次迭代，controller 应提交给人类进行指导
- 人类可以选择：继续迭代、带着已知问题通过、或中止

**分歧处理：**
- 审查者是参考性质的 - 他们标记问题但不阻碍
- 如果 agent 认为审查者反馈不正确，应在修复中解释原因
- 如果同一问题在 3 次迭代后仍有分歧，提交给人类

**格式错误的审查者输出：**
- Controller 应验证审查者输出包含必需字段（状态、问题（如适用））
- 如果格式错误，重新调度审查者并附上关于预期格式的说明
- 2 次格式错误的回复后，提交给人类

## 需要更改的文件

**新文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改的文件：**
- `skills/brainstorming/SKILL.md` - 在 spec 编写后添加审查循环
- `skills/writing-plans/SKILL.md` - 添加逐 chunk 审查循环，更新任务语法示例
