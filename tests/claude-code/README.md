# Claude Code Skills 测试

使用 Claude Code CLI 对 superpowers skills 进行自动化测试。

## 概述

本测试套件验证 skills 是否正确加载，以及 Claude 是否按预期遵循它们。测试以无头模式（`claude -p`）调用 Claude Code 并验证行为。

## 要求

- 已安装 Claude Code CLI 且在 PATH 中（`claude --version` 应能正常工作）
- 已安装本地 superpowers 插件（安装方法见主 README）

## 运行测试

### 运行所有快速测试（推荐）：
```bash
./run-skill-tests.sh
```

### 运行集成测试（慢，10-30 分钟）：
```bash
./run-skill-tests.sh --integration
```

### 运行指定测试：
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 以详细输出运行：
```bash
./run-skill-tests.sh --verbose
```

### 设置自定义超时：
```bash
./run-skill-tests.sh --timeout 1800  # 集成测试 30 分钟
```

## 测试结构

### test-helpers.sh
skills 测试的通用函数：
- `run_claude "prompt" [timeout]` - 带提示词运行 Claude
- `assert_contains output pattern name` - 验证模式存在
- `assert_not_contains output pattern name` - 验证模式不存在
- `assert_count output pattern count name` - 验证精确计数
- `assert_order output pattern_a pattern_b name` - 验证顺序
- `create_test_project` - 创建临时测试目录
- `create_test_plan project_dir` - 创建示例计划文件

### 测试文件

每个测试文件：
1. 引入 `test-helpers.sh`
2. 用特定提示词运行 Claude Code
3. 用断言验证预期行为
4. 成功返回 0，失败返回非零

## 测试示例

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# Ask Claude about the skill
output=$(run_claude "What does the my-skill skill do?" 30)

# Verify response
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 当前测试

### 快速测试（默认运行）

#### test-subagent-driven-development.sh
测试 skill 内容与要求（约 2 分钟）：
- Skill 加载与可访问性
- 工作流顺序（规范符合性先于代码质量）
- 已记录自审要求
- 已记录计划读取效率
- 已记录规范符合性审查者的怀疑态度
- 已记录审查循环
- 已记录任务上下文提供

### 集成测试（使用 --integration 标志）

#### test-subagent-driven-development-integration.sh
完整工作流执行测试（约 10-30 分钟）：
- 创建带 Node.js 环境的真实测试项目
- 创建含 2 个任务的实现计划
- 使用 subagent-driven-development 执行计划
- 验证实际行为：
  - 计划在开始时读取一次（而非每个任务读取）
  - 子代理提示词中提供完整任务文本
  - 子代理在报告前执行自审
  - 规范符合性审查先于代码质量审查
  - 规范审查者独立阅读代码
  - 产出可用的实现
  - 测试通过
  - 创建正确的 git 提交

**测试内容：**
- 工作流确实端到端可用
- 我们的改进确实被应用
- 子代理正确遵循 skill
- 最终代码可用且经过测试

#### test-worktree-native-preference.sh
针对 using-git-worktrees skill 的 RED-GREEN-REFACTOR 验证（约 5 分钟）：
- RED：不含 Step 1a 的 skill —— 代理应使用 `git worktree add`
- GREEN：含 Step 1a 的 skill —— 代理应使用原生 EnterWorktree 工具
- PRESSURE：与 GREEN 相同，但在紧迫语境下且存在预先创建的 `.worktrees/`
- 演练场景 `worktree-creation-under-pressure.yaml` 仅覆盖 PRESSURE 阶段

## 添加新测试

1. 创建新测试文件：`test-<skill-name>.sh`
2. 引入 test-helpers.sh
3. 使用 `run_claude` 和断言编写测试
4. 添加到 `run-skill-tests.sh` 中的测试列表
5. 设为可执行：`chmod +x test-<skill-name>.sh`

## 超时注意事项

- 默认超时：每个测试 5 分钟
- Claude Code 响应可能需要时间
- 如有需要，用 `--timeout` 调整
- 测试应保持聚焦，避免长时间运行

## 调试失败的测试

使用 `--verbose` 可看到完整的 Claude 输出：
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

不使用 verbose 时，仅失败项显示输出。

## CI/CD 集成

在 CI 中运行：
```bash
# Run with explicit timeout for CI environments
./run-skill-tests.sh --timeout 900

# Exit code 0 = success, non-zero = failure
```

## 说明

- 测试验证 skill 的*指令*，而非完整执行
- 完整工作流测试会非常慢
- 聚焦于验证关键 skill 要求
- 测试应具有确定性
- 避免测试实现细节