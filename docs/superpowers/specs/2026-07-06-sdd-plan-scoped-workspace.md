# SDD 计划作用域工作区 — 设计

- **日期：** 2026-07-06
- **状态：** 方向已批准（Jesse，2026-07-06）；本规范记录调查推荐的修复方案
- **问题归属：** subagent-driven-development 技能（`skills/subagent-driven-development/`）

## 问题

SDD 的持久进度工作区（`.superpowers/sdd/`，v6.0.0/v6.0.3 引入）没有计划标识，也没有生命周期终点。每个产物都以裸任务编号为键（`progress.md`、`task-N-brief.md`、`task-N-report.md`），而 SKILL.md 指示启动的控制器把找到的任何账本当作自己的进度：

> 技能启动时检查账本：
> `cat "$(git rev-parse --show-toplevel)/.superpowers/sdd/progress.md"`。其中列为完成的任务即 DONE——不要重新派发；从第一个未标记完成的任务继续。

在同一工作树中执行**后续计划**的新会话，会把前一个计划的账本读作自己的。按字面理解技能，它会跳过任务。工作区从不被删除，因此陈旧状态无限期存续并累积。

### 观察到的故障（serf 仓库，2026-06-22 → 2026-07-05）

- **跨计划冲突，临时绕过：** `cc-plugin-marketplaces` 工作树在三个计划间累积了 68 个文件。P2 控制器不得不发明 `progress-p2.md` 和 `p2-task-N-report.md` 来避开 P1 的账本；P2 的简报在默认路径上静默覆盖了 P1 的；还残留一个废弃的 `progress-p3.md` 桩文件。
- **Git 污染，三次：** SDD 临时文件被提交，需要两次清理提交（`8305e340d`、`c966261a5`）；今天 serf main 上有三个产物被跟踪，包括一份在另一台机器上编写的报告，现在会在每个新工作树中物化。后续计划的任务 1 报告覆盖了一份无关的已跟踪报告，留下永久的 `git status` 噪音。
- 自忽略的 `.gitignore` 仅在脚本运行时写入。手动追加账本的控制器（已观察到）从不创建它，而一旦文件被跟踪，gitignore 就无能为力。

### 根因

标识不存在于数据中；正确性依赖于没有触发器的清理。任何仅依赖计划结束清理的修复，恰恰会在账本存在所要应对的崩溃/压缩场景中失败。标识必须是结构性的。

## 设计

### 1. 每计划工作区目录（结构性标识）

工作区变为 `.superpowers/sdd/<plan-slug>/`，其中 `<plan-slug>` 是计划文件去掉 `.md` 扩展名后的基名（计划文件名已是带日期的 kebab-case，例如 `2026-07-04-plugin-marketplaces-p1-backend-core`）。不同计划的产物不再冲突；陈旧的同级目录是惰性的，因为没有指令指向它。

脚本接口（均在 `skills/subagent-driven-development/scripts/`）：

- `sdd-workspace PLAN_FILE` — 解析并创建 `<repo-root>/.superpowers/sdd/<plan-slug>/`，维护 `.superpowers/sdd/.gitignore`（父级，内容 `*`）处的自忽略 `.gitignore`，打印计划目录的绝对路径。缺少参数或计划文件不存在时报错（退出码 2）。剥离后 slug 必须非空。
- `task-brief PLAN_FILE N [OUTFILE]` — 签名不变；默认 OUTFILE 通过 `sdd-workspace PLAN_FILE` 移到 `<workspace>/task-N-brief.md`。
- `review-package PLAN_FILE BASE HEAD [OUTFILE]` — 新增 PLAN_FILE 作为第一个参数；默认 OUTFILE 移到 `<workspace>/review-<base7>..<head7>.diff`。

不为旧扁平布局提供兼容路径：脚本和 SKILL.md 在同一插件版本中一起发布，且没有其他东西调用这些脚本。（已明确确认：不做向后兼容处理。）

### 2. 账本标明其计划（为手写账本兜底）

账本仍为 `<workspace>/progress.md`。创建时，其第一行必须为：

```
# SDD ledger — plan: docs/superpowers/plans/<plan-file>.md
```

SKILL.md 的技能启动检查变为计划作用域，并携带一个以该可观察行为键的条件守卫，以正面方式表述（配方，而非禁令）：用 `sdd-workspace PLAN_FILE` 解析你的计划工作区，读取那里的 `progress.md`；计划行指向不同计划文件的账本属于另一个计划的进度——原地保留，使用你自己计划的工作区。这覆盖了不运行脚本而手写账本的控制器（在 serf ask_user 会话中观察到）以及旧扁平路径上的升级前残留。

守卫的确切措辞从属于评估结果（见评估）；仅为 RED 基线中实际观察到的失败添加计数器。

### 3. 工作区生命周期终点（卫生，非正确性）

