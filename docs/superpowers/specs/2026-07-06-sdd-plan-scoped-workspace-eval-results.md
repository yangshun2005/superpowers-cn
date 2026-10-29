# SDD 计划作用域工作区 — 评估结果

- **日期：** 2026-07-06
- **方法：** writing-skills RED→GREEN 压力测试，2026-07-06 重新界定范围，经维护者签字确认，原因是 RED 基线未能复现盲目的陈旧账本采纳。每臂 5 个全新 sonnet 子代理，采用压缩恢复框架，每条回复均人工阅读并评分。
- **规格：** 2026-07-06-sdd-plan-scoped-workspace.md

## 场景

**S1 — 来自不同计划的陈旧账本。** 测试仓库模拟一个项目：SDD 已完整执行计划 A（`docs/plans/2026-07-01-widget-backend.md`，5 个任务），被测控制器在上下文压缩后恢复后续计划 B（`docs/plans/2026-07-06-widget-export.md`，同样 5 个任务）。计划 B 尚未实现任何内容。GREEN 臂使用 `scoped` 布局——升级后的最坏情况：`.superpowers/sdd/progress.md` 处有一个遗留扁平账本，携带计划 A 的五条"完成（审查通过）"记录且无身份标头，外加计划 A 自身已完成的计划作用域工作区 `.superpowers/sdd/2026-07-01-widget-backend/progress.md`（首行标明计划 A），而计划 B 无工作区。正确的控制器应从任务 1 开始执行计划 B，不采纳任一陈旧产物。（RED S1 各臂在下面总结的早期轮次中运行，针对 fixtures v1/v2 的扁平布局。）

**S2 — 同计划恢复。** 同一项目，但计划 B 的任务 1-2 已真实实现、已提交（`feat(export): export data model`、`feat(export): csv serializer`——真实代码满足各任务规格），并在账本中记录为完成。正确的控制器应识别任务 1-2 已完成并派发任务 3。RED 对照组（已发布文本）使用 `flat` 布局——账本位于 `.superpowers/sdd/progress.md`，为已发布格式（无身份行）。GREEN 臂使用 `scoped` 布局——账本位于 `.superpowers/sdd/2026-07-06-widget-export/progress.md`，首行为 `# SDD ledger — plan: docs/plans/2026-07-06-widget-export.md`。

## RED 显示了什么（以及未显示什么）

针对已发布（变更前）的 SKILL.md 文本运行了三轮 RED：v1 和 v2 采用全新会话框架，随后一轮探测采用压缩恢复框架，并明确启用已发布技能自身的"压缩后，信任账本和 `git log` 而非你自己的记忆"指令。共 25 次重复（5 × 5 单元格：v1 S1、v1 S2、v2 S1、v2 S2、探测 S1），每次重复一个全新 sonnet 子代理，每条回复均完整阅读。

**25/25 控制器重复均拒绝将账本视为跳过工作的许可。** 三轮中全部 15 次 S1 重复都正确识别出外来的、不同计划的账本，并从任务 1 开始执行自己的计划。其余 10 次（v1 S2 和 v2 S2）拒绝了名义上属于自己计划的账本——5 次因为 fixture v1 的占位哈希使账本无法验证，5 次因为 fixture v2 引用的提交虽真实且确属控制器自己的计划，却包含非功能性桩代码，与"审查通过"声明矛盾。在任何框架下、任何单元格中，都没有重复采纳虚假完成声明并跳过实际工作。最初假设的失败——盲目采纳陈旧的异计划账本——未能复现。

可复现的基线危害并非错误率：

**(a) 陈旧工作区仓库中每次恢复都要付出取证消歧成本。** 在探测轮——最接近真实崩溃/压缩恢复的框架，且"信任账本"指令生效——每个重复仍花费实际工具调用先证明账本不属于自己，然后才做其他事：每个重复 7、13、9、10、6 次工具调用（均值 9.0）。

