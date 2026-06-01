# Code Quality Reviewer Prompt 模板

当 dispatch 一个 code quality reviewer subagent 时使用此模板。

**目的:** 验证实施是否构建良好（整洁、经过测试、可维护）

**仅在 spec compliance review 通过后 dispatch。**

```
Task tool (general-purpose):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
  DESCRIPTION: [task summary]
```

**除了标准代码质量关注点外，审查者还应检查:**
- 每个文件是否有一个明确的职责和良好定义的接口？
- 单元是否被分解为可以独立理解和测试的？
- 实施是否遵循计划中的文件结构？
- 这次实施是否创建了已经很大的新文件，或显著增长了现有文件？（不要标记预先存在的文件大小 — 专注于这次更改贡献了什么。）

**Code reviewer 返回:** Strengths, Issues (Critical/Important/Minor), Assessment
