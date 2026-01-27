# Code Quality Reviewer Prompt Template

当 dispatch 一个 code quality reviewer subagent 时使用此模板。

**目的:** 验证实施是否构建良好（整洁、经过测试、可维护）

**仅在 spec compliance review 通过后 dispatch。**

```
Task tool (superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
  DESCRIPTION: [task summary]
```

**Code reviewer 返回:** Strengths, Issues (Critical/Important/Minor), Assessment
