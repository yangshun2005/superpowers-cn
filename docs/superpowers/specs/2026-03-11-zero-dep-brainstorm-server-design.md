# 零依赖头脑风暴服务器

用单个零依赖的 `server.js`（仅使用 Node.js 内置模块）替换头脑风暴配套服务器中内置的 node_modules（express、ws、chokidar —— 714 个受跟踪文件）。

## 动机

将 node_modules 内置到 git 仓库会带来供应链风险：冻结的依赖无法获得安全补丁，714 个第三方代码文件未经审计即被提交，对内置代码的修改看起来像普通提交。虽然实际风险较低（仅限 localhost 的开发服务器），但消除它并不困难。

## 架构

单个 `server.js` 文件（约 250-300 行），使用 `http`、`crypto`、`fs` 和 `path`。该文件承担两种角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被 require 时**（`require('./server.js')`）：导出 WebSocket 协议函数供单元测试使用

### WebSocket 协议

仅针对文本帧实现 RFC 6455：

**握手：** 使用 SHA-1 + RFC 6455 魔术 GUID，从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种带掩码的长度编码：
- 小：载荷 < 126 字节
- 中：126-65535 字节（16 位扩展）
- 大：> 65535 字节（64 位扩展）

使用 4 字节掩码键对载荷进行 XOR 去掩码。返回 `{ opcode, payload, bytesConsumed }`，缓冲区不完整时返回 `null`。拒绝未掩码的帧。

**帧编码（服务器到客户端）：** 未掩码帧，使用相同的三种长度编码。

**处理的 Opcode：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。无法识别的 opcode 会收到状态码 1003（Unsupported Data）的关闭帧。

**有意跳过：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。localhost 客户端之间的小型 JSON 文本消息不需要这些。扩展和子协议在握手中协商 —— 不声明它们，它们就永远不会启用。

**缓冲区累积：** 每个连接维护一个缓冲区。收到 `data` 时追加，并循环调用 `decodeFrame` 直到返回 null 或缓冲区为空。

### HTTP 服务器

三个路由：

1. **`GET /`** —— 按 mtime 提供 screen 目录中最新的 `.html`。检测完整文档与片段，将片段包装进 frame 模板，注入 helper.js。返回 `text/html`。当不存在 `.html` 文件时，提供一个硬编码的等待页面（"Waiting for Claude to push a screen..."），并注入 helper.js。
2. **`GET /files/*`** —— 从 screen 目录提供静态文件，MIME 类型从硬编码的扩展名映射（html、css、js、png、jpg、gif、svg、json）中查找。未找到则返回 404。
3. **其他所有请求** —— 404。

WebSocket 升级通过 HTTP 服务器上的 `'upgrade'` 事件处理，与请求处理器分离。

### 配置

环境变量（均为可选）：

- `BRAINSTORM_PORT` —— 绑定的端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` —— 绑定的接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` —— 启动 JSON 中 URL 的主机名（默认：当 host 为 `127.0.0.1` 时为 `localhost`，否则与 host 相同）
- `BRAINSTORM_DIR` —— screen 目录路径（默认：`/tmp/brainstorm`）

### 启动顺序

1. 如果 `SCREEN_DIR` 不存在则创建（`mkdirSync` 递归）
2. 从 `__dirname` 加载 frame 模板和 helper.js
3. 在配置的 host/port 上启动 HTTP 服务器
4. 在 `SCREEN_DIR` 上启动 `fs.watch`
5. 监听成功后，向 stdout 输出 `server-started` JSON：`{ type, port, host, url_host, url, screen_dir }`
6. 将相同的 JSON 写入 `SCREEN_DIR/.server-info`，以便 stdout 被隐藏时（后台执行）代理能找到连接详情

### 应用层 WebSocket 消息

当客户端发来 TEXT 帧时：

1. 解析为 JSON。解析失败则记录到 stderr 并继续。
2. 以 `{ source: 'user-event', ...event }` 记录到 stdout。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每行一个事件）。

### 文件监听

`fs.watch(SCREEN_DIR)` 替代 chokidar。HTML 文件事件：

- 新文件（存在文件的 `rename` 事件）：若存在则删除 `.events` 文件（`unlinkSync`），以 JSON 记录 `screen-added` 到 stdout
- 文件变更（`change` 事件）：以 JSON 记录 `screen-updated` 到 stdout（不清除 `.events`）
- 两种事件：向所有已连接的 WebSocket 客户端发送 `{ type: 'reload' }`

按文件名去抖，约 100ms 超时，防止重复事件（在 macOS 和 Linux 上常见）。

### 错误处理

- WebSocket 客户端的畸形 JSON：记录到 stderr，继续
- 未处理的 opcode：以状态码 1003 关闭
- 客户端断开：从广播集合中移除
- `fs.watch` 错误：记录到 stderr，继续
- 无优雅关闭逻辑 —— shell 脚本通过 SIGTERM 处理进程生命周期

## 变更内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单文件） |
| express、ws、chokidar 依赖 | 无 |
| 无静态文件服务 | `/files/*` 从 screen 目录提供 |

## 保持不变

- `helper.js` —— 无变更
- `frame-template.html` —— 无变更
- `start-server.sh` —— 一行更新：`index.js` 改为 `server.js`
- `stop-server.sh` —— 无变更
- `visual-companion.md` —— 无变更
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台的 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上对单个扁平目录可靠
- Shell 脚本需要 bash（Windows 上为 Git Bash，Claude Code 需要它）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require `server.js` 的导出，直接测试 WebSocket 帧编码/解码、握手计算和协议边界情况。

**集成测试**（`server.test.js`）：测试完整服务器行为 —— HTTP 服务、WebSocket 通信、文件监听、头脑风暴工作流。使用 `ws` npm 包作为仅测试用的客户端依赖（不交付给最终用户）。