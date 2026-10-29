# 可视化伴侣认证加固设计

**日期：** 2026-06-10
**状态：** 供 Drew 审阅的草案

## 目标

修复 PR #1720 头脑风暴可视化伴侣中发现的安全与可靠性缺口，不改变伴侣的核心工作流，也不新增运行时依赖。

修复必须测试先行，并留下明确的自动化证据证明：

- 跨源浏览器标签页无法借 Cookie 注入伴侣事件
- 重启重连不依赖浏览器 Cookie 行为也能工作
- 引导后 Bearer 密钥不再留在可见 URL 中
- `/files/*` 无法提供内容目录之外的文件
- 未来同源内置 UI 库仍可正常工作

## 威胁模型

伴侣为单次头脑风暴会话提供 agent 生成的本地 UI。重要资产包括：

- 伴侣提供的屏幕内容
- 会话密钥
- `state/events`，agent 将其作为用户反馈读取
- 伴侣会话目录下的本地文件

范围内的攻击者：

- 另一个 `localhost` 端口上的恶意浏览器标签页
- 能向伴侣发起请求、但不应能以伴侣 UI 身份认证的浏览器页面
- 服务器绑定到非回环接口时的直接远程客户端
- 通过 URL 历史、Referer 或已提交的本地状态意外泄露
- 逃逸 `/files/*` 的内容目录符号链接或路径技巧

本次修复的范围外：

- 恶意 agent 编写的屏幕 HTML
- 伴侣屏幕加载的恶意同源内置 JavaScript

此范围外边界是有意为之。伴侣屏幕是 agent UI 表面的一部分。它们今天可能使用内联脚本，未来可能使用 Alpine 或 Three.js 等同源内置库。防御恶意屏幕 HTML 需要更大的沙箱 iframe 架构加窄消息桥；这不在本次 PR 加固的范围内。

## 当前故障

自动化与有头浏览器测试在 PR 分支中发现以下故障：

1. 真实伴侣页面设置 Cookie 后，跨源 localhost 页面可打开经 Cookie 认证的 WebSocket，并向 `state/events` 写入攻击者控制的选项。
2. `/files/*` 提供指向 `content/` 之外的符号链接，包括指向含密钥 URL 的 `state/server-info` 的符号链接。
3. 会话密钥仍留在实际屏幕页面的 URL 中，同源屏幕 JavaScript 及意外的 Referer/历史记录可见。
4. 辅助脚本用无密钥的 `ws://host` URL 重连。在有头 Chrome 中，同端口/同令牌重启后，浏览器不再向重启后的服务器出示 Cookie，打开的标签页卡在墓碑页直到手动刷新。
5. Shell lint 与生命周期测试需清理，使测试在 Codex 中稳定通过。

## 设计

### 1. 引导式带密钥加载

`GET /?key=<token>` 变为引导响应，而非屏幕响应。

密钥有效时，服务器：

1. 照旧设置 HttpOnly 会话 Cookie
2. 返回一个小型 HTML 引导页
3. 引导页将密钥存入标签页作用域的 `sessionStorage`
4. 引导页用 `location.replace('/')` 导航到 `/`

此后，可见屏幕 URL 为裸 `/`，而非 `/?key=...`。

带有效 Cookie 的 `GET /` 提供当前屏幕。无有效 Cookie 的 `GET /` 仍返回友好的 403 页面。`GET /?key=<wrong>` 返回 403。

为何用 `sessionStorage`：辅助脚本需要一个能跨同端口重启存活、且不只依赖 Cookie 行为的重连凭据。由于屏幕 HTML 是可信的同源 UI，在此威胁模型下将密钥存入标签页作用域存储是可接受的。这明显优于将密钥留在地址栏、历史记录和 Referer 表面。

### 2. WebSocket 同源强制

WebSocket 升级必须通过两项检查：

1. 通过查询密钥或 Cookie 的有效会话认证
2. 若存在 `Origin` 头，必须与请求目标源匹配

源检查应比较：

```text
Origin === "http://" + req.headers.host
```

浏览器攻击者页面示例：

```text
Origin: http://localhost:9999
Host: localhost:58088
```

