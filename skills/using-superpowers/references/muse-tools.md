# Muse 工具映射

技能以动作描述（“派发子代理”、“创建待办”、“读取文件”）。在 Muse 上，这些动作对应以下工具。

| 技能请求的动作 | Muse 对应工具 |
|----------------------|----------------|
| 读取文件 | `read_file` |
| 读取多个文件 | `read_file`（多次调用）或 `search` |
| 创建新文件 | `write_file` |
| 编辑文件 | `edit_file` |
| 运行 shell 命令 | `bash` |
| 搜索文件内容 | `search` |
| 按名称查找文件 | `search` 配合 `glob` |
| 获取 URL | `web_fetch` |
| 搜索网页 | `web_search` |
| 调用技能 | 对 `skills/<name>/SKILL.md` 执行 `read_file`，或使用原生技能工具 |
| 派发子代理（`Subagent (general-purpose):` 模板） | `subagent_spawn` 并填充提示词 |
| 任务跟踪（“创建待办”、“标记完成”） | `write_todos` 或 `bash` 任务文件 |
| 向用户提问 | `request_user_input` |

## 指令文件

当技能提到“你的指令文件”时，在 Muse 上指项目根目录下的 **`CLAUDE.md`** 或 **`AGENTS.md`**。Muse 会在已配置的位置按层级加载这些文件。

## 技能调用

Muse 通过 `muse skills` 提供原生技能支持。要调用 Superpowers 技能，读取其 `SKILL.md` 并遵循其中的指令。引导技能（`using-superpowers`）会通过插件 Hook 在 `SessionStart` 时自动注入——你已在遵循它，无需重新加载。

## 子代理派发

使用 `subagent_spawn` 将工作委派给隔离的子代理。派发前先填充提示词模板（如 `implementer-prompt.md`、`task-reviewer-prompt.md`）。若无子代理工具可用，则直接内联完成工作，不要虚构工具调用。

## 任务跟踪

使用 `write_todos` 进行清单跟踪。为每个技能清单项创建一个待办，随进度标记 in_progress/completed。若 `write_todos` 不可用，则通过 `write_file`/`edit_file` 维护一个 markdown 任务文件。