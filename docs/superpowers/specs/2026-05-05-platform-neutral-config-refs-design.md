# 平台无关的配置文件引用 — 阶段 B 设计

## 背景

阶段 A（见 `2026-05-05-platform-neutral-prose-design.md`）已将通用的第三人称"Claude"表述替换为 agent 中立形式。本阶段处理下一类问题：技能中对各平台指令文件（CLAUDE.md、AGENTS.md、GEMINI.md）的引用。

插件运行在多个 harness 上，每个 harness 读取各自的指令文件。技能若把 CLAUDE.md 当作唯一文件来命名，就是 Claude Code 中心的假设，在 Codex / Gemini CLI / OpenCode 上不成立。

## 范围内

活跃技能中的两处具体行：

1. **`skills/writing-skills/SKILL.md:58`** — `Project-specific conventions (put in CLAUDE.md)`
2. **`skills/receiving-code-review/SKILL.md:30`** — `"You're absolutely right!" (explicit CLAUDE.md violation)`

## 范围外

- **`skills/using-superpowers/SKILL.md:22, 26`** — 指令优先级列表。该列表已同时列出三者（CLAUDE.md、GEMINI.md、AGENTS.md），这是正确的：该节在就多平台插件上*什么算作用户指令*做出真实声明。无需改动。
- **历史 / 示例产物**：
  - `skills/systematic-debugging/CREATION-LOG.md` — 归属路径（`~/.claude/CLAUDE.md`）是历史事实。
  - `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` — 整个文件是测试 CLAUDE.md 内容变体的完整示例。文件名、正文以及 `testing-skills-with-subagents.md` 中的引用都保留；规范化它们会破坏该示例。
- **平台工具引用** — 阶段 D 候选：
  - `skills/using-superpowers/SKILL.md:40`（关于 GEMINI.md 的 Gemini CLI 工具映射说明）
  - `skills/using-superpowers/references/gemini-tools.md`（`save_memory` 持久化到 GEMINI.md）

## 替换规则

两处不同的调用，每个范围内行各一处。

### 规则 1："项目特定约定放在哪里"

`writing-skills/SKILL.md:58`：

- **改前：** `Project-specific conventions (put in CLAUDE.md)`
- **改后：** `Project-specific conventions (put in your instructions file)`

使用通用短语，而非选定某个文件名。不同 harness 读取不同文件（CLAUDE.md、AGENTS.md、GEMINI.md 等），技能不应假设其一。平台工具参考文档（`references/{codex,copilot,gemini}-tools.md`）才是命名各平台首选文件的正确位置。

### 规则 2："(explicit CLAUDE.md violation)" 括号

`receiving-code-review/SKILL.md:30`：

- **改前：** `"You're absolutely right!" (explicit CLAUDE.md violation)`
- **改后：** `"You're absolutely right!" (explicit instruction-file violation)`

该括号承担实际作用——它表明这句话不只是风格不佳，而是主动违反许多用户写入指令文件的规则。"Instruction file"是覆盖 AGENTS.md / CLAUDE.md / GEMINI.md 的天然跨平台术语，保留原信号，既不选定某个文件名，也不弱化为"common"。

## 提交计划

原子提交，按顺序：

1. **`writing-skills/SKILL.md`** — 在"项目约定放在哪里"行中将 CLAUDE.md → "your instructions file"
2. **`receiving-code-review/SKILL.md`** — 在违规括号中将 CLAUDE.md → instruction-file
3. **平台工具参考文档** — 向每个 `references/{codex,copilot,gemini}-tools.md` 添加各平台首选的指令文件名（CLAUDE.md、AGENTS.md、GEMINI.md 等），使读者能将"your instructions file"解析为真实文件名。

每条提交信息注明"Phase B"及切片。

## 验证

每次提交后：

- 阅读周围段落，确认语法和含义仍可解析。
- `grep -n "CLAUDE\.md" <touched-file>` — 活跃正文中无剩余命中（例外已记录）。

两次提交后：

- `grep -rn "CLAUDE\.md" skills/` 应仅返回已记录的例外（CREATION-LOG、CLAUDE_MD_TESTING 及其入站引用、using-superpowers 中的优先级列表）。

## 非目标

- 不要改动 `using-superpowers/SKILL.md` 中优先级列表的顺序。重排 CLAUDE.md / GEMINI.md / AGENTS.md 是审美改动，不是替换，此处不在范围内。
- 不要重命名 `examples/CLAUDE_MD_TESTING.md` 或改动其内容。
- 不要修改 Gemini CLI 特定的工具引用（阶段 D 候选）。

## 实现说明

此处所述的阶段 B 涵盖三次提交和三个非 Claude Code 平台工具引用。实现更进一步：为对称性，在提交 `8505703` 中新增了第四个引用 `references/claude-code-tools.md`，使 Claude Code 的指令文件约定和工具名列表与其他平台并列，而非隐含在周围技能正文中。该新增未在本规范中预见，但与其意图一致。