

> **[中文化] superpowers**
>
> 此项目是 [obra/superpowers](https://github.com/obra/superpowers) 的中文翻译版本。
> - 原项目 Stars: 307936
> - 主语言: Shell
> - 许可证: MIT
> - 翻译日期: 2026-10-29
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

Superpowers 是一套面向编码 Agent 的软件开发方法论，由一组可组合的 skills 和初始指令构成，目的是让 Agent 在动手写代码前先弄清需求、产出规格与实施计划，再按计划推进开发。它解决的是编码 Agent 容易直接跳进写代码、缺少需求澄清和计划约束的问题。skills 会自动触发，用户无需额外操作。

### 快速上手（最短路径）

安装方式因所用的 harness（宿主工具）而异；如果你同时使用多个 harness，需要为每个分别安装。以下以 Claude Code 为例：

1. 从 Anthropic 官方插件市场安装：

   ```bash
   /plugin install superpowers@claude-plugins-official
   ```

2. 或者先注册 Superpowers 市场，再从该市场安装：

   ```bash
   /plugin marketplace add obra/superpowers-marketplace
   ```

   ```bash
   /plugin install superpowers@superpowers-marketplace
   ```

3. 安装后正常启动你的编码 Agent 即可，skills 会自动触发，无需额外配置。

其他 harness（如 Antigravity、Codex、Cursor、Gemini CLI、OpenCode、Pi、Qwen Code 等）的安装命令各不相同，原文在 Installation 一节按 harness 分别列出，请参考 README 正文对应小节。

### 环境要求

- **操作系统**：原文未说明。
- **硬件**：原文未说明。
- **依赖**：需要一个受支持的编码 Agent harness（如 Claude Code、Codex、Cursor、Gemini CLI、OpenCode、Pi、Qwen Code 等，完整列表见 README 目录）。
- **版本要求**：原文未说明；package.json 中项目版本为 7.0.1。
- **网络**：安装命令多从 GitHub 或各 harness 的插件市场拉取，国内网络环境可能需要更换镜像源或加速下载。

### 常见坑与注意事项

- **按 harness 分别安装**：原文明确说明，如果你使用多个 harness，需要为每个 harness 单独安装 Superpowers，不能共用一次安装。
- **OpenCode 需单独安装**：原文特别指出，即使你已在其他 harness 中使用 Superpowers，OpenCode 也要单独安装。
- **安装命令因 harness 而异**：不同 harness 的安装方式不同（有的走插件市场，有的从仓库安装），照抄其他 harness 的命令可能不适用。
- **更新方式不同**：部分 harness 提供单独的更新命令（如 `devin plugins update superpowers`、`gemini extensions update superpowers`），Antigravity 则用相同安装命令重装来更新；其余 harness 的更新方式原文未说明。

---



---

# Superpowers

Superpowers 是一套完整的软件开发方法论，专为你的编码智能体打造，构建在一组可组合的技能和初始指令之上，确保智能体会使用这些技能。

## 目录

- [工作原理](#工作原理)
- [商业服务](#商业服务)
- [快速开始](#安装)
  - [Claude Code](#claude-code)
  - [Antigravity](#antigravity)
  - [Codex App](#codex-app)
  - [Codex CLI](#codex-cli)
  - [Cursor](#cursor)
  - [Devin CLI](#devin-cli)
  - [Factory Droid](#factory-droid)
  - [Gemini CLI](#gemini-cli)
  - [GitHub Copilot CLI](#github-copilot-cli)
  - [Grok Build CLI](#grok-build-cli)
  - [Kimi Code](#kimi-code)
  - [OpenCode](#opencode)
  - [Pi](#pi)
  - [Qwen Code](#qwen-code)
  - [Hermes Agent](#hermes-agent)
  - [Muse](#muse)
- [基本工作流](#基本工作流)
- [出问题时怎么办](#出问题时怎么办)
- [社区](#社区)
- [内容一览](#内容一览)
- [理念](#理念)
- [贡献](#贡献)
- [更新](#更新)
- [许可证](#许可证)
- [可视化伴侣遥测](#可视化伴侣遥测)

## 工作原理

从你启动编码智能体的那一刻就开始了。一旦它发现你在构建某个东西，它*不会*直接跳进去写代码。相反，它会退后一步，问你真正想做什么。

一旦从对话中梳理出规格，它会分块展示给你，每块都短到能真正读完并消化。

在你确认设计后，智能体会整理出一份实现计划，清晰到足以让一位热情但品味差、缺乏判断力、不了解项目背景、且讨厌测试的初级工程师照着执行。它强调真正的红/绿 TDD、YAGNI（你不会需要它）和 DRY。

接下来，一旦你说"开始"，它会启动*子智能体驱动开发*流程，让智能体逐个完成每项工程任务，检查和审查它们的工作，然后继续推进。你的智能体常常能自主工作几个小时而不偏离你制定的计划。

还有很多细节，但这就是系统的核心。而且因为技能会自动触发，你不需要做任何特殊操作。你的编码智能体天生就拥有 Superpowers。

## 商业服务

如果你在企业中使用 Superpowers，并希望获得商业支持、额外工具或托管支出，欢迎随时联系我们：sales@primeradiant.com。

## 安装

安装方式因宿主环境而异。如果你使用多个宿主，请分别为每个宿主安装 Superpowers。

### Claude Code

Superpowers 可通过 [Claude 官方插件市场](https://claude.com/plugins/superpowers)获取。

#### 官方市场

- 从 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场提供 Superpowers 以及一些其他相关的 Claude Code 插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从该市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

从此仓库将 Superpowers 作为插件安装：

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity 会列出每个已安装技能的描述，而 `using-superpowers` 技能的描述会提示模型在对话开始时加载该技能。使用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过 [Codex 官方插件市场](https://github.com/openai/plugins)获取。

- 在 Codex 应用中，点击侧边栏的 Plugins。
- 你应该能在 Coding 区域看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按提示操作。

### Codex CLI

Superpowers 可通过 [Codex 官方插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或在插件市场中搜索 "superpowers"。

### Devin CLI

- 从此仓库安装插件：

  ```bash
  devin plugins install obra/superpowers
  ```

- 更新到最新版本：

  ```bash
  devin plugins update superpowers
  ```

### Factory Droid

- 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 安装扩展：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 稍后更新：

  ```bash
  gemini extensions update superpowers
  ```

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Grok Build CLI

Superpowers 可通过 [Grok 官方插件市场](https://github.com/xai-org/plugin-marketplace)获取。

- 从 xAI 官方市场安装插件：

  ```bash
  grok plugin install superpowers@xai-official --trust
  ```

- 或在 TUI 中打开市场，搜索 Superpowers 并安装：

  ```text
  /marketplace
  ```

### Kimi Code

Superpowers 已在 Kimi Code 的插件市场中提供。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 进入 `Marketplace` > `Superpowers` 并安装。

- 或直接从本仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用自己的插件安装方式；即使你已在其他宿主中使用 Superpowers，也需要单独安装。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

从此仓库将 Superpowers 作为 Pi 包安装：

```bash
pi install git:github.com/obra/superpowers
```

本地开发时，用此检出作为临时包运行 Pi：

```bash
pi -e /path/to/superpowers
```

Pi 包会加载 Superpowers 技能以及一个小扩展，该扩展在会话启动时和压缩后注入 `using-superpowers` 引导。Pi 原生支持技能，因此不需要兼容的 `Skill` 工具。子智能体和任务列表工具仍作为可选的 Pi 配套包。

### Qwen Code

Qwen Code 可直接从 Claude Code 市场安装插件。

- 从此仓库安装插件，并在提示时选择 `superpowers`：

  ```bash
  qwen extensions install obra/superpowers
  ```

- 稍后更新：

  ```bash
  qwen extensions update superpowers
  ```

### Hermes Agent

从此仓库将 Superpowers 作为 Hermes 插件安装：

```bash
hermes plugins install obra/superpowers --enable
```

安装后重启所有活跃的 Hermes 会话。注意：Hermes 没有压缩后钩子，因此如果超长会话在首轮之后发生压缩，会丢失引导——如果技能停止触发，请开启新会话。

### Muse

Superpowers 可作为原生 Muse 插件使用——同一仓库、同一技能、所有宿主通用。`using-superpowers` 引导通过原生 `SessionStart` 钩子注入，与 Claude Code、Codex、Cursor、Gemini、Pi 等一致——无需逐会话选择加入。

- 从本地检出安装：

  ```bash
  muse plugins install ./
  muse plugins approve superpowers
  ```

  或克隆后安装：

  ```bash
  git clone https://github.com/obra/superpowers.git
  muse plugins install ./superpowers
  muse plugins approve superpowers
  ```

- 稍后更新：

  ```bash
  muse plugins update superpowers
  ```

安装后重启所有活跃的 Muse 会话，使 `SessionStart` 钩子生效——技能会立即激活，钩子需在首次安装时批准。验证方法：开启新会话并发送 `Let's make a react todo list`——正常安装会在写任何代码之前自动触发 `brainstorming`。版本记录在 `.version-bump.json` 中，`scripts/bump-version.sh` 会保持同步。

## 基本工作流

1. **brainstorming** - 在构建任何东西之前激活。通过提问让你描述，从而梳理出你真正想要什么以及为什么，然后复述其理解供你纠正。小改动在聊天中达成一致；项目则产出一份构建者可以据此规划的书面设计文档。

2. **using-git-worktrees** - 设计批准后激活。在新分支上创建隔离工作区，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 设计批准后激活。将工作拆分为小任务（每个 2-5 分钟）。每个任务都有确切的文件路径、完整代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 有计划后激活。要么为每个任务派发新的子智能体并在每个任务后审查（最彻底），要么在当前会话中内联实现所有任务，最后对整个分支做一次全新审查（最经济）。

5. **test-driven-development** - 实现期间激活。强制执行 RED-GREEN-REFACTOR：写失败的测试，看它失败，写最小代码，看它通过，提交。删除在测试之前写的代码。

6. **requesting-code-review** - 任务之间激活。对照计划审查，按严重程度报告问题。关键问题会阻塞进度。

7. **finishing-a-development-branch** - 任务完成时激活。验证测试，提供选项（合并/PR/保留/丢弃），清理工作树。

**智能体在任何任务之前都会检查相关技能。** 这是强制工作流，不是建议。

## 出问题时怎么办

有时会话会出问题：技能在不该触发时触发，该触发时却沉默，或者智能体忽略计划、重复工作、消耗的 token 超出预期。让你的编码智能体"figure out what went wrong with superpowers in this session"，它就会调用 **diagnosing-superpowers** 技能。要检查更早的会话，请指明："figure out what went wrong with superpowers in session `<id>`"。

该技能会读取会话记录，以行级证据报告发生了什么，如果你需要，还会打包一份脱敏的 bug 报告包。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的伙伴们构建。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz)，获取社区支持、提问并分享你用 Superpowers 构建的东西
- **Issues**：https://github.com/obra/superpowers/issues
- **发布公告**：[订阅](https://primeradiant.com/superpowers/)以获取新版本通知

## 内容一览

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因流程（包含根因追踪、纵深防御、基于条件的等待技术）
- **verification-before-completion** - 确保确实修复了
- **diagnosing-superpowers** - 用证据查明会话中出了什么问题；导出脱敏包或提交 Issue

**协作**
- **brainstorming** - 在构建前梳理意图
- **writing-plans** - 详细的实现计划
- **executing-plans** - 内联计划执行：一个上下文，一次最终审查
- **dispatching-parallel-agents** - 并发子智能体工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 两阶段审查的快速迭代（先规格合规，再代码质量）

**元技能**
- **writing-skills** - 按最佳实践创建新技能（包含测试方法论）
- **using-superpowers** - 技能系统介绍

## 理念

- **测试驱动开发** - 始终先写测试
- **系统化而非临时应对** - 流程优于猜测
- **降低复杂度** - 以简单为首要目标
- **证据优于声明** - 宣布成功前先验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

Superpowers 的通用贡献流程如下。请注意，我们通常不接受新技能的贡献，且任何技能更新都必须在我们支持的所有编码智能体上可用。我们也不接受智能体编写的、用于修复他人提交的 Issue 的 PR，除非维护者要求，即使你遇到了同样的问题；请将你的报告添加到现有 Issue 中。

1. Fork 仓库
2. 切换到 'dev' 分支
3. 为你的工作创建分支
4. 遵循 `writing-skills` 技能来创建和测试新技能及修改后的技能
5. 提交 PR，填写 Pull Request 模板的每一部分，包括由人用自己的话写的摘要。

技能行为测试使用 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 的 drill 评估框架，克隆到 `evals/`——设置见 `evals/README.md`。插件基础设施测试位于 `tests/`，通过相关的 `run-*.sh` 或 `npm test` 运行。

完整指南见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新在一定程度上取决于编码智能体，但通常是自动的。

## 许可证

MIT 许可证 - 详情见 LICENSE 文件

## 可视化伴侣遥测

由于技能和插件不会向创建者提供任何反馈，我们不知道有多少人在使用 Superpowers。默认情况下，brainstorming 可选可视化伴侣功能上的 Prime Radiant 标志从我们的网站加载。它包含正在使用的 Superpowers 版本。它不包含任何关于你的项目、提示或编码智能体的细节。我们看不到你的点击或你正在构建的任何内容。这有助于我们大致了解有多少人在使用 Superpowers 以及使用的是哪个版本。这 100% 是可选的。要禁用，请将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设为任意真值。Superpowers 也遵循 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 退出选项。