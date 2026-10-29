# Superpowers for Kimi Code

在 [Kimi Code](https://github.com/MoonshotAI/kimi-code) 中使用 Superpowers 的完整指南。

## 安装

Superpowers 已在 Kimi Code 的插件市场中提供。

打开插件管理器：

```text
/plugins
```

进入 `Marketplace` > `Superpowers` 并安装。

也可以从本仓库安装：

```text
/plugins install https://github.com/obra/superpowers
```

如需针对 `dev` 分支验证未发布版本，请显式指定分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

Kimi Code 会将插件变更应用到新会话。安装、更新、启用、禁用或重新加载插件后，请用 `/new` 启动新会话。

## 工作原理

Kimi 插件清单位于 `.kimi-plugin/plugin.json`。

该清单做三件事：

1. 将 Kimi Code 指向现有的 `skills/` 目录。
2. 通过 `sessionStart.skill` 在会话启动时加载 `using-superpowers`。
3. 通过 `skillInstructions` 提供 Kimi 专用的工具映射。

Kimi Code 直接从本仓库读取 Superpowers 技能。没有复制的技能、符号链接、Hook 或额外的运行时依赖。

## 工具映射

技能描述的是动作，而非硬编码某个运行时的工具名。在 Kimi Code 上，这些动作对应如下：

- “询问用户” / “提出澄清问题” -> `AskUserQuestion`
- “创建待办” / “在待办列表中标记完成” -> `TodoList`
- “派发子代理” -> `Agent`
- “调用技能” -> Kimi Code 原生的 `Skill` 工具
- “读取文件” / “写入文件” / “编辑文件” -> `Read`、`Write`、`Edit`
- “运行 shell 命令” -> `Bash`
- “搜索文件内容” -> `Grep`
- “按路径或模式查找文件” -> `Glob`
- “获取 URL” -> `FetchURL`
- “搜索网页” -> `WebSearch`

## 更新

使用 Kimi Code 的插件管理器：

```text
/plugins
```

选择 Superpowers 并在其中更新。更新后用 `/new` 启动新会话。

## 故障排查

### 插件未加载

1. 运行 `/plugins info superpowers` 检查诊断信息。
2. 确认插件已启用。
3. 安装或更新后用 `/new` 启动新会话。

### 直接从 GitHub 安装使用了旧发布版本

当仓库 URL 为裸地址且存在最新 GitHub 发布版本时，Kimi Code 会安装该发布版本。如需在下一个 Superpowers 发布版本之前测试未发布的变更，请显式安装分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

### 技能未触发

1. 确认 `/plugins info superpowers` 显示插件已启用。
2. 用 `/new` 启动新会话。
3. 尝试验收提示词：`Let's make a react todo list`。正常安装应在编写代码前加载 `brainstorming`。