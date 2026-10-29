# Codex 效率修复 — 设计

日期：2026-07-30
状态：已由 Jesse 在会话中批准
分支：`codex-efficiency-fixes`，基于 `dev`

## 来源

- 评估活动收尾报告：`superpowers-autoresearch/reports/2026-07-codex-efficiency-campaign.md`
  （处理项表 §4；下文每项处理都有评分器和已测量的 `dev` 基线）。
- Codex 源码调研：`superpowers-autoresearch/docs/2026-07-29-codex-multiagent-v2-capabilities.md`
  （针对 Codex CLI 源码的 file:line 引用；为 T2、T3、T5 提供依据）。
- 已发布的实验记录：`superpowers-evals/docs/experiments/`。
- Drew 的衍生方案栈（PR #2036、#2035）是**证据，而非采纳的文本**：
  Jesse 希望在采纳其中任何一项之前更深入地研究这些修复；它们仅用于说明问题陈述。

## 目标

将 codex-efficiency 评估活动中五项证据充分的处理作为 superpowers 技能/文档变更交付，每项在切出 PR 之前，由活动评分器按其预注册标准评分。第二阶段（收尾处理项表中的其余所有内容）随后进行，每项都先以新的基线工作为前提。

## 范围决策（已与 Jesse 确定）

- **第一阶段 = 证据充分的五项**（下文 T1–T5）。第二阶段各项在修复交付前都需要一个失败的基线（区分规则：零结果不确定即停止）。
- **一个分支，每项处理一个 PR。** 开发和测试组在 `codex-efficiency-fixes` 上进行；当某项处理超过其标准时，将其连同评估证据切为针对 `dev` 的独立 PR。未经 Jesse 逐 PR 批准不得合并。
- **T4 以跨 harness 方式交付，并附带全局回归测试组**（Claude Code、Codex、Gemini），采用变体 C 形态：仪式随规模缩放，审批永不缩放。

## 五项处理

### T1. SDD 禁止 worker 发起审查

**证据：** 4 个语料库中 9/9 的深度 2 spawn 都是实现者发起的审查者；全部 9 个都是控制器本就会派发的审查的同任务重复。派发契约从未说明审查不是 worker 的职责；实现者提示中的"自我审查"在子代理可再 spawn 的 harness（Codex）上被具体化为一个审查者子代理。

**变更：**
- `skills/subagent-driven-development/implementer-prompt.md`：增加明确的"你不得派发子代理"条款——自我审查意味着阅读自己的 diff；所有审查派发由控制器负责；你 spawn 的审查者会重复流程已提供的审查。
- `skills/subagent-driven-development/SKILL.md`：在任务循环中增加一行派发契约，并增加一行 Red Flags："独立审查会加强我的报告" → 审查是控制器的下一步；你的审查者是重复席位。
- 措辞与 harness 无关（在子代理无法 spawn 的地方为空操作）。

**评分方式：** `score_e6.py`（按 spawner 角色统计深度 2 spawn、重复审查族）；同范围变体用 `score_e5.py`。
**基线：** 9/9 由 worker 发起，0 反例。
**标准：** 0 个 worker 发起的深度 2 spawn，且审查覆盖保持不变（每个任务仍恰好获得一次控制器派发的任务审查）。

### T2. 事件驱动等待

**证据：** 每个语料库中 60–78% 的 `wait_agent` 调用超时（dev 67.1%，spinout 60.2%）。源码调研：V2 等待是事件订阅，而非轮询——一次长等待的唤醒延迟与 10 秒轮询相同，但调用次数约为其 1/90；已完成子代理的 FINAL_ANSWER 会被推入父代理邮箱，并在下一次模型请求时被取用，完全无需等待。

**变更**（`skills/using-superpowers/references/codex-tools.md`）：
- 绝不用短超时轮询。
- 本地仍有工作时不要等待——子代理结果会通过邮箱随你的下一轮到达。
- 真正空闲时，发起一次 `wait_agent`，使用长 `timeout_ms`（900000+；harness 上限 3600000）。
- 说明 V2 注意事项：完成邮件携带 `trigger_turn=false`，不会唤醒空闲控制器——这正是 `wait_agent` 的唯一职责。

**评分方式：** `score_e7.py`（超时率、轮询间隔节奏、缓存重计费估算——重计费数字仍标注为估算）。
**基线：** dev 超时率 67.1%。
**标准：** 超时率 < 25%，且任务完成无损失。