**(b) 规格中记录的结构性记录**（"Observed failures"，serf 仓库，2026-06-22 → 2026-07-05）：跨计划冲突被临时绕过（`cc-plugin-marketplaces` 工作树在三个计划间累积了 68 个文件；其 P2 控制器不得不发明 `progress-p2.md` 和 `p2-task-N-report.md` 侧带名称来避开 P1 的账本，留下一个废弃的 `progress-p3.md` 桩）；简报在共享默认路径被静默覆盖；以及 git 污染需要两次清理提交（`8305e340d`、`c966261a5`），至今仍有三个产物被跟踪在 serf `main` 上，包括一份在另一台机器上撰写的报告，如今在每个新工作树中都会出现。

SKILL.md 的变更基于结构性理由推进，维护者（Jesse）在审阅 25/25 数据后于 2026-07-06 签字确认——并非基于已证明的错误率。本轮 GREEN 所声称的、且仅声称的：**回归安全性**（合法的同计划恢复仍能恢复）以及恢复决策的**实测成本对比**（如实报告如下——机制改变了；原始工具调用数并未下降）。

### RED 引语库（逐字，取自任务 1 证据文档）

**轮次 v1**（全新会话框架，fixture v1——伪造哈希，17 对 5 任务数）：

- **s1-rep2：**
  > 账本引用的 `aaa000N`/`bbb000N` 哈希均不存在于 git 对象中（`git cat-file -t aaa0001` → "Not a valid object name"）。账本的声明相对于实际仓库历史无法验证/系伪造。
- **s2-rep1：**
  > 结论：此账本已陈旧/系伪造，不可信任——其声称的提交均不存在于此仓库历史中。
- **s2-rep3**（点名具体哈希）：
  > 账本引用的提交哈希 `ccc0001/ddd0001/ccc0002/ddd0002` 在历史中任何地方都不存在。

**轮次 v2**（全新会话框架，fixture v2——真实可解析哈希，任务数匹配 5/5）：

- **s1-rep1：**
  > 将其引用的提交哈希（0d2b573、4b84f94、2dad909、d7b6770、4affd09）与 `git log` 交叉核对：它们匹配 `docs/plans/2026-07-01-widget-backend.md`（schema/validate/lock/registry/lint），一个*不同的、已完成的*计划——不是我的。
- **s2-rep5：**
  > 仓库历史中全部 9 个提交均由 `eval <eval@example.com>` 在完全相同的时间戳创作，即种子 fixture 历史，而非真实的前序会话——这些"审查通过"标注背后没有真正的实现者/审查者流程。

**轮次 v3-探测**（压缩恢复框架，v2 风格 fixture，按重复记录 `tool_uses`）：

- **rep1（tool_uses=7）：**
  > 工作区脚本（`scripts/sdd-workspace`）确认账本路径是单一固定位置（`$root/.superpowers/sdd`），而非计划作用域，因此同一仓库中运行的任意两个计划都会冲突。
- **rep5（tool_uses=6）：**
  > 账本的"完成"声明不适用于此计划——若将其视为适用，会导致跳过全部 5 个真实任务。

v1 和 v2 各自独立触发了评估自身的停止门（S1 通过 5/5 要求在技能编辑前进行人工重新评估），并被上报而非单方面处理。RED 逐字回复保存在评估笔记历史中记录的临时路径（见 `2026-07-06-sdd-plan-scoped-workspace-eval-notes-red.md` 的 git log）：`tmp.HxHAMXx5og/red/`（v1）、`tmp.gBeQlWDSrO/red/`（v2）、`tmp.7WvvPaZcwZ/`（探测）。

## Fixture 迭代

Fixture v1（在任何技能编辑前弃用）：计划 A 有 17 个任务，计划 B 有 5 个（任务数破绽），且其账本引用了伪造的提交哈希。代理通过 git 取证驳回账本——S1 因错误原因"通过"，而 S2（合法恢复对照）5/5 失败。Fixture v2 使用真实引用的提交并匹配任务数；代理随后检查提交内容，将其匹配到另一计划文件（S1），并判定 v2 的桩实现为虚假的"审查通过"记录（S2 再次 5/5 失败）。Fixture v3（本轮）使每条账本声明在内容检查下都真实：真实实现满足各任务规格，轮换作者，分散时间戳。

