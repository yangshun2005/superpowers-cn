# 诊断 Superpowers 会话 — 设计

日期：2026-08-27
状态：已由 Jesse 在会话中批准；规格待评审
分支：`diagnosing-superpowers`，基于 `dev`

## 目标

一个核心技能 `diagnosing-superpowers`，供用户在 superpowers 会话出问题时调用。它与用户一起锁定问题，检查磁盘上的会话记录，并带证据报告发生了什么。应请求时，它导出一个脱敏包，供远程 agent 判断 superpowers 本身是否需要改动；它还能查找本机其他表现出相同行为的会话。

该技能只报告，从不诊断 superpowers。推测 superpowers 的 bug 或提出改动建议是远程分诊者的职责；被问到时，技能会说明这一点。

## 范围决策（已与 Jesse 确定）

- **v1 为纯文字技能。** 不附带脚本。由模型完成工作，大量使用子 agent。若文字版验证了形态，后续可加确定性工具。
- **Harness 覆盖。** 只有对照磁盘文件验证过的格式才提供字段级细节的参考文档：Claude Code 和 Codex。其他 harness 一律走发现流程。运行中的 harness 应当了解自己的会话存储；技能要求它使用这一知识，并如实说明能读到什么、读不到什么。不虚构格式。
- **先做问题收集。** 技能开场先问用户想诊断什么，并与用户一起把问题收敛为具体的问题陈述。后续扫描都服务于该陈述。
- **质量按过程证据判断**：对照会话约定的计划（设计、计划、验收标准、spec/plan 文件），以及记录能证明的内容（跑过的测试、声明背后的验证、提交与声明是否一致、评审反馈是否处理）。不是对最终 diff 做代码评审。
- **脱敏级别由用户决定。** 技能会询问，并告知用户：若是为 superpowers 报 bug，信息越多越可能得到帮助。
- **精确记录 superpowers 身份**：实际加载的安装根目录、版本、若为检出则记录 git sha，以及会话读取或注入的每个技能文件的 sha1。
- **技能触发是一等分析维度**：何时触发了什么、响应什么请求、技能自身的触发描述匹配但未触发或触发过晚的位置。

## 技能布局

```
skills/diagnosing-superpowers/
  SKILL.md
  references/
    claude-code-sessions.md
    codex-sessions.md
    other-harnesses.md
    context-safety.md
    github-issues.md
  prompts/
    analyst-common.md
    skill-timeline.md
    plan-adherence.md
    repeated-work.md
    stumbles.md
    quality-evidence.md
    request-conflicts.md
    cost-and-time.md
    scrub.md
    scrub-audit.md
    similar-session.md
  templates/
    case.md
    report.md
    bundle-README.md
    issue.md
tests/diagnosing-superpowers/
  test-skill-structure.sh
```

与 `subagent-driven-development` 结构相同：精简的 SKILL.md 承载工作流、硬性规则和 Red Flags；每个子 agent 任务一个文件，使每个子 agent 只读一个 prompt；参考文件仅在 harness 匹配时加载。

### SKILL.md frontmatter

```
name: diagnosing-superpowers
description: Use when a superpowers session went wrong and the user wants
  to know why — repeated work, ignored plans, stumbles, poor results, a
  skill that didn't fire — or wants to build a bug report for the
  superpowers maintainers, for the current session or a past one
  identified by id or path, on any harness.
```

只写触发条件，不写工作流摘要（见 `writing-skills` 的 Skill Discovery Optimization）。SKILL.md 控制在 1,000 词以内（结构测试会强制检查；本仓库的流程类技能为 350–4,800 词，而本技能有七步工作流）：工作流、硬性规则、Red Flags 和指引。其余内容都在 prompt、参考和模板文件中。

## 工作流

技能运行时，每一步都是一个待办项。

### 1. 问题收集

一次问一个问题，直到问题具体化：哪个会话、用户期望什么、实际发生了什么、最早在哪里注意到。抱怨通常很模糊（"太慢了"、"为什么多做这些活？"、"为什么这么贵？"、"它到底在干什么？"）；收集环节把每条转成明确陈述，写明会话、已知时的轮次范围，以及用户关心的可观测指标（实际耗时、token、重复动作、某个具体意外动作）。把达成一致的陈述写入 case 文件（见下）。若用户说目标是为 superpowers 报 bug，此时记下；导出时技能会提一次可按需提供 bundle。