### T3. codex-tools.md 更正

**证据：** 当前指南中有五处说法与 Codex 源码矛盾（均已在能力文档中 file:line 引用）：
1. `close_agent` 在 multi-agent V2 中不存在（仅 V1）。V2 会自动 LRU 驱逐已完成的子代理；不关闭无任何代价；`followup_task` 会透明地重新加载被驱逐的子代理。
2. 修复轮次始终可以通过 `followup_task` 恢复实现者——dev 中"如果你的 harness 无法向已 spawn 的代理发送另一条消息，则将每个修复轮次作为新的实现者派发"这一分支在 V2 上已失效。
3. 角色文件（`~/.codex/agents/**.toml`）确实会通过隔离 fork 上的 `agent_type` 附加到 spawn（0.145+）。
4. 全历史 fork 接受 `model`/`reasoning_effort` 覆盖；只有 `agent_type` 被拒绝。（出于上下文卫生原因，隔离 fork 仍是 SDD 指南，并准确陈述。）
5. 派发指南绝不能提及非 V2 模型预设——V2 spawn 允许列表仅限 v2 预设；其他会硬报错。

**变更：** 重写 `skills/using-superpowers/references/codex-tools.md` 的 multi-agent 段落，使其对版本诚实（在 V1 与 V2 行为不同处标注）。

**评分方式：** 源码引用（已验证）；共享测试组上无评分器回归。`score_e8.py` 保留为 V1/V2 schema 检测器，而非卫生评分器——不交付 `close_agent` 检查清单。

### T4. 头脑风暴三路径路由器（变体 C：始终审批）

**证据：** 微观——当前 HARD-GATE 文本将一项有界任务推向完整仪式 5/5，而 Z-null（无指导）和三路径路由器都能区分 5/5：绝对化措辞压制了模型本就能做出的区分。完整测试组——仪式量适度缩放（16.7 对 24.0 次工具调用，有界对架构），但双文档仪式（spec 文件 → plan 文件）在每次重复中都无条件运行。实测浪费是无条件的产物仪式，而非审批门。

**设计（变体 C）：** 三条路径缩放的是**产物**；每条路径在实现前都保留人工审批：
- **Spike**（可行性问题，明确一次性）：用 2–3 句话陈述问题和预期探测，获得点头，开始。无文档。发现以建议形式返回；构建的任何东西都保持标注为一次性。
- **Bounded**（对现有、已理解流程的界定良好的变更）：在聊天中呈现简短设计，获得批准，实现。无 spec 文件，不调用 writing-plans。
- **Architectural**（重构组件、新子系统、公共接口变更）：当前完整流程——spec 文档、审查、writing-plans。

**防护（全部随路由器交付）：**
- 分类要大声说出（"这看起来是有界的，所以我在这里呈现简短设计，而不是写 spec"），以便人工可以覆盖。
- 在两条路径之间犹豫时，选择较重的一条。
- 单向棘轮：路径中途发现的隐藏复杂性会升级路径；绝不在任务中途降级。
- 新增 Red Flags 行，针对将分类当作逃生舱（"我把它称为有界以跳过文档"）。

**变更**（`skills/brainstorming/SKILL.md`）：HARD-GATE 保留"审批前不实现"，并去掉"无论感知到的简单性如何"作为仪式驱动因素；反模式部分重新表述（罪过是跳过审批，而非跳过文档）；检查清单步骤 6–9 成为架构路径；流程图增加路由器；新增 Red Flags 行。这是精心调校的内容——编辑遵循 writing-skills 方法论，且仅在具备下文完整评估证据时才交付。

**评分方式（三层）：**
1. **微观**（`ceremony-path-micro.py`，改编）：变体 C 字面文本，加上活动从未测试的对抗性模糊简报（一个模式匹配为有界但隐藏公共接口变更的任务）。标准：spike/bounded/arch 可区分（每格 ≥4/5）；模糊简报升级为 FULL（≥4/5）；arch 从不降级（5/5）。
2. **Codex 仪式测试组：** 修复臂上的 `cx-ceremony-{spike,bounded,arch}`，各 3 次重复，`score_e4.py` 普查。标准：bounded 重复显示审批轮但零提交 spec 文件且零 writing-plans 仪式；arch 重复保留完整双文档流程；spike 重复保持最小。
3. **全局回归测试组：** 在 Claude Code 和 Gemini 上运行相同的三个仪式场景（rig 工作：这些场景目前受 codex 门控），各 3 次重复；外加在全部三个 harness 上的触发验收检查（"Let's make a react todo list" 自动触发头脑风暴进入完整/架构路径）。

