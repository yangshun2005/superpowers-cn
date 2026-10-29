# Superpowers 会话诊断包

会话：<session-id>
运行环境：<name> <version>（<provenance label>）    Superpowers：<version>（<sha 或 "not a checkout">；<provenance label>）
脱敏级别：skeleton | evidence | full
构建时间：<ISO timestamp>

将头部版本字段标注为历史证据、未验证快照、当前观测或未知。`environment.json` 对每个环境字段及其支撑位置携带相同的来源区分。

## 这是什么

一份经过清洗的编码代理会话记录，该会话安装了 superpowers 并出了问题。它让未在场的代理或人员判断 superpowers 是否有责任，若有，应改什么。内部报告以 `path:line` 证据说明发生了什么。按设计，它不含对 superpowers 的诊断，也不含修复建议；那是读者的工作。

## 文件

- `report.md` — 诊断报告（问题陈述、结论、环境、会话、时间线、发现、关联性、覆盖说明）。
- `case.md` — 分析人员据以工作的案卷。
- `environment.json` — 环境部分的机器可读副本。
- `timeline.md` — 逐轮时间线。
- `findings/<dimension>.md` — 各维度的原始分析发现。
- `transcripts/<session-id>.md` — 每个受检会话的逐轮精简呈现（绝非原始 JSONL）。各工具结果正文级别：

  | 级别 | 工具结果正文 |
  |---|---|
  | skeleton | 有意限制；替换为 `[tool result: <tool>, <bytes> bytes, exit <code>]` |
  | evidence | 保留被引用事件，包括支撑发现所需的命令和结果 |
  | full | 全部保留 |
- `scrub-log.md` — 用到的每个占位符及其类别（绝不含原始值）。

## 如何阅读

先读 `report.md` §1–2，再读 §7（关联性）及其引用的证据行，然后读 `transcripts/` 中对应的轮次。`path:line` 引用指向报告者机器上的原始文件；精简转录中以 `[L<n>]` 标记保留相同行号。

## 脱敏

占位符形如 `<EMAIL-1>`、`<PERSON-2>`、`<SECRET-3>`、`<HOST-4>`、`<REPO-5>`、`<ORG-6>`、`<PROPRIETARY-7>`；主目录路径改写为 `~/…`。同一占位符在本包内始终指代同一原始值。

## 生成者说明

完成的包以实际结果替换这些说明。

清洗后，仅使用本包检查每条实质性导出的发现：将其引用解析到已包含的转录/来源标记，阅读被引用的命令/结果或引文，核实其支撑该主张。仅有路径和行存在不足以支撑。当脱敏级别或必要保留移除支撑时，记录具体限制。

核对报告、案卷、环境、发现、README 及任何本地 Issue 草稿。对照最终文件（不含日志本身）刷新 scrub-log 计数。移除过时的导出声明；区分包准备与归档交付。保留从历史锚点到已包含证据的映射。

将独立隐私审计与证据有用性分开记录：
- 隐私审计：CLEAN 或未解决的遗漏。
- 证据支撑：supported 或 limited，附受影响发现及原因。

若检查后内容变更，重复受影响的检查。提交最终日志、文件列表及两项结果，供现有归档审批。归档已审文件并核实交付的归档与之一致。在已审包之外记录归档交付，而非在审批后更改其内容。清洗并非穷尽性隐私认证。