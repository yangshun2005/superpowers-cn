# Gemini CLI 工具映射

技能以动作描述（“派发子代理”、“创建待办”、“读取文件”）。在 Gemini CLI 上，这些动作对应以下工具。

| 技能请求的动作 | Gemini CLI 对应工具 |
|----------------------|----------------------|
| 读取文件 | `read_file` |
| 一次读取多个文件 | `read_many_files` |
| 创建新文件 | `write_file` |
| 编辑文件 | `replace` |
| 运行 shell 命令 | `run_shell_command` |
| 搜索文件内容 | `grep_search` |
| 按名称查找文件 | `glob` |
| 列出文件和子目录 | `list_directory` |
| 获取 URL | `web_fetch` |
| 搜索网络 | `google_web_search` |
| 调用技能 | `activate_skill` |
| 派发子代理（`Subagent (general-purpose):` 模板） | `invoke_agent`，`agent_name: "generalist"`（可通过 `@generalist` 聊天语法调用——见[子代理支持](#subagent-support)） |
| 多个并行派发 | 同一响应中多次调用 `invoke_agent` |
| 任务跟踪（“创建待办”、“标记完成”） | `write_todos`（状态：pending、in_progress、completed、cancelled、blocked） |

## 指令文件

当技能提到“你的指令文件”时，在 Gemini CLI 上指的是 **`GEMINI.md`**。Gemini CLI 按层级加载 `GEMINI.md`：全局文件位于 `~/.gemini/GEMINI.md`，项目级文件位于工作区目录及其祖先目录，子目录中的 `GEMINI.md` 在工具访问这些目录中的文件时加载。

## 个人技能目录

用户级技能位于 **`~/.gemini/skills/`**，**`~/.agents/skills/`** 作为跨运行时别名（与 Codex 和 Copilot CLI 共享）。同一作用域下两个目录都存在时，`.agents/skills/` 优先。每个技能是一个子目录，包含 `SKILL.md`（带有 `name` 和 `description` frontmatter）。

## 子代理支持

Gemini CLI 通过 `invoke_agent` 工具派发子代理，该工具接受 `agent_name` 和 `prompt` 参数。同样的派发也以聊天语法快捷方式提供：输入 `@generalist <prompt>` 等同于调用 `invoke_agent` 并设置 `agent_name: "generalist"`。内置代理名称包括 `generalist`、`cli_help`、`codebase_investigator`，以及（启用浏览器工具时）`browser_agent`。

技能以 `Subagent (general-purpose):` 派发，并引用提示模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`）或提供内联提示。在 Gemini CLI 上：

| 技能派发形式 | Gemini CLI 对应方式 |
|---------------------|----------------------|
| 引用 `*-prompt.md` 模板（implementer、task-reviewer、code-reviewer 等） | 填写模板，然后调用 `invoke_agent`，`agent_name: "generalist"`，传入填写后的提示 |
| 引用 `superpowers:requesting-code-review` 的 `./code-reviewer.md` | 调用 `invoke_agent`，`agent_name: "generalist"`，传入填写后的审查模板 |
| 内联提示（未引用模板） | 调用 `invoke_agent`，`agent_name: "generalist"`，传入你的内联提示 |

### 提示填写

技能提供的提示模板包含 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]` 等占位符。在将完整提示传给 `invoke_agent` 之前，填写所有占位符。提示模板本身包含代理的角色、审查标准和预期输出格式——子代理会遵循它。

### 并行派发

Gemini CLI 支持并行派发子代理。在同一响应中多次调用 `invoke_agent`（或在一个提示中多次调用 `@generalist`），即可并行运行独立的子代理任务。依赖任务保持串行，但不要仅为保持更简单的历史记录而将独立的子代理任务串行化。

## 其他 Gemini CLI 工具

以下工具为 Gemini CLI 独有：

| 工具 | 用途 |
|------|---------|
| `save_memory`（旧版） | 当 `experimental.memoryV2 = false` 时跨会话持久化事实 |
| `get_internal_docs` | 查阅 Gemini CLI 内置文档 |
| `ask_user` | 向用户提出结构化问题（文本 / 单选 / 多选） |
| `enter_plan_mode` / `exit_plan_mode` | 进入和退出只读计划模式 |
| `update_topic` | 更新当前对话的主题 / 战略意图元数据 |
| `complete_task` | 表示 Gemini 子代理已完成，并将结果返回给父代理 |
| `tracker_create_task`、`tracker_update_task`、`tracker_get_task`、`tracker_list_tasks`、`tracker_add_dependency`、`tracker_visualize` | 支持依赖和可视化的丰富任务跟踪器 |
| `read_mcp_resource`、`list_mcp_resources` | MCP 资源访问 |