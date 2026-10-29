# 将 Superpowers 移植到新 Harness

本指南说明如何为新 harness（非 Claude Code 的 IDE、CLI 或 agent 运行器）添加支持，使 Superpowers 技能能在其中像原生一样自动触发。

本指南分两层。**第 1–3 部分**说明系统如何工作、如何判断某个 harness 是否可支持；动手前先读这些。**第 4–8 部分**是给 agent（由人类伙伴监督）执行的规范性流程，端到端完成移植直至分发。附录索引了当前参考集成，便于复制最接近的一个。

集成机制因 harness 而异，且会持续变化。本指南刻意只讲**不变量**——无论机制如何都必须成立的东西——并指向一个可复制的在线参考实现。当本指南与代码冲突时，以代码为准；修正本指南。

## 开始之前

添加 harness 是本仓库风险最高的贡献类型。动笔之前：

- 完整阅读 `CLAUDE.md` 和 `.github/PULL_REQUEST_TEMPLATE.md`——贡献者规则和新 harness PR 要求不是可选项。
- 搜索**开放和已关闭**的 PR，看是否有人尝试过该 harness。若有，先弄清它为何停滞，再开始你自己的。

---

## 第 1 部分 — Superpowers 如何跨 harness 工作

Superpowers 在所有地方内容相同。每个 harness 变化的是那层薄薄的交付层：把内容送到模型面前，并把其指令翻译成 harness 的原生工具。三个组件：

1. **技能（与 harness 无关）。** `skills/` 中的一切都是事实来源，每个 harness 逐字共享。技能只描述*动作*——"调用技能"、"读取文件"、"派发子 agent"、"创建待办"——从不指名具体工具。正因如此，同一份技能正文无需修改即可在 Claude Code、Codex、Gemini、pi 等上运行。

2. **工具映射（按 harness）。** 每个 harness 都需要把动作词汇翻译成其真实工具名。该翻译位于 `skills/using-superpowers/references/<harness>-tools.md` 和/或 harness 的启动注入器中（见第 5 部分）。例如："*派发子 agent* → 调用 `task` 并传 `subagent_type`。"

3. **启动引导（按 harness）。** 每次会话开始时，完整的 `skills/using-superpowers/SKILL.md` 被注入模型上下文，包裹在 `<EXTREMELY_IMPORTANT>` 标签中，并附加工具映射。正是这个被注入的技能教会模型：技能存在，且行动前必须检查相关技能。**启动引导就是整个集成。** 没有它，技能文件是惰性的——存在于磁盘，从不被调用。

### 让这一切成立的两条规则

**1. 技能指名动作，不指名工具。** **不要**为适配你的 harness 而修改技能正文。移植只添加工具映射参考和启动注入器；绝不伸手进 `skills/*/SKILL.md` 替换工具名。（项目贡献者指南把技能内容视为精心调校的行为塑造代码；为"合规"而改写会被当场拒绝。）

**2. 一切通过 harness 自身的安装机制交付。绝不编辑用户文件。** 启动引导、技能和工具映射都作为*harness 所安装内容的一部分*交付——插件、扩展、市场条目、扩展捆绑的上下文文件。移植**不得**伸手进用户的全局或个人配置（`~/.gemini/config/AGENTS.md`、`settings.json`、`trustedFolders.json`、手改的 `~/.bashrc` 等）注入任何东西。harness 拥有它加载的内容；你的安装产物是你唯一能写的东西。如果安装机制确实无法承载启动引导，那是需要暴露的限制（第 6 部分）——绝不是手改用户配置的许可证。（形态 C *不是*例外：Gemini 的上下文文件没问题，因为它随*已安装扩展*一起交付，并由清单的 `contextFileName` 声明——harness 加载扩展自己的文件，而不是你在用户主目录里编辑的文件。）

---

## 第 2 部分 — 这个 harness 能被支持吗？

只有当 harness 能完成以下全部事项时，才能支持 Superpowers。写代码前先检查——若第一条不满足，停止。

### 硬性要求：会话启动时自动注入

harness 必须允许你在**每次会话开始时注入文本到模型上下文，无需人类伙伴逐会话选择加入。** 这是唯一不可协商的能力。它可以采取任何形式：

