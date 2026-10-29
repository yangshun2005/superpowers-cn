# 可视化伴侣指南

基于浏览器的可视化头脑风暴伴侣，用于展示线框图、图表和选项。

## 何时使用

按问题逐个决定，而非按会话决定。判断标准：**用户看到它是否比读到它理解得更好？**

**当内容本身是视觉性的，使用浏览器：**

- **UI 线框图** — 线框、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉对比** — 对比两种布局、两种配色方案、两个设计方向
- **设计打磨** — 当问题涉及外观与感觉、间距、视觉层次时
- **空间关系** — 状态机、流程图、以图表呈现的实体关系

**当内容是文本或表格性的，使用终端：**

- **需求与范围问题** — "X 是什么意思？"、"哪些功能在范围内？"
- **概念性 A/B/C 选择** — 在文字描述的方案之间做选择
- **权衡列表** — 优缺点、对比表格
- **技术决策** — API 设计、数据建模、架构方案选择
- **澄清性问题** — 任何答案是文字而非视觉偏好的问题

*关于* UI 主题的问题不自动就是视觉问题。"你想要哪种向导？"是概念性的——用终端。"这些向导布局中哪个感觉对？"是视觉性的——用浏览器。

## 工作原理

服务器监视一个目录中的 HTML 文件，并将最新的一个提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到它并可点击选择选项。选择结果记录到 `state_dir/events`，你在下一轮读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供（仅注入辅助脚本）。否则，服务器自动将你的内容包裹进框架模板——添加页头、CSS 主题、连接状态和所有交互基础设施。**默认写内容片段。** 只有需要完全控制页面时才写完整文档。

## 启动会话

```bash
# Start AFTER the user approves the companion. --open auto-opens their browser on
# the first screen; --project-dir persists mockups and enables same-port restart.
bash scripts/start-server.sh --project-dir /path/to/project --open

# Returns: {"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

保存响应中的 `screen_dir` 和 `state_dir`。使用 `--open` 时，浏览器会在你推送第一个屏幕时自动打开——你无需让用户打开它，但仍应分享 URL 作为后备（无头/远程环境不会自动打开）。

**URL 包含会话密钥（`?key=…`）。** 服务器拒绝任何不带密钥的请求，因此始终给用户 `url` 字段中的**完整** URL——绝不要去掉查询字符串，也绝不要给出裸的 `http://host:port`。密钥控制 HTTP 和 WebSocket 访问，使无关的浏览器标签页或网络上的其他机器无法读取屏幕或注入事件。首次加载后浏览器通过 cookie 记住密钥，因此刷新和 `/files/*` 资源无需重复携带。

**查找连接信息：** 服务器将启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动服务器且未捕获 stdout，读取该文件获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 下的会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传入，使线框图持久保存在 `.superpowers/brainstorm/` 中并在服务器重启后保留。不传则文件进入 `/tmp` 并被清理。提醒用户将 `.superpowers/` 加入 `.gitignore`（如果尚未加入）。

**按平台启动服务器：**

**Claude Code：**
```bash
# Default mode works — the script backgrounds the server itself.
bash scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本自动检测并切换到前台模式（会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，使服务器跨对话轮次存活，然后在下一轮读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex：**
```bash
# Codex reaps background processes. The script auto-detects CODEX_CI and
# switches to foreground mode. Run it normally — no extra flags needed.
bash scripts/start-server.sh --project-dir /path/to/project --open
```

**Gemini CLI：**
```bash
# Use --foreground and set is_background: true on your shell tool call
# so the process survives across turns
bash scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**Copilot CLI：**
```bash
# Start it with Copilot CLI's non-blocking/background shell mechanism so the
# server survives across turns. Keep --foreground so the harness, not the
# script, owns backgrounding. The launcher is a .sh, so invoke it via bash
# (on Windows, call Git Bash's bash.exe from the PowerShell tool).
bash scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在对话轮次间持续在后台运行。如果你的环境会回收分离的进程，使用 `--foreground` 并通过平台的背景执行机制启动命令。

如果浏览器无法访问该 URL（远程/容器化环境常见），绑定非回环主机：

```bash
bash scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环流程