### 2. 定位

把用户提到的每个会话解析为磁盘上的确切路径。

- **当前会话。** 模型使用其 harness 对记录写入位置的自身知识。对 Claude Code 和 Codex，参考文件给出目录布局、如何选出当前会话（本 cwd 下最近修改的文件，并用首条用户消息匹配确认）、子 agent 记录所在位置，以及哪些字段承载模型、harness 版本、技能/插件归属、压缩和错误。对其他 harness，`other-harnesses.md` 要求：找到你的会话存储，说明发现了什么以及置信度；若找不到，如实说明并向用户要路径。
- **历史会话。** 用户给出 id、路径、日期加描述，或"发生 X 的那次"。解析为确切路径，并在分析前引用首条 prompt 和时间戳与用户确认身份。
- **子 agent。** 枚举属于该会话的每个子 agent/侧链记录，视为会话的一部分。
- **进行中的会话。** "它现在在干什么"意味着会话可能仍在运行、文件正在写入。读取现有内容，记录读取时的行数和 mtime，并在覆盖说明中注明会话仍在进行。
- **主机与 superpowers 身份。** 记录操作系统及版本；harness 及版本；出现过的每个模型 id；会话实际加载的 superpowers 安装根目录（marketplace 缓存与开发检出可能不同）、manifest 中的版本、若为检出则记录 git sha；会话读取或注入的每个技能文件的 sha1，按文件当前状态计算，若文件 mtime 晚于会话则标记，因为哈希可能与会话当时所见不一致；配置的其他插件、扩展和 MCP server；存在的指令文件（CLAUDE.md、AGENTS.md、GEMINI.md 等）仅列路径。
- **查看过的一切都要报告**：每个会话 id 和路径，包括因不匹配被排除的候选及原因。

工作区为 `~/.superpowers/diagnosing-superpowers/<session-id>/`（位于主目录，因此不会落入项目树或提交）。技能在创建后立即在聊天中打印该路径，并在报告中再次给出。其中的 `case.md` 保存问题陈述、解析出的路径、身份事实和上下文安全规则。每个子 agent 都会拿到其路径。

### 3. 分诊

控制器自行阅读所报问题附近的记录区域（遵守上下文安全规则），形成初步判断。然后并行派发分析子 agent，每个维度一个，各自拿到 case 文件路径和对应 prompt 文件。长会话中，控制器把一个维度按轮次范围拆分并合并结果。

子 agent 以统一格式返回发现：

```
- finding: <one sentence, what happened>
  evidence: <path:line> — "<short quote>"
  turns: <first>–<last>
  confidence: high | medium | low
```

维度及各自查找的内容：

- **技能时间线。** 按人类轮次：调用了哪些技能和插件（有 harness 归属字段则用，否则看对 `SKILL.md` 文件的读取）、调用前是什么请求、技能触发描述匹配请求但未触发的轮次，以及触发过晚的情况。还包括用到的每个非 superpowers 插件、技能、agent 或 MCP 工具及其位置。
- **计划遵循。** 还原会话约定的计划、spec、设计或待办列表；把每一步映射到实际发生的事；标记跳过、重排、静默改动或凭空新增的步骤。标记压缩和恢复点，因为之后计划漂移很常见。
- **重复工作。** 同一文件被多次读取或编辑、同一命令重跑、同一子 agent 任务重复派发、已做出的决策被重新推导。
- **磕绊。** 工具错误、失败命令、重试、回退的编辑、走回头路、用户纠正、权限拒绝、hook 失败、API 错误、崩溃、上下文溢出。
- **质量证据。** 跑过的测试及结果；"完成"、"已验证"、"通过"等声明，以及验证输出是否先于这些声明；提交与声明是否一致；评审反馈是被处理还是敷衍。
- **请求冲突。** 跨轮次相互矛盾的用户指令、与 CLAUDE.md/AGENTS.md 冲突的指令、被要求忽略的请求。只有人类输入的 prompt 才算用户指令。
- **成本与时间。** 按人类轮次、按子 agent、按工具统计 token（输入、输出、缓存）和实际耗时；最大的单条工具结果；压缩次数及位置；事件之间的空闲间隔；占总量大头的轮次。Claude Code 每条消息带 `usage`；Codex 发出 `token_count` 事件。

