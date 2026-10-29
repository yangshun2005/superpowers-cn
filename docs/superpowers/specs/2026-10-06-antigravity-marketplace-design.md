# Antigravity Marketplace 插件

## 目标

将 Superpowers 列入 Google Antigravity 插件 Marketplace，作为与 Google 合作的一部分。这需要一个原生 Antigravity 插件清单，包含 Marketplace 展示字段、一个 logo，以及一个在原生插件路径上可用的会话引导机制。

来源：《Antigravity Plugin Marketplace Publishing Guide》（Google 提供的 PDF），以及下文记录的针对 `agy` 1.3.1 的探测结果。

## 当前状态（agy 1.3.1）

- `agy plugin install https://github.com/obra/superpowers` 可以成功，但仅通过 agy 的 Gemini 扩展回退路径：`agy plugin list` 报告 `"source": "gemini-cli"`。引导机制依赖 `GEMINI.md` 中的 `@` 行，模型将其视为提示，并用 `view_file` 跟进。验收测试在此路径上通过。
- agy 将我们的 Claude 格式 `hooks/hooks.json` 复制到安装根目录并命名为 `hooks.json`，每次会话都解析失败：
  `invalid hook "hooks": command hook must specify 'command'`。
- 在仓库上运行 `agy plugin validate .` 失败：`missing plugin.json`。

## 探测结果（agy 1.3.1）

每次探测都安装一个临时插件，运行 `agy -p`，然后卸载。

1. agy 的 Hook 事件为 `PreToolUse`、`PostToolUse`、`PreInvocation`、`PostInvocation` 和 `Stop`。没有 `SessionStart`。agy 的内嵌文档使用具名顶层 Hook（`{"<name>": {"PreToolUse": [...]}}`），与 PDF 中的 `{"hooks": {...}}` 示例不同。
2. 插件中的 `rules/AGENTS.md` 会在每次会话中作为常驻规则加载。
3. 在规则中，相对于规则文件的 `@` 路径会被改写为绝对安装路径（`@../skills/x/SKILL.md` 变为 `@/Users/<u>/.gemini/config/plugins/<plugin>/skills/x/SKILL.md`）。其内容不会被内联。规则中不会替换 `${PLUGIN_ROOT}`。无法相对于规则文件解析的路径保持原样。
4. 存在 `.antigravity-plugin/plugin.json` 时，agy 以 `"source": "antigravity"` 安装插件，找到全部 15 个技能，并跳过 Hook（没有根目录 `hooks.json`），因此解析警告消失。
5. 仓库的暂存副本带有原生清单和一行 `rules/AGENTS.md`（`@../skills/using-superpowers/SKILL.md`），通过了验收测试：模型读取 `using-superpowers`，然后 `antigravity-tools.md`，然后 `brainstorming`，并回复 "Using superpowers:brainstorming…" 及一个澄清问题。没有写代码。
6. 仓库根目录的 `AGENTS.md`（贡献者指南）会被复制到安装目录，但不会作为规则加载。只有 `rules/AGENTS.md` 会被加载。

## 设计

### 1. 原生清单：`.antigravity-plugin/plugin.json`

指南允许清单位于 `.antigravity-plugin/plugin.json`，这与我们其他 harness 的点目录一致。字段：

| 字段 | 值 |
|---|---|
| `name` | `superpowers` |
| `displayName` | `Superpowers` |
| `version` | 当前发布版本，由 `.version-bump.json` 维护 |
| `description` | `Make your coding agent a better engineer`（Jesse 的文案；比指南推荐的 120–160 更短，最多 160） |
| `logo` | `assets/antigravity-logo.png` |
| `suggestedPrompts` | `I've got an idea for something I'd like to build`、`Let's figure out why this broke`（Jesse 的文案） |
| `category` | `Developer Tools` |
| `keywords` | 小写，与 `.claude-plugin/plugin.json` 的 keywords 一致 |
| `author` | `{ "name": "Jesse Vincent", "email": "jesse@fsck.com" }`，与 `.claude-plugin/plugin.json` 一致 |
| `homepage`、`repository` | `https://github.com/obra/superpowers` |
| `license` | `MIT` |

没有 `skills` 或 `mcpServers` 字段：默认值（`skills/`、无 MCP 配置）正是我们想要的，探测 4 确认默认发现机制能找到所有技能。

### 2. 引导机制：技能发现，无额外文件

agy 会列出每个已安装技能的描述，`using-superpowers` 的 "Use when starting any conversation" 会让模型加载它。插件不附带 `rules/`，也不附带 `hooks.json`。

这取代了最初的设计——一行 `rules/AGENTS.md` 指针。最终审查发现 Cursor 也会从同一仓库根目录加载插件附带的任何 `rules/*.md`。在 agy 1.3.1 上探测并否决的替代方案：