### T5. 子代理发起的 spawn 上显式指定模型

**证据：** 在 CLI 0.146 上根 spawn 100% 显式指定模型（dev 14/14）；实际差距在深度 2——2/2 子代理发起的 spawn 省略了 `model`。源码调研：`model` 不带 `reasoning_effort` 会将 effort 重置为**模型**的默认值，而非父代理的。

**变更**（`skills/using-superpowers/references/codex-tools.md`）：
- 你发起的每次 spawn——包括作为子代理——都设置 `model` 和 `reasoning_effort`；点名 effort 重置陷阱。
- 建议在 `~/.codex/config.toml` 中设置 `[agents].default_subagent_model` 和 `[agents].default_subagent_reasoning_effort`，作为任何漏网之鱼的机器级兜底。

**评分方式：** 共享测试组上的 `score_e1.py`（按深度统计每次 spawn 的显式模型率）。
**基线：** 深度 2：0/2 显式。
**标准：** 每个深度的每次 spawn 都携带显式 model + effort。预注册注意事项：如果 T1 完全消除深度 2 spawn，T5 按根 spawn 回归评分（保持 100%）加文档正确性，并在深度 2 记录为零结果不确定——此时配置兜底成为实际机制。

## 评分计划

- **共享 SDD 测试组**承载 T1、T2、T5：`cx-sdd-small`，修复分支臂（`/tmp/sp-arm-fix`），跨两个容器通道 8 次重复。dev 基线已测量；不重跑基线。
- **T4 测试组**如上所列（微观 + codex 仪式 + 全局回归）。
- **预注册：** 每个测试组在运行**之前**，在 `superpowers-autoresearch/logs/2026-07-30-codex-efficiency-fixes.md` 中获得一条假设日志条目（预测、评分器、标准）。常设规则沿用：仅追加日志、对修复臂运行中的评分器匹配进行人工检查（非循环验证）、不提交原始 rollout、每个结论中正确性与成本并列。
- **归因：** 在一个合并分支上使用正交评分器；意外回归按处理提交二分。
- **预算：** 共享测试组约 $40，codex 仪式约 $40，全局回归约 $40–80，微观约 $5 → 第一阶段约 $150–200，来自活动 $1000 中剩余的约 $850。

## 流程

- 工作在 `codex-efficiency-fixes` worktree（基于 `dev` 分支）中进行；通过 subagent-driven-development 从书面计划执行。
- 技能文本变更遵循 writing-skills 方法论。
- 场景/rig 变更（为 Claude Code/Gemini 解除仪式场景门控、对抗性微观简报）按授权落地到 `superpowers-evals` main。
- 每项处理针对 `dev` 一个 PR，各自附带评估证据和标准标识块；仅在 Jesse 逐 PR 批准后合并。

## 第二阶段队列（基线优先；不在本计划任务中）

每项在修复交付前都需要一个失败的基线：
1. **派发路由 / 长会话漂移**——需要长会话引出 rig（在 CLI 0.146 上全新会话无法复现该病理）。Drew 的方案栈为处理形态提供参考。
2. **验证租约 / 证据回执**——需要先将子串感知的重复计数器加入 `score_e3.py`（当前基线 1/23 精确字符串对太弱）。
3. **补救上限**——小样本基线（2/3 重复）需要更多重复。
4. **跨任务竞态探测重新设计**——`score_e5.py` 的探测因设计权衡而零结果不确定；需要更强的探测。
5. **E5 D4 shell 命令解析器**——修复审查范围分类器无法解析复合命令；这是评分器工作，而非技能工作。

## 范围之外

- 采纳 Drew 的衍生方案栈（#2036/#2035）或其文本。
- RoboRev、Codex token 遥测（独立代码库）。
- `close_agent` 卫生检查清单（V2 无此工具——在活动中已关闭为不交付）。
- 除 T4 回归测试组之外的 Claude Code/Gemini 特定效率处理。