控制器对照自己的阅读核对发现，丢弃没有 `path:line` 的条目，然后写报告。

### 4. 报告

写入 `~/.superpowers/diagnosing-superpowers/<session-id>/report.md`，同时在聊天中展示。章节顺序固定，便于远程分诊者依赖：

1. **问题陈述**，与收集阶段达成的一致。
2. **分诊结论。** 证据表明所报问题附近发生了什么，用文字叙述，附 `path:line` 引用和明确的置信度。不对 superpowers 做根因断言，也不提改进建议。
3. **环境。** 第 2 步记录的全部内容：主机、harness、模型、superpowers 身份及技能文件哈希表、其他插件和 MCP server、存在的指令文件。
4. **检查过的会话。** 每个 id 和绝对路径，包括子 agent 记录，以及被排除的候选及原因。
5. **时间线。** 按人类轮次：请求（一行）、触发的技能、派发的子 agent、压缩/错误/恢复事件。
6. **发现。** 每个维度一个小节（技能时间线、计划遵循、重复工作、磕绊、质量证据、请求冲突、成本与时间），采用上述发现格式。空维度写"未发现"以及检查了什么。
7. **Superpowers 相关性。** 取其一：*未显示*、*可能*、*很可能*，并附支持该判断的证据行。这是技能唯一表达对 superpowers 看法的地方，且止于相关性：不指认缺陷，不提改动。
8. **覆盖说明。** 未读的内容（范围、文件）及原因、哪些 harness 特性不可用、用户应复核的事项。

语言规则：可以说"证据表明 X"；不能说"superpowers 应该……"或"这是技能 Y 的 bug"。给用户的建议（"下次这样做"）也不行：技能只报告所见。若用户问该修什么，技能指向 GitHub issue 步骤，并提出可导出 bundle。

### 4a. GitHub issue

当报告第 7 节为*可能*或*很可能*，或用户要求时运行。

1. **搜索** `obra/superpowers` 的开放和已关闭 issue，按症状搜：技能名、错误字符串、问题陈述中的可观测指标。有 `gh` 就用；否则用 curl 调公开搜索 API（`https://api.github.com/search/issues`）；再不行就给用户搜索 URL 并停止。
2. **展示匹配项**（编号、标题、状态、一行说明为何匹配），建议用户把报告或 bundle 补充到最接近的一条。
3. **若无匹配**，按 `templates/issue.md` 起草 issue：提交者是谁、用户想要什么、用用户自己的话写的摘要、问题陈述、分诊结论、环境章节（包括本仓库要求每个 issue 披露的模型 / harness / harness 版本 / 已安装插件）、检查过的会话，以及任何 bundle 的脱敏级别。展示确切文本；仅在用户批准且摘要保留其原话后，才用 `gh issue create` 创建，并加 `bug` 和 `automated-issue-report` 标签。GitHub 会静默丢弃无推送权限提交者的标签，因此模板页脚才是技能提交 issue 的持久标记。`gh` 无法附加文件，技能需告知用户 bundle 路径，让其通过网页 UI 附加。没有 `gh` 时，技能给出基于 `bug_report.md` 模板的预填新建 issue 链接，该模板会为任何提交者加上 `bug` 标签；GitHub 对该 URL 限制约 8,000 字符。
4. 未经用户批准确切文本，不向任何地方发布任何内容。

### 5. 导出（按需）

仅在用户要求时运行。技能绝不主动构建 bundle：bundle 是用户自己的会话数据，为他人打包，未经请求就递上一份会显得冒犯。若用户在收集阶段说过目标是报 bug，技能提一次可按需提供脱敏 bundle，然后等待。交付压缩包时，技能说明其中包含什么、脱敏替换了什么、自动脱敏可能遗漏内容，以及用户应在分享前逐个文件检查。bundle 写入 `~/.superpowers/diagnosing-superpowers/<session-id>/bundle/`，压缩包放在其旁边。

1. **询问脱敏级别。** 说明：若是为 superpowers 报 bug，提供的信息越多，维护者越可能帮上忙。级别：
   - *skeleton*：不含工具结果正文；
   - *evidence*：仅发现中引用事件的工具结果正文；
   - *full*：所有工具结果正文，经脱敏。
   技能建议默认 *evidence*。
