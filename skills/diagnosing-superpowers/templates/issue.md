Title: <技能或症状>：<一行可观察现象>（<harness>）

- [x] 我已搜索现有 Issue，确认这不是重复问题（搜索词：<query terms>；最接近：<#n 标题，或“无”>）

## 谁提交这份报告？（必填）

<本次会话的模型和 harness>，代表 <你的伙伴希望如何被称呼，或“我的人类伙伴”> 提交，其请求为：“<本次会话中导致诊断或该 Issue 的请求，已脱敏>”

## 人类撰写的摘要（必填，200 字以内）

<你的伙伴对出了什么问题、本应发生什么的原话，按原样粘贴。请向他们索取；此处留占位符意味着该 Issue 尚不可提交。若篇幅过长，请他们自行精简，而非由你编辑其措辞。>

## 环境（必填）

| 字段 | 值 | 来源 / 支持证据 |
|-------|-------|-------------------------------|
| Superpowers 版本 | <version>（<sha 或“非检出”>） | <历史证据 / 未验证快照 / 当前观察 / 未知>；<location> |
| Harness（Claude Code、Cursor 等） | <harness> | <label>；<location> |
| Harness 版本 | <version> | <label>；<location> |
| 你的模型 + 版本 | <model ids seen> | <label>；<location> |
| 已安装的所有插件 | <list> | <label>；<location> |
| 操作系统 + Shell | <os version>，<shell> | <label>；<location> |

## 这是 Superpowers 的问题还是平台的问题？

- [ ] 我已确认，未安装 Superpowers 时不会出现此问题

我如何验证未安装 Superpowers 时此问题不复现：
<“未尝试”，或你的伙伴在未安装 Superpowers 时运行了什么、发生了什么，若有记录路径请附上。仅当记录显示问题未复现时，才勾选上方复选框。>

下方为涉及 Superpowers 的证据；这并不确立因果关系。

## 发生了什么？

<问题陈述，然后是分诊结论，将 `path:line` 引用改写为 `transcript line <n>`。>

## 复现步骤

1. <第一条人类提示，已脱敏>
2. <导致问题的各轮对话，每行一条>
3. <可观察现象>

## 预期行为

<来自问题陈述>

## 实际行为

<来自分诊结论>

## 调试日志或对话记录

会话 ID：<ids>。已交付本地归档：<path，脱敏级别 <level> | 未构建>。附加包：<无声明；仅在批准后附加>。
根据诊断报告，Superpowers 的涉及程度：<可能 | 很可能>，证据位于 <transcript lines>。本报告不提出修复方案。

---
使用 `diagnosing-superpowers` 技能提交。提交模型和 harness 见“谁提交”；被诊断会话的模型、harness、版本和插件见环境表。