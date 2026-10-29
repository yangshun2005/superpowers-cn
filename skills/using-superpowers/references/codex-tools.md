## Subagent 调度需要多智能体支持

在 Codex 配置（`~/.codex/config.toml`）中添加：

```toml
[features]
multi_agent = true
```

这会启用 `dispatching-parallel-agents` 和 `subagent-driven-development` 等技能所用的多智能体工具。你能获得哪些工具，取决于模型预设选中的多智能体版本（当前预设运行 V2，旧预设运行 V1）。当实际工具列表与任何表格（包括本表）不一致时，以实际工具列表为准。

- **派生：** 用 `spawn_agent {fork_turns: "none"}` 给子智能体一个干净的上下文；默认的 `"all"` 会把你的完整对话记录复制给子智能体。在 Codex 0.145+ 上，`~/.codex/agents/` 下的角色文件通过 `agent_type` 附加到隔离 Fork。全历史 Fork 接受 `model` 和 `reasoning_effort` 覆盖（仅 `agent_type` 在此被拒绝）——隔离 Fork 是 SDD 的默认选择，是为了上下文卫生，而非因为覆盖需要它们。
- **修复轮次：** 用 `followup_task` 恢复实现者——它会投递你的消息、触发一轮对话，并透明地重新加载被 harness 驱逐的子智能体。绝不要因为"已派生的智能体无法再次发消息"这一猜测而重新派发一个新的实现者；在 V2 上它总是可以。
- **生命周期：** V2 没有 `close_agent`。已完成的子智能体在需要槽位时会被自动驱逐；不关闭它们没有代价。只有 V1 会话有 `close_agent`——在那里，评审返回后关闭评审者，每个实现者的任务评审通过后关闭该实现者。
- **模型名称：** 绝不要把技能、表格或旧会话中的模型名称直接复制进 `spawn_agent`，必须先对照当前的派生允许列表检查——V2 只接受支持 V2 的预设，其余会硬报错。

## 等待子智能体

`wait_agent` 是事件订阅，不是轮询：长等待会在子智能体产生邮箱活动的瞬间唤醒，延迟与短等待相同。短超时轮询毫无收益，且每次轮询都消耗一次工具调用——以及一次上下文重新计费。在实测会话中，约三分之二的等待调用都是超时的短轮询。

- 当你还有本地工作时，完全不要等待。已完成子智能体的最终答案会推送到你的邮箱，并在你下一轮对话时送达。
- 当你确实空闲且有子智能体未完成时，分段等待：`wait_agent` 配合 `timeout_ms` 300000-600000（5-10 分钟）。每段结束后——无论唤醒还是超时——发一行状态、运行 `list_agents`，并追查任何已完成但未汇报的子智能体。绝不要堆叠短于五分钟的轮询；事件订阅唤醒有界分段的快慢与短轮询相同。
- 完成邮件无法唤醒空闲的控制器（它投递时不触发一轮对话）；覆盖这个空闲窗口正是 `wait_agent` 的唯一职责。某段等待超时且无活动，是提醒你去核对，而不是缩短下一段。

## 派生时的模型路由

你发出的每个 `spawn_agent`——包括你自己作为被派生子智能体执行扇出时——都要按所执行技能的模型选择规则，显式设置 `model` 和 `reasoning_effort`。只设置 `model` 是个陷阱：子智能体的 effort 会静默重置为该模型的默认值，而非你的值。

请人类伙伴在 `~/.codex/config.toml` 中添加机器级兜底，让任何漏网的派生仍路由到刻意选择的层级，而不是静默继承会话中最贵的模型：

```toml
[agents]
default_subagent_model = "<a mid-tier model from your spawn allowlist>"
default_subagent_reasoning_effort = "medium"
```

## 环境检测

创建 worktree 或收尾分支的技能，应先以只读 git 命令检测环境再继续：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接 worktree 中（跳过创建）
- `BRANCH` 为空 → 分离 HEAD（无法从沙箱创建分支/推送/发起 PR）

各技能如何使用这些信号，见 `using-git-worktrees` 第 0 步和 `finishing-a-development-branch` 第 1 步。

## Codex App 收尾

当沙箱阻止分支/推送操作（外部管理的 worktree 中处于分离 HEAD）时，智能体提交所有工作，并告知用户使用 App 的原生控件：

- **"Create branch"** —— 命名分支，然后通过 App UI 提交/推送/发起 PR
- **"Hand off to local"** —— 将工作转移到用户的本地检出

智能体仍可运行测试、暂存文件，并输出建议的分支名、提交信息和 PR 描述供用户复制。