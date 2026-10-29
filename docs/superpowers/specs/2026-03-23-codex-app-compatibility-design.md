# Codex App 兼容性：Worktree 与收尾技能适配

让 superpowers 技能在 Codex App 的沙箱化 worktree 环境中正常工作，同时不破坏现有 Claude Code 或 Codex CLI 的行为。

**工单：** PRI-823

## 动机

Codex App 在它管理的 git worktree 中运行 agent——分离 HEAD，位于 `$CODEX_HOME/worktrees/` 下，并带有 Seatbelt 沙箱，会阻止 `git checkout -b`、`git push` 和网络访问。有三个 superpowers 技能假设 git 访问不受限：`using-git-worktrees` 用命名分支创建手动 worktree，`finishing-a-development-branch` 按分支名合并/推送/创建 PR，`subagent-driven-development` 两者都依赖。

Codex CLI（开源终端工具）没有这个冲突——它没有内置的 worktree 管理。我们的手动 worktree 方案填补了那里的隔离空白。问题特指 Codex App。

## 实证发现

2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙箱 | 完全访问沙箱 |
|---|---|---|
| `git add` | 可用 | 可用 |
| `git commit` | 可用 | 可用 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 可用 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 可用 |
| `gh pr create` | **被阻止**（网络） | 可用 |
| `git status/diff/log` | 可用 | 可用 |

其他发现：
- `spawn_agent` 子 agent **共享**父线程的文件系统（通过标记文件测试确认）
- 无论 worktree 从哪个分支启动，App 头部都会显示"Create branch"按钮
- App 原生收尾流程：Create branch → Commit 弹窗 → Commit and push / Commit and create PR
- `network_access = true` 配置在 macOS 上静默失效（issue #10390）

## 设计：只读环境检测

三个只读 git 命令可在无副作用的情况下检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

派生两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON`——agent 位于由其他东西（Codex App、Claude Code Agent 工具、之前的技能运行或用户）创建的 worktree 中
- **ON_DETACHED_HEAD：** `BRANCH` 为空——不存在命名分支

为什么用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在普通仓库中，两者解析到同一个 `.git` 目录
- 在链接 worktree 中，`git-dir` 是 `.git/worktrees/<name>`，而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等——避免了 `show-toplevel` 会产生误报
- 通过 `cd && pwd -P` 解析可处理相对路径问题（`git-common-dir` 在普通仓库中返回相对路径 `.git`，但在 worktree 中返回绝对路径）和符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接 worktree？ | 分离 HEAD？ | 环境 | 操作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整技能行为（不变） |
| 是 | 是 | Codex App worktree（workspace-write） | 跳过 worktree 创建；收尾时输出交接载荷 |
| 是 | 否 | Codex App（完全访问）或手动 worktree | 跳过 worktree 创建；完整收尾流程 |
| 否 | 是 | 不常见（手动分离 HEAD） | 正常创建 worktree；收尾时警告 |

## 变更

### 1. `using-git-worktrees/SKILL.md`——添加 Step 0（约 12 行）

在"Overview"和"Directory Selection Process"之间新增一节：

**Step 0：检查是否已在隔离工作区中**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过 worktree 创建。改为：
1. 跳到 Creation Steps 下的"Run Project Setup"小节——`npm install` 等是幂等的，为安全起见值得运行
2. 然后"Verify Clean Baseline"——运行测试
3. 报告分支状态：
   - 在分支上："已在隔离工作区 `<path>`，分支 `<name>`。测试通过。可以开始实现。"
   - 分离 HEAD："已在隔离工作区 `<path>`（分离 HEAD，外部管理）。测试通过。注意：收尾时需要创建分支。可以开始实现。"

如果 `GIT_DIR == GIT_COMMON`，继续完整的 worktree 创建流程（不变）。

Step 0 触发时跳过安全验证（.gitignore 检查）——对外部创建的 worktree 无关紧要。

更新 Integration 部分的"Called by"条目。将每个条目的描述从上下文特定文本改为："确保隔离工作区（创建或验证现有）"。例如，`subagent-driven-development` 条目从"REQUIRED: Set up isolated workspace before starting"改为"REQUIRED: Ensures isolated workspace (creates one or verifies existing)"。

**沙箱回退：** 如果 `GIT_DIR == GIT_COMMON` 且技能进入 Creation Steps，但 `git worktree add -b` 因权限错误失败（例如 Seatbelt 沙箱拒绝），将其视为晚检测到的受限环境。回退到 Step 0 的"已在工作区"行为——跳过创建，在当前目录运行设置和基线测试，并相应报告。

Step 0 报告后，停止。不要继续到 Directory Selection 或 Creation Steps。

**其余不变：** Directory Selection、Safety Verification、Creation Steps、Project Setup、Baseline Tests、Quick Reference、Common Mistakes、Red Flags。

### 2. `finishing-a-development-branch/SKILL.md`——添加 Step 1.5 + 清理守卫（约 20 行）

**Step 1.5：检测环境**（在 Step 1"Verify Tests"之后，Step 2"Determine Base Branch"之前）

运行检测命令。三条路径：

- **路径 A** 完全跳过 Step 2 和 Step 3（不需要基础分支或选项）。
- **路径 B 和 C** 正常经过 Step 2（Determine Base Branch）和 Step 3（Present Options）。

**路径 A——外部管理的 worktree + 分离 HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的收尾控件操作的是已提交的工作。

然后向用户呈现以下内容（不要呈现 4 选项菜单）：

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

分支名派生：如果有工单 ID 则使用（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个词转为 slug，否则省略建议。避免在分支名中包含敏感内容（漏洞描述、客户名称）。

跳到 Step 5（对外部管理的 worktree，清理是空操作）。

**路径 B——外部管理的 worktree + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

正常呈现 4 选项菜单。（Step 5 清理守卫会独立重新检测外部管理状态。）

**路径 C——正常环境**（`GIT_DIR == GIT_COMMON`）：

按现状呈现 4 选项菜单（不变）。

**Step 5 清理守卫：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 检测（不要依赖之前的技能输出——收尾技能可能在不同会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove`——宿主环境拥有此工作区。