1. **检查服务器存活**，然后**写 HTML** 到 `screen_dir` 中的新文件：
   - **必需：在引用 URL 或推送屏幕前确认服务器存活。** 检查 `$STATE_DIR/server-info` 存在且 `$STATE_DIR/server-stopped` 不存在。如果已关闭，用 `start-server.sh` 以**相同的 `--project-dir`** 重启——它复用同一端口，用户已打开的标签页会自动重连（服务器关闭期间显示"已暂停"覆盖层），你无需发送新 URL。服务器空闲 4 小时后自动退出（可通过 `--idle-timeout-minutes` 配置）。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **绝不复用文件名** — 每个屏幕用新文件
   - 使用你的文件创建工具——**绝不要用 cat/heredoc**（会向终端倾倒噪音）
   - 服务器自动提供最新文件

2. **告诉用户预期并结束你的回合：**
   - 提醒他们 URL（每一步都提醒，不只是第一次）
   - 简要文字总结屏幕内容（例如"展示首页的 3 种布局选项"）
   - 请他们在终端回复："看一下并告诉我你的想法。如果愿意，点击选择某个选项。"

3. **在你的下一轮** — 用户在终端回复后：
   - 如果 `$STATE_DIR/events` 存在则读取——包含用户的浏览器交互（点击、选择），以 JSON 行形式
   - 与用户的终端文字合并以获得完整图景
   - 终端消息是主要反馈；`state_dir/events` 提供结构化交互数据

4. **迭代或推进** — 如果反馈改变了当前屏幕，写新文件（如 `layout-v2.html`）。仅当当前步骤验证后才进入下一个问题。

5. **返回终端时卸载** — 当下一步不需要浏览器时（如澄清性问题、权衡讨论），推送等待屏幕以清除陈旧内容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这防止用户在对话已推进时仍盯着已解决的选择。当下一个视觉问题出现时，照常推送新的内容文件。

6. 重复直到完成。

## 编写内容片段

只写页面内部的内容。服务器自动将其包裹进框架模板（页头、主题 CSS、连接状态和所有交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就这样。无需 `<html>`、CSS 或 `<script>` 标签。服务器提供所有这些。

## 可用的 CSS 类

框架模板为你的内容提供以下 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 给容器添加 `data-multiselect` 让用户选择多个选项。每次点击切换该项的选中样式。

```html
<div class="options" data-multiselect>
  <!-- same option markup — users can select/deselect multiple -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### 线框图容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### 分屏视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### 优缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### 线框元素（线框图构建块）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版与区块

- `h2` — 页面标题
- `h3` — 区块标题
- `.subtitle` — 标题下方的次要文字
- `.section` — 带底部外边距的内容块
- `.label` — 小号大写标签文字

## 浏览器事件格式

当用户在浏览器中点击选项时，其交互记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。推送新屏幕时该文件自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整事件流展示用户的探索路径——他们可能在最终确定前点击多个选项。最后的 `choice` 事件通常是最终选择，但点击模式可能揭示犹豫或值得询问的偏好。

如果 `$STATE_DIR/events` 不存在，用户未与浏览器交互——仅使用其终端文字。

## 设计提示

- **保真度与问题匹配** — 布局问题用线框图，打磨问题用精细设计
- **每页解释问题** — "哪种布局感觉更专业？"而非仅"选一个"
- **推进前先迭代** — 如果反馈改变当前屏幕，写新版本
- **每屏最多 2-4 个选项**
- **重要时使用真实内容** — 摄影作品集用真实图片（Unsplash）。占位内容会掩盖设计问题。
- **保持线框图简单** — 聚焦布局和结构，而非像素级完美设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 绝不复用文件名 — 每个屏幕必须是新文件
- 迭代时：追加版本后缀如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
bash scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，线框图文件保留在 `.superpowers/brainstorm/` 中供后续参考。只有 `/tmp` 会话在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`