# 测试 Superpowers Skills

本文档描述了如何测试 Superpowers skills，特别是像 `subagent-driven-development` 这样的复杂 skills 的集成测试。

## 概述 (Overview)

测试涉及 subagents、工作流和复杂交互的 skills 需要以 headless 模式运行实际的 Claude Code 会话，并通过会话记录验证其行为。

## 测试结构 (Test Structure)

```
tests/
├── claude-code/
│   ├── test-helpers.sh                    # Shared test utilities
│   ├── test-subagent-driven-development-integration.sh
│   ├── analyze-token-usage.py             # Token analysis tool
│   └── run-skill-tests.sh                 # Test runner (if exists)
```

## 运行测试 (Running Tests)

### 集成测试 (Integration Tests)

集成测试使用实际的 skills 执行真实的 Claude Code 会话:

```bash
# Run the subagent-driven-development integration test
cd tests/claude-code
./test-subagent-driven-development-integration.sh
```

**注意:** 集成测试可能需要 10-30 分钟，因为它们执行带有多个 subagents 的实际实施计划。

### 要求 (Requirements)

- 必须从 **superpowers plugin directory** 运行 (不是从 temp 目录)
- Claude Code 必须已安装且可用作 `claude` 命令
- 本地 dev marketplace 必须已启用: `"superpowers@superpowers-dev": true` in `~/.claude/settings.json`

## 集成测试: subagent-driven-development

### 它测试什么 (What It Tests)

集成测试验证 `subagent-driven-development` skill 正确地:

1.  **Plan Loading**: 在开始时读取一次计划
2.  **Full Task Text**: 向 subagents 提供完整的任务描述 (不让他们读取文件)
3.  **Self-Review**: 确保 subagents 在报告前执行自我审查
4.  **Review Order**: 在代码质量审查之前运行规范合规性审查
5.  **Review Loops**: 当发现问题时使用审查循环
6.  **Independent Verification**: 规范审查者独立阅读代码，不信任实施者的报告

### 它如何工作 (How It Works)

1.  **Setup**: 创建一个带有最小实施计划的临时 Node.js 项目
2.  **Execution**: 以 headless 模式运行带有 skill 的 Claude Code
3.  **Verification**: 解析会话记录 (`.jsonl` file) 以验证:
    - Skill 工具被调用
    - Subagents 被分派 (Task tool)
    - TodoWrite 用于跟踪
    - 实施文件被创建
    - 测试通过
    - Git commits 显示正确的工作流
4.  **Token Analysis**: 按 subagent 显示 token 使用细分

### 测试输出 (Test Output)

```
========================================
 Integration Test: subagent-driven-development
========================================

Test project: /tmp/tmp.xyz123

=== Verification Tests ===

Test 1: Skill tool invoked...
  [PASS] subagent-driven-development skill was invoked

Test 2: Subagents dispatched...
  [PASS] 7 subagents dispatched

Test 3: Task tracking...
  [PASS] TodoWrite used 5 time(s)

Test 6: Implementation verification...
  [PASS] src/math.js created
  [PASS] add function exists
  [PASS] multiply function exists
  [PASS] test/math.test.js created
  [PASS] Tests pass

Test 7: Git commit history...
  [PASS] Multiple commits created (3 total)

Test 8: No extra features added...
  [PASS] No extra features added

=========================================
 Token Usage Analysis
=========================================

Usage Breakdown:
----------------------------------------------------------------------------------------------------
Agent           Description                          Msgs      Input     Output      Cache     Cost
----------------------------------------------------------------------------------------------------
main            Main session (coordinator)             34         27      3,996  1,213,703 $   4.09
3380c209        implementing Task 1: Create Add Function     1          2        787     24,989 $   0.09
34b00fde        implementing Task 2: Create Multiply Function     1          4        644     25,114 $   0.09
3801a732        reviewing whether an implementation matches...   1          5        703     25,742 $   0.09
4c142934        doing a final code review...                    1          6        854     25,319 $   0.09
5f017a42        a code reviewer. Review Task 2...               1          6        504     22,949 $   0.08
a6b7fbe4        a code reviewer. Review Task 1...               1          6        515     22,534 $   0.08
f15837c0        reviewing whether an implementation matches...   1          6        416     22,485 $   0.07
----------------------------------------------------------------------------------------------------

TOTALS:
  Total messages:         41
  Input tokens:           62
  Output tokens:          8,419
  Cache creation tokens:  132,742
  Cache read tokens:      1,382,835

  Total input (incl cache): 1,515,639
  Total tokens:             1,524,058

  Estimated cost: $4.67
  (at $3/$15 per M tokens for input/output)

========================================
 Test Summary
========================================

STATUS: PASSED
```

## Token Analysis Tool

### 用法 (Usage)

从任何 Claude Code 会话分析 token 使用:

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

### 查找会话文件 (Finding Session Files)