否则，按现状检查和移除。注意：现有 Step 5 文本说"For Options 1, 2, 4"，但 Quick Reference 表和 Common Mistakes 部分说"Options 1 & 4 only"。新守卫添加在此现有逻辑之前，不改变哪些选项触发清理。

**其余不变：** 选项 1-4 逻辑、Quick Reference、Common Mistakes、Red Flags。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md`——各 1 行编辑

两个技能都有相同的 Integration 部分行。从：
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**其余不变：** 派发/审查循环、提示模板、模型选择、状态处理、红旗。

### 4. `codex-tools.md`——添加环境检测文档（约 15 行）

末尾新增两节：

**Environment Detection：**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.
```

**Codex App Finishing：**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

## 不变的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md`——子 agent 提示未改动
- `executing-plans/SKILL.md`——仅 1 行 Integration 描述变更（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md`——无 worktree 或收尾操作
- `.codex/INSTALL.md`——安装流程不变
- 4 选项收尾菜单——为 Claude Code 和 Codex CLI 完全保留
- 完整 worktree 创建流程——为非 worktree 环境完全保留
- 子 agent 派发/审查/迭代循环——不变（文件系统共享已确认）

## 范围摘要

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（Step 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（Step 1.5 + 清理守卫） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 个文件共约 50 行新增/变更。零新文件。零破坏性变更。

## 未来考虑

如果第三个技能需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件（方案 B）。现在不需要——只有 2 个技能使用。

## 测试计划

### 自动化（实现后在 Claude Code 中运行）

1. 普通仓库检测——断言 IN_LINKED_WORKTREE=false
2. 链接 worktree 检测——`git worktree add` 测试 worktree，断言 IN_LINKED_WORKTREE=true
3. 分离 HEAD 检测——`git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 收尾技能交接输出——在受限环境中验证交接消息（而非 4 选项菜单）
5. **Step 5 清理守卫**——创建链接 worktree（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入，运行 Step 5 清理检测（`GIT_DIR` 与 `GIT_COMMON`），断言它不会调用 `git worktree remove`。然后 `cd` 回主仓库，运行相同检测，断言它会调用 `git worktree remove`。之后清理测试 worktree。

### 手动 Codex App 测试（5 项）

1. Worktree 线程中的检测（workspace-write）——验证 GIT_DIR != GIT_COMMON，分支为空
2. Worktree 线程中的检测（完全访问）——相同检测，不同沙箱行为
3. 收尾技能交接格式——验证 agent 输出交接载荷，而非 4 选项菜单
4. 完整生命周期——检测 → 提交 → 收尾检测 → 正确行为 → 清理
5. **Local 线程中的沙箱回退**——启动 Codex App **Local 线程**（workspace-write 沙箱）。提示："Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change." 预检查：`git checkout -b test-sandbox-check` 应失败并报 `Operation not permitted`。预期：技能检测到 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到 Step 0"已在工作区"行为——运行设置、基线测试，从当前目录报告就绪。通过：agent 优雅恢复，无晦涩错误消息。失败：agent 打印原始 Seatbelt 错误、重试，或以令人困惑的输出放弃。

### 回归

- 现有 Claude Code 技能触发测试仍通过
- 现有 subagent-driven-development 集成测试仍通过
- 正常 Claude Code 会话：完整 worktree 创建 + 4 选项收尾仍正常工作