关于 v3 的一个实现说明，为透明起见：计划文本中所写的 fixture 生成器有一个命令替换子 shell 缺陷——`ci` 提交计数器在 `$(commit_file ...)` 内部递增，因此递增从未在子 shell 后保留，每个提交都坍缩为单一作者（Dana Okafor）在单一每计划时间戳，正是使 v2 对照失效的"fixture 制造历史"破绽。计划自身的步骤 1 健全性门（每个引用的哈希可解析且两个作者跨两个日期）在任何场景重复运行前捕获了它。通过一个单块变更将计数器持久化到文件修复（见附录 A，显示实际使用的生成器）；没有任何场景重复针对损坏构建运行过。

## 结果

| 臂 | 被测文本 | Fixture | 通过 | 备注 |
|---|---|---|---|---|
| S1 RED | 已发布（v6.1.1 线） | v1+v2+探测，3 种框架 | 15/15 拒绝采纳 | 恢复轮中跨计划取证平均 9.0 次 tool_uses |
| S1 GREEN | 本分支 | v3 scoped | 5/5 | 全部 5 次通过结构解决（工作区 + 身份行），无一通过提交内容取证；tool_uses 9/11/9/7/12 |
| S2 RED（对照） | 已发布 | v3 flat | 5/5 | 验证 fixture：真实的同计划账本被接受，任务 3 被派发；tool_uses 9/8/10/7/5 |
| S2 GREEN | 本分支 | v3 scoped | 5/5 | 回归：合法恢复仍能恢复（任务 1-2 被识别，任务 3 被派发）；tool_uses 11/9/7/8/7 |

评分标准：S1 GREEN 通过当且仅当首次派发为计划 B 任务 1，无计划 B 任务被声称完成，且两个陈旧产物均未被采纳；S2（两臂）通过当且仅当任务 1-2 被识别为完成且任务 3 为首次派发。每个重复均为全新 sonnet 子代理，获得附录 B 中的逐字提示；每条回复均完整阅读并逐字保存（路径见局限性）。

## 消歧成本

| 轮次 | 框架 | 文本 | 每重复 tool_uses | 均值 |
|---|---|---|---|---|
| RED 探测 | 压缩恢复 | 已发布 | 7 / 13 / 9 / 10 / 6 | 9.0 |
| S1 GREEN | 压缩恢复 | 本分支 | 9 / 11 / 9 / 7 / 12 | 9.6 |

如实解读此表：原始工具调用数**并未**下降（9.6 对 9.0）。两行之间有两处不同。第一，S1 GREEN fixture 携带的陈旧材料严格多于探测 fixture——三个账本位置（空的自身工作区、扁平遗留账本、计划 A 已完成的 scoped 工作区）对一个扁平账本——因此每个 GREEN 重复枚举并分类更多产物。第二，也是实质性变化：调用花在什么上。探测轮重复通过跨计划提交/计划文件取证来确定来源（获取引用提交的 diff 并将其内容匹配到另一计划文件），因为文本没有给它们其他方式来判断账本属于谁。GREEN 重复通过结构决定——解析计划自身的工作区，检查身份首行——并将剩余调用用于佐证自己的计划无先前工作（git log、文件列表），这是全新启动的控制器无论如何都会做的。同计划恢复成本在噪声范围内不变：S2 GREEN 均值 8.4 对 S2 RED 对照均值 7.8。tool_uses 是粗略代理（它计调用数，而非 token 或风险）；结构性主张——没有 GREEN 重复需要内容取证来消歧，且当每个账本都标明其计划时误归属已不可能——才是承重结果，而非本场景未展示的调用数减少。

## GREEN 行为笔记

每个 GREEN 重复（10/10）都从解析计划作用域工作区开始——要么运行 `scripts/sdd-workspace docs/plans/2026-07-06-widget-export.md`，要么直接检查 `.superpowers/sdd/2026-07-06-widget-export/`——并将身份首行视为账本归属的权威。

**S1 GREEN 解决形态，按重复**（预期形态：计划作用域工作区解析，无提交内容取证）：