即使浏览器发送伴侣 Cookie，也必须拒绝。

合法伴侣页面示例：

```text
Origin: http://localhost:58088
Host: localhost:58088
```

密钥或 Cookie 有效时应接受。

直接的非浏览器客户端可省略 `Origin`；它们仍需会话密钥。

### 3. 辅助脚本重连凭据

`helper.js` 应从 `sessionStorage` 读取标签页作用域密钥，并附加到 WebSocket URL：

```text
ws://<host>/?key=<stored-key>
```

若无存储密钥，辅助脚本回退到当前仅 Cookie 的 `ws://<host>` 行为。这为已加载、有有效 Cookie 但无存储条目的页面保留兼容性。

### 4. `/files/*` 遏制

文件服务器应继续拒绝空名称和点文件。还必须确保文件是 `CONTENT_DIR` 内的真实常规文件。

以 realpath 遏制为边界：

- 计算 `realContentDir = fs.realpathSync(CONTENT_DIR)`
- 计算 `realFilePath = fs.realpathSync(filePath)`
- 仅当 `realFilePath` 等于 `realContentDir` 的后代时才提供
- 拒绝符号链接及内容目录之外的任何内容，返回 404

服务器应继续使用 `path.basename`，因此仍不支持嵌套路径。

### 5. 减少泄露的响应头

添加不阻止内联脚本或未来同源内置库的保守响应头：

```text
Referrer-Policy: no-referrer
Cache-Control: no-store
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Cross-Origin-Resource-Policy: same-origin
```

本次不添加限制性 `script-src` CSP。伴侣当前注入内联辅助 JavaScript，未来屏幕可能加载同源内置库。

### 6. Gitignore 持久会话状态

将 `.superpowers/` 加入仓库根 `.gitignore`，以免使用 `--project-dir` 时持久化的伴侣状态和 `.last-token` 被意外提交。

### 7. 测试稳定性与 Lint

清理所触及的 start/stop 脚本中的 shell lint 警告。

更新调用 `start-server.sh --idle-timeout-minutes` 的生命周期测试，使其不会在 Codex 的 `CODEX_CI` 前台自动检测下挂起。当测试期望脚本返回启动 JSON 时，应使用 `--background` 强制后台模式。

## 测试策略

所有行为变更应遵循 TDD：

1. 编写失败的聚焦测试
2. 运行并确认因预期原因失败
3. 实现最小修复
4. 重跑聚焦测试
5. 重跑完整 brainstorm-server 套件

必需的聚焦回归：

- 有效带密钥的 `/` 返回引导页，而非屏幕内容
- 引导页将密钥存入 `sessionStorage` 并剥离 URL
- 仅 Cookie 的 `/` 仍提供屏幕内容
- 辅助脚本使用 `sessionStorage` 密钥构造 WebSocket URL
- 同源 Cookie WebSocket 可打开
- 跨源 Cookie WebSocket 被拒绝且不写入事件
- 直接密钥 WebSocket 无 `Origin` 仍可打开
- `content/` 下指向 `state/server-info` 的符号链接返回 404
- 普通 HTML、引导页、403 和文件响应均带安全响应头
- 同端口/同令牌重启可用存储密钥认证重连
- 所触及 shell 脚本通过 shell lint
- 生命周期套件在 Codex 下不挂起

## 验收标准

- `cd tests/brainstorm-server && npm test` 反复通过且不挂起。
- 此前从另一 localhost 源写入 `attacker-injected` 的安全探针现在无法打开 WebSocket，且 `state/events` 保持不变。
- 指向 `server-info` 的符号链接探针返回 404。
- 有头或无头浏览器的带密钥加载最终停在裸 `/` URL，状态胶囊达到 Connected。
- 同端口/同令牌重启自动重连，无需手动刷新。
- `scripts/lint-shell.sh` 对所触及 shell 脚本通过。

## 推迟的工作

若项目日后需将屏幕 HTML 视为不可信，应设计独立的沙箱 iframe 架构。该架构应将生成的屏幕隔离在独立源或沙箱框架中，仅暴露窄 `postMessage` 桥用于用户选择。不要将其并入本次修复。