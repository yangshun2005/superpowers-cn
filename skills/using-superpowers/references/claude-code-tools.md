# Claude Code 工具说明

Claude Code 是参考 harness：技能使用它的词汇（`Agent` 表示子代理调度、todos、`Skill`）。本说明只涉及 Claude Code 能比技能默认形态更省成本地运行计划的那一处。它需由你的搭档主动启用，且不改变技能的任何要求。

## 为子代理驱动开发提供更省成本的编排

控制器会话是 superpowers:subagent-driven-development 运行中成本最高的席位：它要读取每个调度结果和每份报告，且通常跑在会话能力最强的模型上。Claude Code 支持嵌套子代理（默认在主对话下三层；`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` 可调整），因此整个循环可以下沉一层运行。

当你的搭档提出此要求——或表示会话模型太贵、不宜用于协调——就用中端模型调度一个编排器子代理，传入计划路径，并指示其端到端使用 superpowers:subagent-driven-development。编排器按该技能的 Model Selection 自行调度实现者和审查者；工作区和账本存放在磁盘上，多一层不会丢失任何东西。它的最终消息必须原样带上“Rulings I made”列表——该列表是决策传达给搭档的途径，你要转述它，而非概括它。

仅对整个计划这样做。为单个任务的调度嵌套一层毫无收益，只会多占一个席位。