- **钩子/事件系统**，在会话开始时运行 shell 命令并读取其 stdout（Claude Code、Cursor、Copilot CLI），或
- **进程内插件/扩展**，带会话启动或消息生命周期回调，可修改消息数组（OpenCode、pi），或
- **指令文件**约定，harness 加载*你安装的扩展随附并声明*的上下文文件（例如 Gemini 的 `contextFileName` 指向扩展自己的 `GEMINI.md`）——而不是你在用户主目录编辑的文件。

如果让 Superpowers 出现在模型面前的唯一方式是让人类伙伴每次会话选择加入（粘贴提示、运行命令、启用模式），则该 harness **不能**被正确支持。第 3 部分的验收测试会失败，PR 会被关闭。这是"移植"不是真正移植的最常见原因。

### 其余能力清单

| 能力 | 为何需要 | 若缺失 |
|---|---|---|
| **技能发现 + 调用** | 模型必须能按需加载技能完整内容 | 若无原生技能工具，获准的降级方案是直接 `read` 相关 `SKILL.md`——见第 5 部分。既无技能工具又无文件读取的 harness 无法工作。 |
| **文件读 / 写 / 编辑** | 几乎所有技能都操作文件 | 必需。无变通。 |
| **运行 shell 命令** | TDD、验证、git 工作流 | 必需。 |
| **子 agent / 任务派发** | `dispatching-parallel-agents`、`subagent-driven-development` | 可降级：若不可用，这些特定技能会告诉模型内联完成或报告缺失能力——*绝不*编造 `Task` 调用。某些 harness 将此置于配置开关后（例如 Codex 需启用多 agent）。 |
| **待办 / 任务跟踪** | 多个技能中的进度跟踪 | 可降级：回退到计划文件或 `TODO.md`。 |
| **Web 抓取 / 搜索** | 少数技能 | 可降级。 |
| **Shell 或多语言脚本执行（Windows）** | 仅 shell 钩子形态需要，且仅当你要支持 Windows | 见第 7 部分。进程内插件 harness 完全绕过此问题。 |

"可降级"意味着：技能已有针对缺失工具的回退措辞。你在工具映射中的工作是：工具存在时指向真实工具，不存在时复用该回退措辞。

### 你可能根本不需要新目录

某些"新 harness"其实是不同安装器下的现有集成。例如 Factory 的 Droid 通过自己的 `plugin install` 命令消费 Claude Code 插件，此处无需新文件。构建前先检查 harness 能否直接加载现有清单。一个只在本仓库加一段 README 的移植是完全可接受的结果。

---

## 第 3 部分 — 完成定义

当**全部**以下条件成立时，移植完成：

1. `using-superpowers` 启动引导在每次会话开始时加载，无需逐会话选择加入。
2. 存在该 harness 的工具映射（位于 `references/<harness>-tools.md`、启动引导内联，或两者——按第 5 部分）。
3. 技能确实能被调用——原生或通过文档化的读 `SKILL.md` 回退——且模型遵循它们。
4. **验收测试通过。** 在干净会话中，用户消息：

   > Let's make a react todo list

   在*写任何代码之前*自动触发 `brainstorming` 技能。捕获完整记录——PR 需要它。
5. 测试覆盖集成（第 5 部分）并通过。
6. 真实用户能通过 harness 自身机制安装（而非手动复制文件），版本在适用处记录于 `.version-bump.json`（第 6 部分）。注意某些安装器会在安装时重写或剥离清单（有一个会将其缩减为 `{"name": …}`），因此"*已安装*文件报告仓库版本"并非总能实现——在源清单跟踪版本，不要把被重写的已安装清单视为失败。

完整验收测试前的快速冒烟检查：启动会话，让模型描述它的 superpowers。若启动引导已注入，它知道自己有。 （OpenCode 的安装文档用 `opencode run --print-logs "hello" 2>&1 | grep -i superpowers` 通过不同机制达到同样目的——日志 grep 而非询问模型；`2>&1` 很重要，因为日志走 stderr。找到你 harness 的等价方式。）

---

## 第 4 部分 — 选择集成形态

有三种结构形态，区别在于*如何让启动引导出现在模型面前*。选择匹配你 harness 所暴露能力的那个，然后复制该参考实现。形态决定了第 5 部分几乎一切——以下步骤据此分支。

