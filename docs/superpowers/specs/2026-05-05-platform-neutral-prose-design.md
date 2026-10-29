# 平台中立文案 — 阶段 A 设计

## 背景

Superpowers 发布到多个 agent 运行时（Claude Code、Codex、Cursor、OpenCode、Copilot CLI、Gemini CLI）。技能内容和支持文档最初是为 Claude Code 编写的，在适用于任何运行时 agent 的地方使用了"Claude"。OpenAI 的 vendored fork（openai/plugins#217）尝试了整体重写，但在多处明显出错——重写了历史归属路径、模型名称和平台特定的安装说明——我们希望避免这个错误，同时仍然移除那些确实是附带性的平台中心化文案。

整个工作按引用类别分阶段进行。**本 spec 仅覆盖阶段 A：** 在非平台特定上下文中提及"Claude"的通用第三人称文案。后续阶段（配置文件引用、营销文案、工具名引用）不在本范围内，将有自己的 spec。

## 范围内

在以下位置中通用文案提及"Claude"：

- `skills/*/SKILL.md` 及活跃技能目录中的支持性 `.md` 文件
- `skills/writing-skills/anthropic-best-practices.md`
- `README.md`（仅限通用文案提及，不含平台营销）

外加一个自造术语重命名：**Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**，位于 `skills/writing-skills/SKILL.md`。

## 范围外

- **平台/运行时声明** — "In Claude Code:"、安装说明、工具映射引用。（阶段 D 候选。）
- **配置文件引用** — CLAUDE.md、AGENTS.md、GEMINI.md 优先级列表及"项目约定放在哪里"的提示框。（阶段 B。）
- **工具名引用** — `Skill`、`Bash`、`Read`、`Task`、`TodoWrite`。技能使用 Claude Code 的工具词汇编写；现有的 `references/{codex,copilot,gemini}-tools.md` 文件负责映射。（编写本 spec 时，计划是推迟或跳过这些。阶段 E 最终完成了它们——在活跃技能中用动作语言替换工具名，并围绕同一词汇统一平台工具引用。）
- **README 中的营销文案** — "Superpowers for Claude Code"、以平台命名的安装章节。（阶段 C。）
- **历史产物** — `docs/plans/*.md`、`docs/superpowers/specs/*.md`、`CREATION-LOG.md`。这些是有日期的、时间点文档；重写它们就是重写历史。
- **模型标识符** — Claude Haiku / Sonnet / Opus。这些是真实产品名称。
- **文件名 / URL 引用** — `CLAUDE.md`、`claude.com`、`claude-plugin/`、`~/.claude/` 下的路径。
- **`anthropic-best-practices.md` 文件名** — 即使我们重写了其中的文案，文件仍以其来源命名。

## 替换风格

使用在英文中读起来自然的混合方式：

- **第二人称 — "your agent"**，当向技能作者谈论*其*运行时时
  - "your agent reads the description"
- **第三人称 — "the agent" / "agents" / "an agent"**，当通用描述系统行为时
  - "Future agents find your skills"
  - "Use words an agent would search for"
  - "Agents read SKILL.md only when the skill becomes relevant"

选择适合周围句子的形式；不要为了强行一致而牺牲通顺。自然时使用复数（"future agents"、"agents read"），而不是总是说"the agent"。

### 保留为"Claude"的例外

- 模型名称：Claude Haiku、Claude Sonnet、Claude Opus
- 文件名和 URL：`CLAUDE.md`、`claude.com`、`~/.claude/`
- 品牌平台名"Claude Code"，凡是指代该运行时本身之处（在后续阶段处理）

### 自造术语重命名

- **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**
  - 出现在 `skills/writing-skills/SKILL.md` 中作为章节标题及附近文案。重命名标题、缩写及任何文件内交叉引用。

## 受影响的文件

基于过滤掉例外后的 `grep` 的近似计数：

| 文件 | 通用文案提及数 |
|------|----------------|
| `skills/writing-skills/SKILL.md` | ~12（含 CSO 标题 + 正文） |
| `skills/writing-skills/anthropic-best-practices.md` | ~30 |
| `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` | ~1 — 文件名保留（它是 CLAUDE.md 测试产物）；"Variant C: Claude.AI Emphatic Style"标题也保留（它是命名特定风格的标签） |
| `README.md` | ~1 |

最终列表在实现期间通过重新运行过滤后的 grep 确认。

## 提交计划

四个原子提交，按顺序：

1. **重命名 CSO → SDO**，位于 `skills/writing-skills/SKILL.md`。机械性、隔离、如果我们改变对该术语的想法易于回退。
2. **活跃技能文案** — 在 `skills/*/SKILL.md` 及支持性 `.md` 中将通用"Claude"改为"agent"形式，排除 `anthropic-best-practices.md`。
3. **`anthropic-best-practices.md` 文案** — 相同的替换规则。单独提交，因为此文件是外部文档的 vendored 改编；隔离变更使未来与上游的对账更易读。
4. **README.md 文案** *（仅当过滤后仍有通用文案提及）*。若为空则跳过。

每个提交消息命名阶段（"Phase A"）和切片（"rename CSO to SDO"、"agent prose in active skills"等），使系列自文档化。

## 验证

每次提交后：

- `grep -rn "Claude" <touched-paths>` — 每个剩余命中必须落入已记录的例外（模型名称、文件名、URL、"Claude Code"平台名、历史产物）。
- 端到端阅读受影响的文件 — 替换不应破坏句子流畅性、代词一致性或列表平行性。
- 无需运行测试；这仅是文案。

最终提交后：

- 在实时会话中浏览每个修改的技能，确认没有读起来别扭之处。

## 非目标

- 不改变行为、结构、标题（CSO→SDO 除外）、示例、代码块或 YAML frontmatter。
- 不引入新章节、提示框或兼容性说明。
- 编辑时不"改进"文案超出替换范围。