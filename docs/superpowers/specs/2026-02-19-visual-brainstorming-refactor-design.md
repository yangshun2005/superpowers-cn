# 视觉头脑风暴重构：浏览器展示，终端命令

**日期：** 2026-02-19
**状态：** 已批准
**范围：** `lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/`

## 问题

视觉头脑风暴期间，Claude 将 `wait-for-feedback.sh` 作为后台任务运行，并阻塞在 `TaskOutput(block=true, timeout=600s)` 上。这完全占用了 TUI——视觉头脑风暴运行时用户无法向 Claude 输入。浏览器成为唯一的输入通道。

Claude Code 的执行模型是回合制的。Claude 无法在单个回合内同时监听两个通道。阻塞式 `TaskOutput` 模式是错误的原语——它模拟了平台并不支持的事件驱动行为。

## 设计

### 核心模型

**浏览器 = 交互式展示。** 展示原型图，让用户点击选择选项。选择结果记录在服务端。

**终端 = 对话通道。** 始终不阻塞，始终可用。用户在这里与 Claude 交流。

### 循环

1. Claude 将 HTML 文件写入会话目录
2. 服务端通过 chokidar 检测到文件，向浏览器推送 WebSocket 重载（不变）
3. Claude 结束回合——告诉用户查看浏览器并在终端中回复
4. 用户查看浏览器，可选地点击选择选项，然后在终端中输入反馈
5. 下一回合，Claude 读取 `$SCREEN_DIR/.events` 获取浏览器交互流（点击、选择），与终端文本合并
6. 迭代或推进

无后台任务。无 `TaskOutput` 阻塞。无轮询脚本。

### 关键删除：`wait-for-feedback.sh`

完全删除。其目的是桥接"服务端将事件记录到 stdout"和"Claude 需要接收这些事件"。`.events` 文件取代了它——服务端直接写入用户交互事件，Claude 使用平台提供的任意文件读取机制读取它们。

### 关键新增：`.events` 文件（每屏事件流）

服务端将所有用户交互事件写入 `$SCREEN_DIR/.events`，每行一个 JSON 对象。这为 Claude 提供了当前屏幕的完整交互流——不仅是最终选择，还有用户的探索路径（点击 A，然后 B，最终选定 C）。

用户探索选项后的示例内容：

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 屏幕内仅追加。每个用户事件作为新行追加。
- 当 chokidar 检测到新 HTML 文件（推送新屏幕）时，文件被清除（删除），防止过期事件残留。
- 如果 Claude 读取时文件不存在，说明没有浏览器交互——Claude 仅使用终端文本。
- 文件仅包含用户事件（`click` 等）——不包含服务端生命周期事件（`server-started`、`screen-added`）。这使其保持精简和聚焦。
- Claude 可以读取完整流以理解用户的探索模式，或仅查看最后一个 `choice` 事件获取最终选择。

## 按文件变更

### `index.js`（服务端）

**A. 将用户事件写入 `.events` 文件。**

在 WebSocket `message` 处理器中，将事件记录到 stdout 后：通过 `fs.appendFileSync` 将事件作为 JSON 行追加到 `$SCREEN_DIR/.events`。仅写入用户交互事件（`source: 'user-event'` 的事件），不写入服务端生命周期事件。

**B. 新屏幕时清除 `.events`。**

在 chokidar `add` 处理器中（检测到新 `.html` 文件），删除 `$SCREEN_DIR/.events`（如果存在）。这是明确的"新屏幕"信号——比在 GET `/` 时清除更好，后者每次重载都会触发。

**C. 替换 `wrapInFrame` 内容注入。**

当前正则锚定在 `<div class="feedback-footer">` 上，该元素正在被移除。替换为注释占位符：移除 `#claude-content` 内的现有默认内容（`<h2>Visual Brainstorming</h2>` 和副标题段落），替换为单个 `<!-- CONTENT -->` 标记。内容注入变为 `frameTemplate.replace('<!-- CONTENT -->', content)`。更简单，且模板格式变化时不会中断。

### `frame-template.html`（UI 框架）

**移除：**
- `feedback-footer` div（textarea、Send 按钮、label、`.feedback-row`）
- 相关 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`、其中的 textarea 和按钮样式）

**新增：**
- `#claude-content` 内的 `<!-- CONTENT -->` 占位符，替换默认文本
- 原 footer 位置的选中指示栏，有两种状态：
  - 默认："点击上方选项，然后返回终端"
  - 选中后："已选择选项 B——返回终端继续"