### 如何判断你属于哪种形态

路由之前，先了解 harness 的*实际*机制——不要假设它文档完善，也不要假设它行为像它 fork 自的某个 harness。

**找到表面：**

- **网上搜索 harness 文档**（扩展 / 插件 / 钩子 / 技能 / MCP / "上下文文件" / "规则文件"）。厂商工具变化快；搜索而非信任训练知识。
- **找到并阅读该 harness 的现有第三方扩展/插件。** 真实可用的例子胜过文档——它展示清单形状、安装命令，以及 harness 实际加载哪些组件。
- 检查 harness 启动时加载什么：设置文件？扩展目录？每项目或全局指令文件（`AGENTS.md`、`<NAME>.md`）？

**若文档不足，经验性地逆向工程**（真实移植者都不得不做以下每一项）：

- 对二进制 `strings` / grep 安装树，找钩子事件名、配置路径和它读取的指令文件。
- **让运行中的模型枚举自己的工具名**——例如"列出你能调用的每个工具的精确机器名。"这是不编造工具名的权威方式（见步骤 4）。
- 用**唯一标记测试**证明每个假设：通过你认为可行的机制注入一个无意义 token，启动新会话，确认 token 确实到达模型。

**Fork 不继承父行为。** 派生自另一 harness 的 harness（例如 Gemini 派生的 CLI）可能暴露父的清单字段和 `@`-include 语法，却*仍不以相同方式尊重它们*。用标记验证；绝不假设父的配方可转移。

然后路由到形态：

- 会话开始时运行 shell 命令并读取其 stdout → **形态 A**。
- 带生命周期回调、可运行代码的插件/扩展模块 → **形态 B**。
- 只有常开指令文件，无钩子也无代码插件 → **形态 C**。

**形态可组合——并非互斥。** *技能发现*机制和*启动引导*机制不必是同一形态——但**两者都必须仍走安装机制**（规则 2）。分开决定两个问题：*技能在哪里被发现？* 和 *启动引导如何每次会话到达模型？* 一个 harness 可能通过插件安装技能，却需要以另一种安装交付方式传递启动引导（扩展声明的上下文文件，或——见下——harness 在会话开始时呈现已安装 `using-superpowers` 技能自身的描述）。若多个安装机制表面自动注入，选最可靠的。你**不得**做的是通过编辑用户全局配置来弥合缺口。

### 形态 A — Shell 钩子

harness 有钩子系统，在会话开始时运行 shell 命令并读取其 stdout 的 JSON。配置的命令运行 `run-hook.cmd`，一个多语言包装器，仅定位 bash 并分派命名脚本；脚本（`hooks/session-start` 或 harness 特定变体）读取 `using-superpowers/SKILL.md` 并打印一个 JSON 对象，其**字段名和嵌套因 harness 而异**。

- 参考：`hooks/session-start`、`hooks/run-hook.cmd`，以及按 harness 的钩子配置 `hooks/hooks.json`（Claude Code）和 `hooks/hooks-cursor.json`（Cursor）。
- 清单：`.cursor-plugin/plugin.json` 是形态 A 清单示例，指向 `./skills/` 和正确的 `hooks-*.json`。Claude Code 的 `.claude-plugin/plugin.json` 不设这两个字段——它按约定自动发现 `skills/` 和 `hooks/hooks.json`。**不要**为形态 A 复制 Codex 的 `.codex-plugin/plugin.json`：它声明空 `hooks` 对象专门抑制 Codex 的 `hooks/hooks.json` 自动发现，因为 Codex 原生呈现技能且不运行会话启动钩子。

> **钩子*系统*不是会话启动*事件*。** 一个 harness 可以有 `hooks.json` 机制——甚至二进制中含字面字符串 `SessionStart`——却没有在会话启动时触发且能注入上下文的钩子事件。（一个真实 harness 只暴露工具前/后和停止事件；`SessionStart` 字符串是遥测。）承诺形态 A 前，确认你需要的*特定事件*存在且能写入模型上下文。若不能，启动引导应放在指令文件中（形态 C）。

### 形态 B — 进程内插件 / 扩展

harness 加载暴露生命周期回调的 JS/TS 模块。你通过 harness API 注册技能目录，并通过在代码中修改消息数组注入启动引导。