当最终整分支审查干净且其修复波（若有）已合并——在移交给 `superpowers:finishing-a-development-branch` 之前——控制器删除其计划的工作区目录（`rm -rf "$WORKSPACE"`）。工作的记录是 git 历史；账本的职责（计划中途压缩恢复）已结束。同级目录从不触碰：崩溃或并行的计划拥有自己的目录，而有意停放的跨计划产物（观察到的模式：`WAVE1-HANDOFF.md`）直接位于 `.superpowers/sdd/` 下，不受任何计划清理影响。

### 4. SKILL.md 触点

- **Durable Progress** 节：通过 `sdd-workspace PLAN_FILE` 解析工作区；账本检查限定在计划自己的工作区；账本创建格式包含计划行；不匹配守卫；完成时删除；`git clean -fdx` 危险提示更新到新路径。
- **Handling Implementer Status / Constructing Reviewer Prompts / File Handoffs / Red Flags / Example Workflow**：将脚本调用更新为新签名（`review-package PLAN_FILE BASE HEAD`）及任何路径提及。`implementer-prompt.md` 和 `task-reviewer-prompt.md` 不含工作区路径（已验证），无需更改。
- 仅当 RED 基线显示结构性修复加守卫文本未能闭合的失败时，才添加 Red Flags。

## 范围外（有意）

- 不更改 `finishing-a-development-branch` 或任何其他技能。
- 除现有父级 `.gitignore` 外，不做 git 层面的守卫来防止提交 `.superpowers/`。
- 不对 serf 仓库做追溯清理（单独跟进）。
- 不做旧布局迁移或回退读取。

## 测试

### 确定性 shell 测试（`tests/claude-code/test-sdd-workspace.sh`，扩展）

- `sdd-workspace PLAN` 打印 `<root>/.superpowers/sdd/<slug>` 并创建它；无计划参数时报错；计划文件缺失时报错。
- 两个不同计划文件解析为两个不同目录；通过 `task-brief` 写入的产物落在各自计划的目录中。
- `review-package PLAN BASE HEAD` 写入计划目录下。
- 父级 `.gitignore` 自忽略：工作区对 `git status` 和 `git add -A` 不可见（现有断言，重新锚定）。
- 链接工作树区分性（现有断言，重新锚定）。
- 现有套件 `test-subagent-driven-development.sh` / `-integration.sh` 审计旧路径预期（初始 grep 未发现；审计无论如何是任务门禁）。

### 评估（writing-skills RED → GREEN，2026-07-06 重新界定）

压力场景作为全新 sonnet 子代理会话，针对临时目录中的夹具仓库运行（绝不在本工作树内），压缩恢复框架，每次重复手工评分；测量输出是控制器的恢复决策（无真实实现者派发）。

**迫使重新界定的 RED 结果（维护者决定，Jesse，2026-07-06）：** 最初假设的失败——控制器盲目将陈旧的外来账本采纳为自己的进度——**未**复现：25/25 次重复，跨三种框架（全新会话、可能已恢复、忠实压缩后恢复且技能的"信任账本"行生效），取证式地将账本引用的提交与 git 历史和计划文件交叉核对，拒绝了外来账本，并从任务 1 开始计划 B——每次恢复花费 6–13 次工具调用做跨计划取证。两次夹具迭代被消耗以诚实证明这一点（v1：伪造哈希被一眼驳回；v2：桩实现被裁定为虚假的"审查干净"记录——S2 对照两次都失败）。完整记录在已提交的评估文档中。

**重新界定的主张和门禁：**

- 该变更基于结构性记录（冲突、临时侧带名称、被覆盖的简报、git 污染——serf 仓库）加上测量的消歧成本发布，维护者明确签字代替 writing-skills 对 SKILL.md 文本的失败基线要求。
- **S1 GREEN（要求 5/5）：** 新作用域布局中存在陈旧的计划 A 工作区，加上旧扁平残留；计划 B 上恢复的控制器直接解析自己的计划作用域工作区并从任务 1 开始；每次重复的 `tool_uses` 对照 RED 基线（7/13/9/10/6）记录，作为成本增量。
- **S2 RED 对照（要求 ≥4/5）和 S2 GREEN（要求 5/5）** 在真实的 v3 夹具上（引用的提交真正实现了其任务的规范，轮换作者，分散时间戳）：合法的同计划恢复——任务 1–2 被识别，任务 3 被派发。这保护账本的原始目的；修复不得破坏它，对照验证夹具。

结果落在 `docs/superpowers/specs/2026-07-06-sdd-plan-scoped-workspace-eval-results.md` 并在 PR 中总结。

## 风险

- **不同目录中相同基名的不同计划之间的 slug 冲突**：接受；计划文件名按约定带日期前缀，相同基名在实践中意味着相同计划（此时恢复是期望行为）。
- **控制器完全跳过脚本**（全部手写）：账本计划行守卫是缓解措施；评估的 S1 测量文本是否真正生效。
- **工作区在崩溃后幸存，从头重新运行已完成的计划**：账本合法地属于同一计划；恢复而非重启是设计行为，`git log` 交叉核对（现有技能文本）覆盖分歧情况。