2. **构建 bundle**，包含以下文件：
   - `README.md`：这是什么、脱敏级别、如何阅读 bundle，以及分诊者的任务（判断 superpowers 是否有责任、该改什么），并注明 bundle 刻意不含修复建议；
   - `report.md`、`case.md`、`environment.json`、`timeline.md`；
   - `findings/`：每个维度一个文件；
   - `transcripts/`：每个检查过的会话按所选级别生成逐轮精简呈现，绝不放原始 JSONL；
   - `scrub-log.md`。
3. **脱敏**由子 agent 逐文件执行：邮箱；人名，替换为角色占位符；账户和组织 UUID；任何像 API key、token 或密码的内容；主机名和 IP；主目录下的绝对路径改写为 `~`；仓库名和 URL（若用户说明仓库是公开的则保留）；用户指明为专有的任何内容。
   每次替换使用稳定占位符（`<EMAIL-1>`、`<PATH-3>`），以保留交叉引用。脱敏日志只列占位符 → 类别，绝不列原值。
4. **脱敏审计**由第二个独立子 agent 执行，其唯一职责是找出第一个遗漏的内容。重复脱敏和审计，直到审计无发现。
5. **用户复核关卡。** 展示脱敏日志和文件列表，请用户抽查，之后才创建压缩包（`zip -r` 或 `tar -czf`，取决于 shell 有什么）。报告压缩包路径。技能绝不上传到任何地方。

### 6. 相似会话（按需）

1. 把确认的发现转成**签名**：具体、可 grep 的标记（技能名加观察到的序列、错误字符串、重复命令模式、"压缩后计划偏离"）、日期窗口和范围（本项目、本机所有项目、单个 harness 或全部）。
2. 发现以元数据优先：按 mtime 和大小列出候选会话文件，提取标记所在行号，只保留有命中的会话。适用上下文安全规则。
3. 候选会话连同签名和 case 文件并行交给子 agent；各自返回 yes / no / partial 及 `path:line` 证据。
4. 结果作为**相似会话**追加到报告：id、路径、日期、harness、匹配了什么、没匹配什么。匹配项可按相同脱敏级别、经相同脱敏、审计和用户关卡加入 bundle。

仅限本机。技能绝不访问他人的会话或远程存储。

## 硬性规则（SKILL.md 及每个子 agent prompt）

- **上下文安全。** 单条记录行可能含 100k+ token（工具结果、图片、hook 载荷）。绝不对记录文件 `cat` 或 `grep` 内容。先取计数和行号（`grep -n … | cut -d: -f1`），再从特定行提取小字段（有 `jq` 用 `jq`，否则 `sed -n Np | cut -c1-500` 或 python3/node 单行命令）。做任何事之前先检查文件大小和行数。
- **只读。** 会话文件绝不修改、移动或删除。
- **给子 agent 确切路径。** "当前会话"对子 agent 而言是父会话，因此控制器始终给子 agent 确切的路径和 id，绝不给描述。
- **仅人类 prompt。** Hook 输出、`<system-reminder>` 块和工具结果都以 user 角色到达。只有人类输入的 prompt 才用于轮次编号和请求冲突发现。在子 agent 记录中，"user"是父 agent。
- **有证据，否则不写。** 每条发现都引用 `path:line`。无引用的发现在核对时丢弃。
- **不诊断 superpowers。** 技能描述发生了什么，不说 superpowers 哪里有问题或该改什么。
- **导出前用户关卡。** 用户看过脱敏日志和文件列表之前，不创建压缩包。
- **发布前用户关卡。** 用户批准确切文本之前，不创建 issue 或评论。

## Red Flags（SKILL.md 表格）

这些行是设计阶段的假设。最终表格根据 RED 阶段（见下）观察到的合理化说辞构建；基线运行中从未出现的行删除，出现的行改写以匹配 agent 实际说法。

| 想法 | 现实 |
|---------|---------|
| "问题很明显，跳过收集" | 用户的问题陈述界定下游一切。要问。 |
| "直接 grep 记录就行" | 一行就可能占满你的整个上下文。先取行号，再取字段。 |
| "这显然是技能 X 的 bug" | 不由你判断。报告证据