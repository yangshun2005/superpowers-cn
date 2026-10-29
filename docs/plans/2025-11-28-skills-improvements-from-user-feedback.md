# 来自用户反馈的技能改进

**日期：** 2025-11-28
**状态：** 草稿
**来源：** 两个 Claude 实例在实际开发场景中使用 superpowers 的反馈

---

## 执行摘要

两个 Claude 实例提供了实际开发会话的详细反馈。反馈揭示了当前技能中的**系统性缺口**——即使遵循了技能，仍让本可预防的 bug 上线。

**关键洞察：** 这些是问题报告，不只是解决方案建议。问题是真实的；解决方案需要仔细评估。

**核心主题：**
1. **验证缺口** - 我们验证操作成功，但不验证是否达到预期结果
2. **进程卫生** - 后台进程累积并跨子代理干扰
3. **上下文优化** - 子代理收到过多无关信息
4. **缺少自我反思** - 没有提示在交接前审视自己的工作
5. **Mock 安全性** - Mock 可能偏离接口而不被发现
6. **技能激活** - 技能存在但未被阅读/使用

---

## 已识别的问题

### 问题 1：配置变更验证缺口

**发生了什么：**
- 子代理测试"OpenAI 集成"
- 设置了 `OPENAI_API_KEY` 环境变量
- 得到状态码 200 响应
- 报告"OpenAI 集成正常"
- **但是** 响应包含 `"model": "claude-sonnet-4-20250514"`——实际使用的是 Anthropic

**根因：**
`verification-before-completion` 检查操作成功，但不检查结果是否反映预期的配置变更。

**影响：** 高——对集成测试产生虚假信心，bug 上线到生产环境

**典型失败模式：**
- 切换 LLM 提供商 → 验证状态码 200 但不检查模型名称
- 启用功能开关 → 验证无错误但不检查功能是否激活
- 切换环境 → 验证部署成功但不检查环境变量

---

### 问题 2：后台进程累积

**发生了什么：**
- 会话期间派发了多个子代理
- 每个都启动了后台服务器进程
- 进程累积（4+ 个服务器在运行）
- 陈旧进程仍占用端口
- 后续 E2E 测试命中配置错误的陈旧服务器
- 测试结果混乱/错误

**根因：**
子代理是无状态的——不知道前一个子代理的进程。没有清理协议。

**影响：** 中高——测试命中错误服务器，虚假通过/失败，调试混乱

---

### 问题 3：子代理提示中的上下文膨胀

**发生了什么：**
- 标准做法：给子代理完整计划文件去阅读
- 实验：只给任务 + 模式 + 文件 + 验证命令
- 结果：更快、更专注，单次尝试完成更常见

**根因：**
子代理在无关的计划章节上浪费 token 和注意力。

**影响：** 中——执行更慢，更多失败尝试

**有效的做法：**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 问题 4：交接前缺少自我反思

**发生了什么：**
- 添加了自我反思提示："用全新的眼光审视你的工作——有什么可以改进的？"
- 任务 5 的实现者发现失败的测试是实现 bug 导致的，不是测试 bug
- 追溯到第 99 行：`strings.Join(metadata.Entrypoint, " ")` 生成了无效的 Docker 语法
- 没有自我反思，只会报告"测试失败"而不给根因

**根因：**
实现者不会自然地在报告完成前退一步审视自己的工作。

**影响：** 中——实现者本可发现的 bug 被交给了审查者

---

### 问题 5：Mock 与接口偏离

**发生了什么：**
```typescript
// Interface defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) defines cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // Wrong!
  })),
}));
```
- 测试通过
- 运行时崩溃："adapter.cleanup is not a function"

**根因：**
Mock 源自 buggy 代码的调用，而非接口定义。TypeScript 无法捕获方法名错误的内联 mock。

**影响：** 高——测试给出虚假信心，运行时崩溃

**为什么 testing-anti-patterns 没能阻止：**
该技能涵盖了测试 mock 行为和未理解就 mock，但没有涵盖"从接口而非实现派生 mock"这一特定模式。

---

### 问题 6：代码审查者文件访问

**发生了什么：**
- 派发了代码审查者子代理
- 找不到测试文件："该文件似乎不存在于仓库中"
- 文件实际存在
- 审查者不知道需要先显式读取

**根因：**
审查者提示不包含显式文件读取指令。

**影响：** 低中——审查失败或不完整

---

### 问题 7：修复工作流延迟

**发生了什么：**
- 实现者在自我反思中发现 bug
- 实现者知道修复方法
- 当前工作流：报告 → 我派发修复者 → 修复者修复 → 我验证
- 额外往返增加延迟而无附加价值

**根因：**
实现者已诊断出问题时，实现者与修复者角色之间仍严格分离。

**影响：** 低——延迟，但无正确性问题

---

### 问题 8：技能未被阅读

**发生了什么：**
- `testing-anti-patterns` 技能存在
- 人类和子代理在写测试前都没读它
- 本可预防一些问题（虽然不是全部——见问题 5）

**根因：**
没有强制子代理阅读相关技能。没有提示包含技能阅读。

**影响：** 中——技能投入如果不用就是浪费

---

## 建议的改进

### 1. verification-before-completion：添加配置变更验证

**新增章节：**

```markdown
## Verifying Configuration Changes

When testing changes to configuration, providers, feature flags, or environment:

**Don't just verify the operation succeeded. Verify the output reflects the intended change.**

### Common Failure Pattern

Operation succeeds because *some* valid config exists, but it's not the config you intended to test.

### Examples

| Change | Insufficient | Required |
|--------|-------------|----------|
| Switch LLM provider | Status 200 | Response contains expected model name |
| Enable feature flag | No errors | Feature behavior actually active |
| Change environment | Deploy succeeds | Logs/vars reference new environment |
| Set credentials | Auth succeeds | Authenticated user/context is correct |

### Gate Function

```
BEFORE claiming configuration change works:

1. IDENTIFY: What should be DIFFERENT after this change?
2. LOCATE: Where is that difference observable?
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: Command that shows the observable difference
4. VERIFY: Output contains expected difference
5. ONLY THEN: Claim configuration change works

Red flags:
  - "Request succeeded" without checking content
  - Checking status code but not response body
  - Verifying no errors but not positive confirmation
```

**Why this works:**
Forces verification of INTENT, not just operation success.

---

### 2. subagent-driven-development：为 E2E 测试添加进程卫生

**新增章节：**

```markdown
## Process Hygiene for E2E Tests

When dispatching subagents that start services (servers, databases, message queues):

### Problem

Subagents are stateless - they don't know about processes started by previous subagents. Background processes persist and can interfere with later tests.

### Solution

**Before dispatching E2E test subagent, include cleanup in prompt:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- Stale processes serve requests with wrong config
- Port conflicts cause silent failures
- Process accumulation slows system
- Confusing test results (hitting wrong server)
```

**权衡分析：**
- 给提示增加样板内容
- 但能防止非常令人困惑的调试
- 对 E2E 测试子代理值得

---

### 3. subagent-driven-development：添加精简上下文选项

**修改步骤 2：用子代理执行任务**

**修改前：**
```
Read that task carefully from [plan-file].
```

**修改后：**
```
## Context Approaches

**Full Plan (default):**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
Use when task is standalone and pattern-based:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern (add similar test, implement similar feature)
- Task is self-contained (doesn't need context from other tasks)
- Pattern reference is sufficient (e.g., "follow TestE2E_FeatureOptionValidation")

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**示例：**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**为什么有效：**
减少 token 使用，提高专注度，在合适时更快完成。

---

### 4. subagent-driven-development：添加自我反思步骤

**修改步骤 2：用子代理执行任务**

**添加到提示模板：**

```
When done, BEFORE reporting back:

Take a step back and review your work with fresh eyes.

Ask yourself:
- Does this actually solve the task as specified?
- Are there edge cases I didn't consider?
- Did I follow the pattern correctly?
- If tests are failing, what's the ROOT CAUSE (implementation bug vs test bug)?
- What could be better about this implementation?

If you identify issues during this reflection, fix them now.

Then report:
- What you implemented
- Self-reflection findings (if any)
- Test results
- Files changed
```

**为什么有效：**
在交接前捕获实现者自己能发现的 bug。有记录的案例：通过自我反思发现了 entrypoint bug。

**权衡：**
每个任务增加约 30 秒，但在审查前捕获问题。

---

### 5. requesting-code-review：添加显式文件读取

**修改 code-reviewer 模板：**

**在开头添加：**

```markdown
## Files to Review

BEFORE analyzing, read these files:

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file.

If you cannot find a file:
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code.
```

**为什么有效：**
显式指令防止"文件未找到"问题。

---

### 6. testing-anti-patterns：添加 Mock 与接口偏离反模式

**新增反模式 6：**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock encodes the bug into the test
- TypeScript can't catch inline mocks with wrong method names
- Test passes because both code and mock are wrong
- Runtime crashes when real object is used

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
BEFORE writing any mock:

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**Detection:**

When you see runtime error "X is not a function" and tests pass:
1. Check if X is mocked
2. Compare mock methods to interface methods
3. Look for method name mismatches
```

**为什么有效：**
直接针对反馈中的失败模式。

---

### 7. subagent-driven-development：要求测试子代理阅读技能

**当任务涉及测试时，添加到提示模板：**

```markdown
BEFORE writing any tests:

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional. Tests that violate anti-patterns will be rejected in review.
```

**为什么有效：**
确保技能实际被使用，而不只是存在。

**权衡：**
每个任务增加时间，但能防止整类 bug。

---

### 8. subagent-driven-development：允许实现者修复自己发现的问题

**修改步骤 2：**

**当前：**
```
Subagent reports back with summary of work.
```

**建议：**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**为什么有效：**
实现者已知修复方法时减少延迟。有记录的案例：本可为 entrypoint bug 节省一次往返。

**权衡：**
提示稍复杂，但端到端更快。

---

## 实施计划

### 阶段 1：高影响、低风险（先做）

1. **verification-before-completion：配置变更验证**
   - 清晰的新增，不改变现有内容
   - 解决高影响问题（测试中的虚假信心）
   - 文件：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：Mock 与接口偏离**
   - 新增反模式，不修改现有内容
   - 解决高影响问题（运行时崩溃）
   - 文件：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：显式文件读取**
   - 模板的简单新增
   - 修复具体问题（审查者找不到文件）
   - 文件：`skills/requesting-code-review/SKILL.md`

### 阶段 2：中等变更（仔细测试）

4. **subagent-driven-development：进程卫生**
   - 新增章节，不改变工作流
   - 解决中高影响问题（测试可靠性）
   - 文件：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自我反思**
   - 改变提示模板（较高风险）
   - 但有记录能捕获 bug
   - 文件：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：技能阅读要求**
   - 增加提示开销
   - 但确保技能实际被使用
   - 文件：`skills/subagent-driven-development/SKILL.md`

### 阶段 3：优化（先验证）

7. **subagent-driven-development：精简上下文选项**
   - 增加复杂度（两种方法）
   - 需要验证不会造成混淆
   - 文件：`skills/subagent-driven-development/SKILL.md`

8
```