- 一个 `PreInvocation` Hook。它在每次模型调用前运行，且 `hooks.json` 必须位于插件根目录。`ephemeralMessage` 会在下次调用时被丢弃；`userMessage` 会持久化，通过 `invocationNum == 0` 和 `initialNumSteps == 1` 门控可确保每次对话只注入一次，但 agy 仍会在每次调用时启动 Hook 进程。
- 没有清单字段可以重定位 `hooks.json` 或 `rules/`。

在无引导文件的情况下验证，12/12 次运行："Let's make a react todo list"（3 次）和 "add a settings page"（3 次）在任何代码之前加载 brainstorming；"this test is failing, can you fix it?"（3 次）在任何编辑之前加载 systematic-debugging；react-todo 在无关的第一轮之后的第二轮（3 次）加载 brainstorming。模型：Gemini 3.8 Flash。风险在于触发依赖于模型遵守技能描述；每次对话一次的 Hook 是失效时的回退方案。

### 3. Logo：`assets/antigravity-logo.png`

512×512 PNG，透明背景。一个白色圆角方块（内缩 31px，圆角半径 100）承载 `assets/superpowers-small.svg` 标记，颜色为 `#1a1a1a`，缩放至方块宽度的约 70% 并居中。无边框；在浅色主题下呈现为白底上的裸标记，可以接受。

`scripts/render-antigravity-logo.sh` 从 `superpowers-small.svg` 合成 SVG，并用 `rsvg-convert` 栅格化。这是重新生成 PNG 的唯一方式；它有帮助文本，并在缺少 `rsvg-convert` 时明确失败。

### 4. 版本号更新

将 `{ "path": ".antigravity-plugin/plugin.json", "field": "version" }` 添加到 `.version-bump.json`。

### 5. Codex 同步

将 `/.antigravity-plugin/` 和 `/assets/antigravity-logo.png` 添加到 `scripts/sync-to-codex-plugin.sh` 的 `EXCLUDES`，并在 `tests/codex-plugin-sync/test-sync-to-codex-plugin.sh` 中覆盖它们。

### 6. README

Antigravity 安装命令保持为 `agy plugin install https://github.com/obra/superpowers`。文字从"运行插件的 session-start Hook"改为描述技能发现。Marketplace 途径一旦存在就添加；在此之前，不做任何相关声明。

### 7. 测试：`tests/antigravity/`

- `test-plugin-manifest.sh`（CI 安全，参照 `tests/kimi/test-plugin-manifest.sh`）：
  - 清单可解析；`name` 匹配 kebab-case 模式且等于 `superpowers`；
  - 所有 Marketplace 必需字段存在且非空：`name`、`description`、`displayName`、`logo`、`suggestedPrompts`（1–3 个非空字符串）、`version`、`author`；
  - `description` 最多 160 个字符；
  - `version` 等于 `.claude-plugin/plugin.json` 的版本；
  - `logo` 文件存在、是 PNG、为正方形且至少 128×128；
  - `.version-bump.json` 列出了该清单；
  - `using-superpowers` 的描述仍包含 "Use when starting any conversation"，引导机制依赖于此；
  - 不存在根目录 `plugin.json`、`hooks.json` 或 `rules/`。
- 当 `agy` 在 `PATH` 上时，还在仓库上运行 `agy plugin validate`，并要求 `[ok]` 且技能已处理。
- 更新 `test-antigravity-tools.sh` 的头部注释，其中仍写着 agy 运行 SessionStart Hook。

实时验收（不在 CI 中）：将分支的目录树安装到 agy，在干净工作区中至少运行三次 "Let's make a react todo list"，并要求每次都在任何代码之前触发 brainstorming。之后卸载。

### 8. 移植指南：单独提交

`docs/porting-to-a-new-harness.md` 描述了不存在的 `.antigravity-plugin/install.sh` 和生成的 `ANTIGRAVITY.md`。在其自己的提交中修复：

- 路由表行：Antigravity 使用暴露的技能索引。
- "no `Skill` tool" 部分：将 Antigravity 的 `contextFileName` 建议替换为技能索引引导，并记录 Hook 和规则发现（无 session-start 事件、`PreInvocation` 每次调用、Hook 中无 `${PLUGIN_ROOT}`、共享插件根目录会将 `rules/` 泄漏到 Cursor）。
- 分发表：Antigravity 从 git URL 安装（`agy plugin install <url>`）并在其 Marketplace 中列出。
- 安装器剥离部分：删除 `install.sh` / `ANTIGRAVITY.md` 示例。
- harness 索引表没有 Antigravity 行；添加一行（清单、引导、工具映射、测试、安装）。

## 范围之外

- Marketplace 提交流程。指南未描述如何提交；这通过 Jesse 的 Google 联系人进行。
- Antigravity 2.0 和 IDE。此处未安装，因此未验证。README 不对它们做任何声明。
- 开放 PR #2350，它将 `using-superpowers` 复制到生成的规则中。Jesse 在此落地后决定如何回应；我起草回复。
- `.hermes-plugin/` 和 `.muse-plugin/` 在 Codex 同步排除项中缺失。无关；记录以便单独修复。