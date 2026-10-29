# Superpowers for OpenCode

在 [OpenCode.ai](https://opencode.ai) 上使用 Superpowers 的完整指南。

## 安装

OpenCode V2 需要 2.0.4 或更高版本。

### OpenCode V1

使用现有的 V1 插件配置：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

### OpenCode V2（2.0.4 或更高版本）

使用 V2 插件配置：

```json
{
  "plugins": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

本地 V2 安装时，配置包含 `index.js` 的仓库目录。OpenCode 2.0.4 和 2.0.7 会拒绝配置直接指向 JavaScript 文件的路径。通过符号链接发现的插件仍受支持。

重启 OpenCode。V2 使用 `opencode` 命令；`opencode2` 可能作为别名可用。插件通过 OpenCode 的插件管理器安装，并注册所有技能。

通过提问验证："Tell me about your superpowers"

### 从旧的基于符号链接的安装迁移（V1）

如果你之前通过 `git clone` 和符号链接安装 superpowers，请移除旧配置：

```bash
# Remove old symlinks
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# Optionally remove the cloned repo
rm -rf ~/.config/opencode/superpowers

# Remove skills.paths from opencode.json if you added one for superpowers
```

然后按照上述安装步骤操作。

## 使用

### 查找技能

使用 OpenCode 原生的 `skill` 工具列出所有可用技能：

```
use skill tool to list skills
```

### 加载技能

```
use skill tool to load brainstorming
```

### 个人技能

在 `~/.config/opencode/skills/` 中创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目技能

在项目的 `.opencode/skills/` 中创建项目专属技能。

**V2 技能优先级：** 项目技能 > 个人技能 > Superpowers 技能。在测试过的 V1 1.18.31 上，当个人技能或项目技能同名时，内置的 Superpowers 技能优先；请为个人技能和项目技能使用不同的名称。迁移不会改变此行为。

## 更新

OpenCode 通过基于 git 的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会将解析后的 git 依赖固定在锁文件或缓存中，因此重启可能不会拉取最新的 Superpowers 提交。如果更新未生效，请清除 OpenCode 的包缓存或重新安装插件。

要固定特定版本，请在规范中添加标签或提交（V1 的 `plugin` 键和 V2 的 `plugins` 键形式相同）：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v7.0.1"]
}
```

在 V2 上，请固定 `v6.4.1` 或更高版本；`v6.3.0` 及更早版本仅在 V1 上加载。

## 工作原理

插件做两件事，使用宿主版本特定的 API：

1. **注册技能目录**，使 OpenCode 无需符号链接或手动配置即可发现所有 superpowers 技能。
    - **V1：** 通过 `config` Hook，注入到 `config.skills.paths`
    - **V2：** 通过 `setup()` 函数使用 `ctx.skill.transform()`（V2 原生 API，运行时确认有效）
2. **注入引导上下文**，使用版本特定的工具映射：V1 会话获得下面的 V1 工具名，V2 会话获得 V2 工具名。
    - **V1：** 通过 `experimental.chat.messages.transform` Hook
    - **V2：** 通过 `ctx.session.hook("context")` —— V2 的等价方式（运行时确认有效）

控制器会话在临时模型上下文中接收 using-superpowers 引导。委派的子会话保留对原生技能的访问，但不接收控制器引导。没有父会话的手动 Fork 保留控制器行为。当 V2 原生压缩保留较早的用户消息时（默认 `compaction.keep.tokens` 预算），引导会放入检查点之前第一条保留的用户消息中，与未压缩会话相同。当压缩移除所有用户消息时，插件会在检查点之后追加一条临时引导消息。无论哪种方式，已保存的历史记录都不变。

如果会话查找失败，插件会为该请求保留引导，并在下一个请求时重试。失败的查找不会缓存为控制器决策。

### 工具映射

技能以动作而非具体运行时工具名来表达。引导会将它们映射到你的 OpenCode 版本实际暴露的工具。

**V1（`opencode` 1.x）：**

- "Create a todo" / "mark complete in todo list" → `todowrite`
- `Subagent (general-purpose):` 模板 → OpenCode 的 `task` 工具，使用 `subagent_type: "general"`（代码库探索用 `"explore"`）
- "Invoke a skill" → OpenCode 原生的 `skill` 工具
- "Read a file" → `read`
- "Create a file" / "edit a file" / "delete a file" → `apply_patch`
- "Run a shell command" → `bash`
- "Search file contents" / "find files by name" → `grep`、`glob`
- "Fetch a URL" → `webfetch`

**V2（`opencode` 2.0.4 或更高版本；`opencode2` 可能作为别名可用）：**

- "Create a todo" → V2 没有任何 todo 工具；映射会告诉模型改用 markdown 文件（或 harness 的计划功能）来跟踪计划
- `Subagent (general-purpose):` 模板 → OpenCode 的 `subagent` 工具，使用 `agent: "general"`（或 `"explore"`）；传入 `sessionID` 以继续之前的子代理
- "Invoke a skill" → OpenCode 原生的 `skill` 工具
- "Read a file" → `read`
- "Create, edit, or delete files" → 可用时使用带 `patchText` 的 `patch`；否则用 `write` 创建或覆盖文件，用 `edit` 做定向修改，用 `shell` 删除
- "Run a shell command" → `shell`（`command`、`workdir`、`timeout`、`background`）
- "Search file contents" / "find files by name" → `grep`、`glob`
- "Fetch a URL" → `webfetch`
- "Search the web" → `websearch`

简而言之，V2 将 `task` 重命名为 `subagent`（代理名从 `subagent_type` 移到 `agent`，通过带 `sessionID` 重新调用来继续），将 `apply_patch` 重命名为 `patch`，将 `bash` 重命名为 `shell`，并完全移除了 todo 工具。可用的修改工具取决于所选模型：`patch` 对选定的 GPT 模型 ID 可用，其他模型使用 `write` 和 `edit`。

（V1 列表已对照安装的 OpenCode 1.18.x CLI 工具清单验证；V2 列表已对照 OpenCode 2.0.4 和 2.0.7 宿主契约验证。）

## 故障排查

### 插件未加载

**V1：** 检查 OpenCode 日志：

```
opencode run --print-logs "hello" 2>&1 | grep -i superpowers
```

**V2：** 插件在后台服务器中加载，`--print-logs` 仅在配合 `--standalone` 时显示其日志：

```
opencode run --standalone --print-logs "hello" 2>&1 | grep -i superpowers
```

或检查 `~/.local/share/opencode/log/opencode.log`，过滤 `role=server`。

同时确认 `opencode.json` 中的插件路径正确，且运行的是较新版本的 OpenCode。

### Windows 安装问题

某些 Windows OpenCode 构建存在上游安装器问题，涉及基于 git 的插件规范，包括 `git+https` URL 的缓存路径，以及 Bun 即使在普通终端中可用也找不到 `git.exe`。如果 OpenCode 无法安装插件，请尝试用系统 npm 安装，并让 OpenCode 指向本地包：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中为你的 OpenCode 版本使用已安装包的绝对路径。OpenCode 不会展开 `~`；`~/...` 条目会被视为包名，而非本地目录。

**V1：**

```json
{
  "plugin": ["C:\\Users\\<you>\\.config\\opencode\\node_modules\\superpowers"]
}
```

**V2（2.0.4 或更高版本）：**

```json
{
  "plugins": ["C:\\Users\\<you>\\.config\\opencode\\node_modules\\superpowers"]
}
```

### 找不到技能

1. 使用 OpenCode 的 `skill` 工具列出可用技能
2. 检查插件是否正在加载（见上文）
3. 每个技能都需要一个带有效 YAML frontmatter 的 `SKILL.md` 文件

### 引导未出现

- **V1：** 检查 OpenCode 版本是否支持 `experimental.chat.messages.transform` Hook。更改配置后重启 OpenCode。
- **V2：** 插件使用 `ctx.session.hook("context")` 注入引导。通过 `opencode api get /api/plugin` 验证插件已加载。更改配置后用 `opencode service restart` 重启。`opencode2` 命令可能作为别名可用。

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode V2 文档：https://opencode.ai/v2/docs/
- OpenCode V1 文档：https://opencode.ai/docs/