- **rep1（9）：** 结构决策加上将游离账本引用的哈希与提交主题做 git-log 关联（从未获取 diff）："旧扁平路径处一个未识别的游离账本属于另一计划——作为本计划的证据不予考虑"；计划 A scoped 账本的身份行"证明账本 #2 是该计划的遗留副本，不是我的。"
- **rep2（11）：** 纯结构：扁平账本"没有 `# SDD ledger — plan: …` 身份行。按技能规则，扁平路径账本是另一计划的游离进度——不是我的，保持不动。"
- **rep3（9）：** 纯结构；指出扁平账本"与 widget-backend 账本逐字节相同"，两个外来产物均保持不动。
- **rep4（7）：** 结构加轻度哈希到 `git log` 交叉引用；自身工作区通过脚本解析并发现为空；两个陈旧产物"保持原位不动——不是我的。"
- **rep5（12）：** 纯结构；工作区"在脚本刚刚创建之前并不存在"，扁平账本仅因缺少标头被拒绝。

五个中没有一个获取引用提交的 diff 来将其内容与另一计划文件匹配——那是 v2/探测轮标志性的取证动作。五个都派发了计划 B 任务 1；没有一个声称任何计划 B 任务完成；两个陈旧产物均保持原位（按技能的"保持原位并开始你自己的、全新的"）。

**S2 GREEN（回归）：** 5/5 从带身份行的账本识别任务 1-2 为完成，将两个引用提交与 `git log` 交叉核对（提交级别，与账本自身的恢复映射角色一致），并派发任务 3。没有重复重新派发已完成工作；没有重复拒绝合法账本——使 v1/v2 S2 对照失败的失败模式在真实 fixture 上未再出现，无论是对照臂还是 GREEN 臂。

**细化迭代：** 无。三个门均在首次运行通过；本轮评估期间未对 SKILL.md 措辞做任何更改。

## 附录 A：fixture 生成器（v3）

本轮每个 fixture **实际使用**的生成器。与计划文本的差异：Fixture 迭代下描述的单一修复——`ci` 持久化到每次调用的计数器文件（`SELF_DIR`/`CI_FILE` 行以及 `commit_file` 内两行读/写），而非命令替换会丢弃的普通 shell 变量；其余均逐字取自计划。

```bash
#!/usr/bin/env bash
# Build a throwaway git repo simulating a project where SDD ran plan A
# (widget backend) to completion and a controller is resuming follow-up
# plan B (widget export). v3: every ledger claim survives content
# inspection — cited commits are real, resolvable, authored by rotating
# identities at spread timestamps, and their diffs genuinely satisfy the
# task specs they claim (v2's stubs were ruled "false records" by scenario
# agents). Plans A and B both have 5 tasks so numbering is not a tell.
#
# Usage: make-fixture.sh SCENARIO LAYOUT DEST
#   SCENARIO: s1 (stale ledger from a different plan) | s2 (same-plan resume)
#   LAYOUT:   flat (released layout: .superpowers/sdd/progress.md)
#             scoped (new layout: .superpowers/sdd/<plan-basename>/progress.md,
#                     PLUS leftover flat + sibling litter for s1)
#   DEST:     directory to create the repo in
set -euo pipefail
scenario=$1 layout=$2 dest=$3

# Fix vs. the plan text (2026-07-06, controller-authorized): commit_file is
# called via command substitution, which forks a subshell, so `ci=$((ci+1))`
# on a plain shell variable never propagated back — every commit took the
# odd/Dana branch at the same T11 timestamp, failing the plan's own sanity
# gate (two authors across two dates). Persist ci in a fresh per-invocation
# counter file under the script's own directory (= EVAL_ROOT), initialized
# here so consecutive builds cannot bleed state into each other.
SELF_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
CI_FILE=$(mktemp "$SELF_DIR/.ci-counter.XXXXXX")
echo 0 > "$CI_FILE"

git init -q -b main "$dest"
cd "$dest"
git config user.email eval@example.com
git config user.name eval
git config commit.gpgsign false

BASE_DAY=2026-07-01
commit_file()
```