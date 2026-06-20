# Claude Code Skills 测试 (Claude Code Skills Tests)

使用 Claude Code CLI 对 superpowers skills 进行的自动化测试。

## 概览

该测试套件验证 skills 是否被正确加载，以及 Claude 是否按预期遵循它们。测试以无头模式（headless mode，`claude -p`）调用 Claude Code 并验证行为。

## 前置要求

- 已安装 Claude Code CLI 并在 PATH 中（`claude --version` 应可用）
- 已安装本地 superpowers 插件（安装方式见主 README）

## 运行测试

### 运行所有快速测试（推荐）：
```bash
./run-skill-tests.sh
```

### 运行集成测试（较慢，10-30 分钟）：
```bash
./run-skill-tests.sh --integration
```

### 运行特定测试：
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 显示详细输出：
```bash
./run-skill-tests.sh --verbose
```

### 设置自定义超时：
```bash
./run-skill-tests.sh --timeout 1800  # 集成测试用 30 分钟
```

## 测试结构

### test-helpers.sh
用于 skills 测试的通用函数：
- `run_claude "prompt" [timeout]` - 用指定 prompt 运行 Claude
- `assert_contains output pattern name` - 验证 pattern 存在
- `assert_not_contains output pattern name` - 验证 pattern 不存在
- `assert_count output pattern count name` - 验证精确计数
- `assert_order output pattern_a pattern_b name` - 验证顺序
- `create_test_project` - 创建临时测试目录
- `create_test_plan project_dir` - 创建示例 plan 文件

### 测试文件

每个测试文件：
1. 引入 `test-helpers.sh`
2. 用特定 prompt 运行 Claude Code
3. 使用断言验证预期行为
4. 成功返回 0，失败返回非零值

## 示例测试

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# 向 Claude 询问该 skill
output=$(run_claude "What does the my-skill skill do?" 30)

# 验证响应
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 当前测试

### 快速测试（默认运行）

#### test-subagent-driven-development.sh
测试 skill 内容和需求（约 2 分钟）：
- Skill 加载和可访问性
- 工作流顺序（spec 合规优先于代码质量）
- 自我审查需求已记录
- 计划读取效率已记录
- Spec 合规审查者的怀疑态度已记录
- 审查循环已记录
- 任务上下文提供已记录

### 集成测试（使用 --integration 标志）

#### test-subagent-driven-development-integration.sh
完整工作流执行测试（约 10-30 分钟）：
- 创建带 Node.js 设置的真实测试项目
- 创建含 2 个任务的实现计划
- 使用 subagent-driven-development 执行计划
- 验证实际行为：
  - 计划在开始时读取一次（而非每个任务都读取）
  - 在 subagent 的 prompt 中提供完整的任务文本
  - subagent 在报告前进行自我审查
  - spec 合规审查发生在代码质量审查之前
  - spec 审查者独立阅读代码
  - 产生可工作的实现
  - 测试通过
  - 创建正确的 git 提交

**它测试的内容：**
- 工作流确实能端到端运行
- 我们的改进确实被应用
- subagent 正确遵循 skill
- 最终代码是功能完备且经过测试的

#### test-worktree-native-preference.sh
对 using-git-worktrees skill 的 RED-GREEN-REFACTOR 验证（约 5 分钟）：
- RED：不含 Step 1a 的 skill —— agent 应使用 `git worktree add`
- GREEN：含 Step 1a 的 skill —— agent 应使用原生的 EnterWorktree 工具
- PRESSURE：与 GREEN 相同，但在已有 `.worktrees/` 的情况下施加紧迫感措辞
- Drill 场景 `worktree-creation-under-pressure.yaml` 仅覆盖 PRESSURE 阶段

## 添加新测试

1. 创建新测试文件：`test-<skill-name>.sh`
2. 引入 test-helpers.sh
3. 使用 `run_claude` 和断言编写测试
4. 添加到 `run-skill-tests.sh` 的测试列表中
5. 设置为可执行：`chmod +x test-<skill-name>.sh`

## 超时注意事项

- 默认超时：每个测试 5 分钟
- Claude Code 可能需要时间响应
- 如有需要，用 `--timeout` 调整
- 测试应聚焦，避免运行时间过长

## 调试失败的测试

使用 `--verbose`，你会看到完整的 Claude 输出：
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

不带 verbose 时，只有失败会显示输出。

## CI/CD 集成

在 CI 中运行：
```bash
# 为 CI 环境设置显式超时
./run-skill-tests.sh --timeout 900

# 退出码 0 = 成功，非零 = 失败
```

## 注意事项

- 测试验证的是 skill 的*指令*，而非完整执行
- 完整工作流测试会非常慢
- 聚焦于验证关键的 skill 需求
- 测试应当是确定性的
- 避免测试实现细节
