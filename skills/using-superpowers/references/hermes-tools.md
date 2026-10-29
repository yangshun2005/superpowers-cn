# Hermes Agent 工具映射

技能以动作形式表达（"派发子代理"、"创建待办"、"读取文件"）。在 Hermes Agent 上，这些动作对应以下工具。

## 工具

| 技能请求的动作 | Hermes 工具 |
|---|---|
| 读取文件 | `read_file` |
| 创建新文件 | `write_file` |
| 编辑文件（定向补丁） | `patch` |
| 运行 shell 命令 | `terminal` |
| 搜索文件内容 | `search_files` |
| 按名称查找文件 | `terminal` 配合 `find` |
| 获取 URL / 读取网页 | `web_extract(urls=[...])` |
| 搜索网页 | `web_search(query=...)` |
| 派发子代理 | `delegate_task(goal=..., context=..., toolsets=[...], role="leaf")` |
| 任务跟踪 | `todo` 工具 |
| 调用技能 | `skill_view("skill-name")` |

## 指令文件

当技能提到"你的指令文件"时，在 Hermes Agent 上指的是项目目录下的 **`AGENTS.md`**，或全局的 **`SOUL.md`**（位于 `~/.hermes/SOUL.md`）。

## 调用技能

Hermes Agent 有一个 `skills` 工具集，包含 `skill_view` 和 `skills_list` 工具。
要调用 superpowers 技能，请使用：

```
skill_view("brainstorming")
skill_view("test-driven-development")
```

如果 `skill_view` 找不到 superpowers 技能（在插件完全注册之前，它可能不会出现在目录中），可回退为直接读取 SKILL.md：

```
read_file(path="~/.hermes/plugins/superpowers/skills/<skill-name>/SKILL.md")
```

此回退机制与不支持原生技能加载的其他运行环境所用的机制相同。

## 子代理派发

使用 `delegate_task` 派发隔离子代理，用于并行或串行工作流：

```
delegate_task(goal="...", context="...", toolsets=[...], role="leaf")
```

如果 `delegate_task` 不可用，请直接内联完成工作，而不是虚构工具调用。

## 任务跟踪

在会话内使用 `todo` 工具进行任务跟踪。多代理任务板请使用 `hermes kanban` CLI（如可用）。将旧的 `TodoWrite` 引用视为任务跟踪动作。