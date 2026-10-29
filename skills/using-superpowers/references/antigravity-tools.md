# Antigravity CLI（`agy`）工具映射

技能用动作描述（“派发子代理”“创建待办”“读取文件”）。在 Antigravity CLI（`agy`）上，这些动作对应以下工具。

| 技能请求的动作 | Antigravity CLI 对应方式 |
|----------------------|----------------------|
| 派发子代理（`Subagent (general-purpose):` 模板） | `invoke_subagent`，使用内置 `TypeName` —— `self` 用于全能力工作，`research` 用于只读 |
| 任务跟踪（“创建待办”“标记完成”） | **任务产物** —— `write_to_file`，设置 `IsArtifact: true` 和 `ArtifactType: "task"`（见[任务跟踪](#task-tracking)）。**不是** `manage_task`，后者管理后台进程。 |

## 任务跟踪

Antigravity **没有待办工具**（`manage_task` 管理后台进程 —— `list`/`kill`/`status`/`send_input` —— 它*不是*清单）。当技能要求创建待办列表或跟踪任务时，维护一个**任务产物**：用 `write_to_file` 保存的 markdown 清单（`IsArtifact: true`、`ArtifactMetadata.ArtifactType: "task"`），随进展用 `replace_file_content` / `multi_replace_file_content` 编辑。

任何多步骤任务开始时，创建任务产物，列出计划的每一步。每完成一步，编辑产物标记为完成（`- [x]`）。计划有变就更新清单。保持其最新 —— 它是剩余工作的唯一依据；对话变长后，开始每一步前先重读它。