- 指示栏的 CSS（低调，视觉权重与现有 header 相近）

**保持不变：**
- 带有 "Brainstorm Companion" 标题和连接状态的 header 栏
- `.main` 包装器和 `#claude-content` 容器
- 所有组件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、占位符、mock 元素）
- 深色/浅色主题变量和媒体查询

### `helper.js`（客户端脚本）

**移除：**
- `sendToClaude()` 函数和"Sent to Claude"页面接管
- `window.send()` 函数（与已移除的 Send 按钮绑定）
- 表单提交处理器——没有反馈 textarea 就没有用途，只会增加日志噪音
- 输入变更处理器——原因相同
- `pageshow` 事件监听器（原本用于修复 textarea 持久化——已无 textarea）

**保留：**
- WebSocket 连接、重连逻辑、事件队列
- 重载处理器（服务端推送时 `window.location.reload()`）
- `window.toggleSelect()` 用于选择高亮
- `window.selectedChoice` 跟踪
- `window.brainstorm.send()` 和 `window.brainstorm.choice()`——这些与已移除的 `window.send()` 不同。它们调用 `sendEvent`，通过 WebSocket 记录到服务端。对自定义完整文档页面有用。

**收窄：**
- 点击处理器：仅捕获 `[data-choice]` 点击，而非所有按钮/链接。宽泛捕获在浏览器作为反馈通道时需要；现在仅用于选择跟踪。

**新增：**
- 在 `data-choice` 点击时，更新选中指示栏文本以显示所选选项。

**从 `window.brainstorm` API 移除：**
- `brainstorm.sendToClaude`——不再存在

### `visual-companion.md`（技能说明）

**重写"The Loop"部分**为上述非阻塞流程。移除所有对以下内容的引用：
- `wait-for-feedback.sh`
- `TaskOutput` 阻塞
- 超时/重试逻辑（600s 超时、30 分钟上限）
- 描述 `send-to-claude` JSON 的"User Feedback Format"部分

**替换为：**
- 新循环（写 HTML → 结束回合 → 用户在终端回复 → 读取 `.events` → 迭代）
- `.events` 文件格式文档
- 指导说明：终端消息是主要反馈；`.events` 提供完整的浏览器交互流作为额外上下文

**保留：**
- 服务端启动/关闭说明
- 内容片段与完整文档的指导
- CSS 类参考和可用组件
- 设计技巧（按问题调整保真度、每屏 2-4 个选项等）

### `wait-for-feedback.sh`

**完全删除。**

### `tests/brainstorm-server/server.test.js`

需要更新的测试：
- 断言片段响应中存在 `feedback-footer` 的测试——更新为断言选中指示栏或 `<!-- CONTENT -->` 替换
- 断言 `helper.js` 包含 `send` 的测试——更新以反映收窄的 API
- 断言 `sendToClaude` CSS 变量使用的测试——移除（函数不再存在）

## 平台兼容性

服务端代码（`index.js`、`helper.js`、`frame-template.html`）完全平台无关——纯 Node.js 和浏览器 JavaScript。无 Claude Code 特定引用。已证明可通过后台终端交互在 Codex 上运行。

技能说明（`visual-companion.md`）是平台适配层。各平台的 Claude 使用自己的工具启动服务端、读取 `.events` 等。非阻塞模型天然跨平台工作，因为它不依赖任何平台特定的阻塞原语。

## 这带来了什么

- **视觉头脑风暴期间 TUI 始终响应**
- **混合输入**——浏览器点击 + 终端输入，自然合并
- **优雅降级**——浏览器宕机或用户未打开？终端仍然可用
- **更简单的架构**——无后台任务、无轮询脚本、无超时管理
- **跨平台**——相同的服务端代码可在 Claude Code、Codex 及任何未来平台上运行

## 这放弃了什么

- **纯浏览器反馈工作流**——用户必须返回终端才能继续。选中指示栏会引导他们，但相比旧的点击 Send 并等待流程多了一步。
- **浏览器内联文本反馈**——textarea 已移除。所有文本反馈通过终端进行。这是有意的——终端是比框架中小 textarea 更好的文本输入通道。
- **浏览器 Send 的即时响应**——旧系统在用户点击 Send 的瞬间 Claude 就响应。现在用户切换到终端期间存在间隙。实际中只有几秒，且用户可以在终端消息中补充上下文。