- 参考：`.opencode/plugins/superpowers.js`（JavaScript）和 `.pi/extensions/superpowers.ts`（TypeScript）。pi 是任何**无原生技能工具** harness 的最接近参考。

### 形态 C — 指令文件

harness 既无 shell 钩子也无代码插件——其会话启动表面是*你安装的扩展随附且清单声明*的上下文文件（例如 Gemini 的 `contextFileName` → 扩展自己的 `GEMINI.md`）。你不能运行代码或修改消息；扩展的上下文文件指向启动引导。没有注入器来组装字符串或剥离 frontmatter——harness 按原样加载引用内容。**这能成立仅因为文件是已安装扩展的一部分**——绝不用"编辑用户全局 `GEMINI.md`/`AGENTS.md`"替代交付你自己的（规则 2）。

- 参考：`gemini-extension.json`（清单，含 `contextFileName`）、`GEMINI.md`（两个 `@`-include——启动引导技能和工具映射参考）、`skills/using-superpowers/references/gemini-tools.md`。
- 注意：`@`-include 是 Gemini 特性。若你的 harness 加载指令文件但无 include 语法，必须将启动引导内容内联到文件中。
- **不要信任 `@`-include 真的被展开——证明它。** Gemini *派生* harness 可能接受 `@./path` 语法，却将其视为*模型可选择读取的提示*（发出文件读取工具调用）而非保证的内联展开。这是启动引导每次会话可靠存在与模型可能读取之间的区别。运行唯一标记测试：若标记*未经工具调用*不在上下文中，**内联内容**而非 `@`-include。

### 路由表

| 若 harness… | 用形态 | 复制自 |
|---|---|---|
| 会话开始时运行 shell 命令并读取其 stdout | A（shell 钩子） | Cursor（`hooks/session-start` + `hooks/hooks-cursor.json` + `.cursor-plugin/`） |
| 是带会话/消息生命周期回调的 JS/TS 插件宿主 | B（进程内） | OpenCode（`.opencode/`）——若无原生技能工具则 pi（`.pi/`） |
| 随附扩展声明且始终加载的上下文文件 | C（指令文件） | Gemini（`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`） |
| 列出每个已安装技能描述，无会话启动事件也无上下文文件字段 | 技能索引启动引导（第 5 部分步骤 5，情况 2 回退） | Antigravity（`.antigravity-plugin/plugin.json` + 仅 `skills/`） |

大多数真实 harness 干净地匹配一行；最后一行是软情况——见步骤 5 的回退注意事项（规则 2 仍成立——绝不编辑用户配置）。

---

## 第 5 部分 — 移植流程

### 步骤 1 — 研究最接近的参考实现

打开第 4 部分为你的形态命名的文件，端到端阅读。以下模式是摘要；代码才是规范。

### 步骤 2 — 创建清单 / 入口点

创建 harness 用于识别插件的一切。精神上匹配现有：

- **形态 A：** 一个 `*-plugin/plugin.json`（见 `.cursor-plugin/plugin.json`），含 `name`、`version`、`description`、作者/许可证/关键词、`"skills": "./skills/"` 和 `"hooks": "./hooks/hooks-<harness>.json"`。加上 `hooks-<harness>.json` 本身，注册会话启动钩子，其命令调用 `run-hook.cmd`。
- **形态 B：** harness 加载的模块（例如 `.<harness>/plugins/*.js`）加上被发现所需的包元数据。提交的包元数据是**仓库根 `package.json`**：`main` 指向 OpenCode 插件，`pi` 字段（`pi.extensions`、`pi.skills`）加 `pi-package` 关键词声明 pi 扩展。按 harness 的本地清单和锁文件不纳入 git——`.opencode/.gitignore` 排除 `node_modules`、`package.json` 和锁文件。对你的 harness *本地*安装产物同样处理，以免污染仓库——但绝不 gitignore 仓库根 `package.json`，它是被跟踪的事实来源。
  - **构建/依赖检查。** 决定 harness 如何加载你的模块：直接运行源码（pi 的 `.ts` 从 `package.json` 原样引用；OpenCode 交付纯 `.js`），还是需要转译/构建步骤？Superpowers 零运行时依赖。pi 的 `import type { ExtensionAPI }` 能成立，正是因为 harness 直接运行 `.ts`、加载时