会话记录存储在 `~/.claude/projects/` 中，工作目录路径被编码:

```bash
# Example for /Users/jesse/Documents/GitHub/superpowers/superpowers
SESSION_DIR="$HOME/.claude/projects/-Users-jesse-Documents-GitHub-superpowers-superpowers"

# Find recent sessions
ls -lt "$SESSION_DIR"/*.jsonl | head -5
```

### 它显示什么 (What It Shows)

- **Main session usage**: 协调者 (你或 main Claude 实例) 的 Token 使用
- **Per-subagent breakdown**: 每个 Task 调用带有:
  - Agent ID
  - Description (extracted from prompt)
  - Message count
  - Input/output tokens
  - Cache usage
  - Estimated cost
- **Totals**: 总体 token 使用和成本估算

### 理解输出

- **High cache reads**: 好 - 意味着 prompt caching 正在工作
- **High input tokens on main**: 预期 - 协调者拥有完整的上下文
- **Similar costs per subagent**: 预期 - 每个 subagent 获得相似的任务复杂性
- **Cost per task**: 典型范围是每个 subagent $0.05-$0.15，取决于任务

## 故障排除 (Troubleshooting)

### Skills 未加载

**问题**: 运行 headless 测试时未找到 Skill

**解决方案**:
1. 确保你正在从 superpowers 目录运行: `cd /path/to/superpowers && tests/...`
2. 检查 `~/.claude/settings.json` 在 `enabledPlugins` 中有 `"superpowers@superpowers-dev": true`
3. 验证 skill 存在于 `skills/` 目录中

### 权限错误

**问题**: Claude 被阻止写入文件或访问目录

**解决方案**:
1. 使用 `--permission-mode bypassPermissions` 标志
2. 使用 `--add-dir /path/to/temp/dir` 授予对测试目录的访问权限
3. 检查测试目录的文件权限

### 测试超时

**问题**: 测试时间太长并超时

**解决方案**:
1. 增加超时: `timeout 1800 claude ...` (30 minutes)
2. 检查 skill 逻辑中的无限循环
3. 审查 subagent 任务复杂性

### 会话文件未找到

**问题**: 测试运行后找不到会话记录

**解决方案**:
1. 检查 `~/.claude/projects/` 中的正确项目目录
2. 使用 `find ~/.claude/projects -name "*.jsonl" -mmin -60` 查找最近的会话
3. 验证测试实际运行了 (检查测试输出中的错误)

## 编写新的集成测试

### 模板 (Template)

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# Create test project
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT

# Set up test files...
cd "$TEST_PROJECT"

# Run Claude with skill
PROMPT="Your test prompt here"
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT" \
  --allowed-tools=all \
  --add-dir "$TEST_PROJECT" \
  --permission-mode bypassPermissions \
  2>&1 | tee output.txt

# Find and analyze session
WORKING_DIR_ESCAPED=$(echo "$SCRIPT_DIR/../.." | sed 's/\\//-/g' | sed 's/^-//')
SESSION_DIR="$HOME/.claude/projects/$WORKING_DIR_ESCAPED"
SESSION_FILE=$(find "$SESSION_DIR" -name "*.jsonl" -type f -mmin -60 | sort -r | head -1)

# Verify behavior by parsing session transcript
if grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"; then
    echo "[PASS] Skill was invoked"
fi

# Show token analysis
python3 "$SCRIPT_DIR/analyze-token-usage.py" "$SESSION_FILE"
```

### 最佳实践 (Best Practices)

1.  **Always cleanup**: 使用 trap 清理临时目录
2.  **Parse transcripts**: 不要 grep 面向用户的输出 - 解析 `.jsonl` 会话文件
3.  **Grant permissions**: 使用 `--permission-mode bypassPermissions` 和 `--add-dir`
4.  **Run from plugin dir**: Skills 仅在从 superpowers 目录运行时加载
5.  **Show token usage**: 始终包含 token 分析以实现成本可见性
6.  **Test real behavior**: 验证实际创建的文件、通过的测试、提交的 commits

## 会话记录格式 (Session Transcript Format)

会话记录是 JSONL (JSON Lines) 文件，其中每行是一个代表消息或工具结果的 JSON 对象。

### 关键字段 (Key Fields)

```json
{
  "type": "assistant",
  "message": {
    "content": [...],
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703
    }
  }
}
```

### 工具结果 (Tool Results)

```json
{
  "type": "user",
  "toolUseResult": {
    "agentId": "3380c209",
    "usage": {
      "input_tokens": 2,
      "output_tokens": 787,
      "cache_read_input_tokens": 24989
    },
    "prompt": "You are implementing Task 1...",
    "content": [{"type": "text", "text": "..."}]
  }
}
```

`agentId` 字段链接到 subagent 会话，`usage` 字段包含该特定 subagent 调用的 token 使用情况。
