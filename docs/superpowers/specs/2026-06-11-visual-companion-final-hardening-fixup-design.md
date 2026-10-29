# Visual Companion 最终加固修复设计

**日期：** 2026-06-11
**状态：** 供 Drew 审阅的草案

## 目标

完成 PR #1720 的 visual companion 加固工作，使分支达到可供 Jesse 审阅的状态：安全行为干净、测试确定性、PR diff 仅包含 companion 相关工作。

这是对现有认证加固设计的修复，不应重新设计 companion 或扩展功能范围。

## 背景

上一轮加固加入了带密钥的会话、同源 WebSocket 检查、URL 密钥剥离、`/files/*` 路径限制、减少泄漏的响应头、IPv6 URL 格式化、Windows 生命周期覆盖以及 PR 证据更新。

最终审阅发现五个遗留问题：

1. 根路径 `GET /` 的屏幕选择逻辑仍可能提供 `content/` 下指向内容目录之外的符号链接或硬链接。
2. 首选端口被占用时，回退服务器可能复用已持久化的 `.last-token`，导致同一项目出现两个使用相同 bearer 密钥的活跃 companion 服务器。
3. 无法提供强所有权证明时，`stop-server.sh` 可能向无关的 `node server.cjs` 进程发送信号。
4. 部分测试可能误连到错误的回退进程、失败时泄漏后台进程，或假定类 Windows 主机支持符号链接。
5. PR 当前存在冲突，因为分支包含一个较早的 `evals` 子模块升级，该升级已单独处理。

## 非目标

- 本轮不添加 HTTPS 隧道或 `wss://` 源语义。
- 不实现 opt-out、自由文本或对比度辅助等 companion 功能。
- 不内置 Alpine、Three.js 或任何其他 JavaScript 库。
- 不尝试沙箱化恶意 agent 编写的屏幕 HTML。
- 不为过期的 stop-server PID 文件添加向后兼容，除非 Drew 明确批准该取舍。

## 继承的安全不变量

本修复保留已设计并实现的认证加固：

- `.last-token` 和 `state/server-info` 仍为仅所有者可访问的敏感状态。
- 回退令牌可出现在启动 JSON 和 `state/server-info` 中，但不得写入 `.last-token`。
- Cookie 仍按端口命名、`HttpOnly`、`SameSite=Strict`，作用域为 `/`。
- WebSocket 升级仍需有效密钥或 Cookie。
- 浏览器提供 `Origin` 头时，仍强制执行 WebSocket `Origin` 检查。
- 无 `Origin` 的直接客户端仅在携带会话密钥时允许。
- 生成的同源屏幕 JavaScript 及未来同源内置库视为可信。沙箱化恶意屏幕 HTML 仍推迟。

## 设计

### 1. 变基到当前 `dev`

在实施前将 `brainstorming-companion` 变基到当前 `origin/dev`。通过采用 `dev` 解决 `evals` 子模块冲突。

变基后：

- `evals` 不得出现在 PR diff 中。
- PR #1720 仍可提及在其他地方运行的 eval 证据，但必须包含确切的外部证据：eval 仓库提交、场景路径、命令、结果产物路径或 id，以及 RED/GREEN 结果。
- PR 正文不得暗示 evals 子模块升级属于本 PR。
- 任何暗示包含子模块升级的早期 PR 正文或评论，必须被最终 PR 正文证据取代。

### 2. 根屏幕路径限制

根屏幕路由必须使用与 `/files/*` 相同的限制边界。

`getNewestScreen()` 应忽略任何未通过"内容目录内常规文件"守卫的 `.html` 候选。该守卫必须解析真实路径，确保所提供文件位于 `CONTENT_DIR` 内。平台报告链接计数时，还必须拒绝链接计数不为 1 的文件，以保留现有硬链接保护。

预期行为：

- `content/` 下指向 `content/` 之外的符号链接被忽略。
- `fs.linkSync` 成功且 `lstat.nlink > 1` 时，`content/` 下指向 `state/server-info` 的硬链接被忽略。
- 若无安全屏幕文件，则提供等待页面。
- 现有 `/files/*` 限制行为不变：空名称、点文件、符号链接、硬链接和目录仍返回 404。

### 3. 回退令牌隔离

端口回退不得复用从持久化 `.last-token` 加载的令牌。

令牌来源应在代码中显式区分：

- 环境变量 `BRAINSTORM_TOKEN` 是操作者/测试的有意覆盖。若首选端口被占用且设置了显式环境令牌，服务器必须失败关闭而非回退，因为被占用的服务器可能使用同一显式令牌。
- `.last-token` 是同端口重连便利的持久化状态。若因首选端口被占用而回退，丢弃已加载的令牌，为回退进程生成新的未持久化令牌。
- 未从 `.last-token` 加载的新生成令牌可在同一进程内复用，因为已知无其他活跃进程持有它。

回退服务器必须继续避免覆盖 `.last-port` 和 `.last-token`。

### 4. Stop-Server 所有权证明

`start-server.sh` 应创建每次启动的服务器实例 id，并作为惰性命令行参数传给 Node，例如：

```text
node server.cjs --brainstorm-server-id=<id>
```

该 id 不是认证凭据，仅作为本地生命周期脚本的进程所有权证据。`server.cjs` 可忽略该参数。

id 必须使用 shell/MSYS 安全字符集，例如 `^[A-Za-z0-9_-]{32,64}$`。将其存入 `state/server-instance-id`，权限为仅所有者可访问。

`stop-server.sh` 应从状态中读取预期 id，仅当目标进程 argv 包含完整 argv 令牌 `--brainstorm-server-id=<id>`（而非松散子串）时才向该 PID 发送信号。优先使用 `/proc/<pid>/cmdline`，不可用时回退到宽格式 `ps` 输出。即使 `server-info` 缺失或 `lsof` 不可用，匹配的实例 id 也足以作为证明。现有端口到 PID 的检查可作为额外证据保留。

无法证明所有权时失败关闭：

- PID 文件缺失
- 服务器 id 缺失或格式错误
- 目标命令行不可用
- 目标命令行不包含预期 id
- 旧/过期会话元数据不含新 id

这有意选择让过期进程继续运行，而非杀死无关进程。

操作者可见的结果应明确：

- PID 文件缺失返回 `not_running`
- 服务器 id 缺失或格式错误返回 `stale_pid`
- 命令行不可用返回 `stale_pid`
- argv id 错误或缺失返回 `stale_pid`
- 成功停止返回 `stopped`

在 `stale_pid` 和 `stopped` 结果下，删除 `server.pid` 和 `server-instance-id`，以免后续停止尝试继续针对同一模糊进程。不要删除持久化会话内容。

### 5. 测试加固

测试应在 macOS 和用于验证的 Windows Git Bash 主机上确定性运行。

必需变更：

- 固定端口测试套件必须在服务器报告回退端口时快速失败，或让所有客户端使用报告的启动端口。
- `stop-server.test.sh` 需在任何后台进程启动前设置顶层清理 trap。
- 符号链接相关断言应探测符号链接能力，仅在主机无法创建可用测试符号链接时跳过该断言。
- 创建冒充进程的测试必须断言：生命周期元数据缺失或不足时，冒充进程存活。
- Windows/MSYS start-server 测试必须断言：类 Windows 检测仍清除 `BRAINSTORM_OWNER_PID`、仍适时自动前台化、仍精确传递实例 id argv。

### 6. 文档与 PR 一致性

Jesse 审阅前，协调审阅者可见的文档与 PR 元数据：

- 更新问题目录，使处置与本 PR 实际交付一致。
- 保持自动打开文档与已实现的 `--open` 行为一致。
- 所有位置记录的默认空闲超时保持 4 小时。
- 变基后按模板审查 PR 正文。
- 在 PR 正文中记录 macOS、Windows、浏览器/手动及外部 eval 证据，附具体命令和结果。

## 测试策略

每个行为变更使用 TDD：

1. 添加或收紧针对性回归测试。
2. 运行并确认因预期原因失败。
3. 实施最小修复。
4. 重跑针对性测试。
5. 重跑完整 brainstorm-server 套件。

必需针对性回归：

| 行为 | 测试文件 | 针对性命令 | 预期 RED | 预期 GREEN |
| --- | --- | --- | --- | --- |
| 根路由忽略符号链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已认证 `GET /` 提供指向内容外的链接 | 响应提供等待页面或安全屏幕 |
| 根路由忽略受支持的硬链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已认证 `GET /` 提供硬链接的 `server-info` | `nlink > 1` 时硬链接候选被忽略 |
| `/files/*` 限制保持不变 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 现有限制测试回归 | 空、点文件、目录、符号链接、硬链接用例仍返回 404 |
| 持久化令牌回退轮换令牌 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 URL 密钥等于持久化的首选端口密钥 | 回退 URL 密钥不同且未写入 `.last-token` |
| 显式令牌回退失败关闭 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 设置 `BRAINSTORM_TOKEN` 时服务器仍回退 | 进程以非零退出且不启动回退 |
| 回退密钥无法认证到原服务器 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退密钥从原端口收到 200 | 原端口拒绝回退密钥 |
| 正确实例 id 允许停止 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 由 start-server 启动的真实服务器存活 | stop 返回 `stopped` 且进程退出 |
| 错误、缺失、格式错误或过期 id 安全 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 冒充进程被发信号 | stop 返回 `stale_pid` 且冒充进程存活 |
| 固定端口套件不能通过回退 | `tests/brainstorm-server/server.test.js`、`tests/brainstorm-server/auth.test.js` | 各自的 `node` 命令 | 测试静默连接回退端口 | 测试明确失败或有意使用报告端口 |
| Shell 清理 trap 在失败时运行 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 失败后子进程残留 | trap 回收后台子进程 |
| Windows/MSYS 启动行为保持生命周期不变量 | `tests/brainstorm-server/start-server.test.sh`、`tests/brainstorm-server/windows-lifecycle.test.sh` | macOS 和 `ballmer` 上的 `bash` 测试命令 | 所有者 PID 或 argv 处理回归 | 所有者 PID 被清除、前台检测成立、id argv 存在 |

每个 RED/GREEN 周期应为 PR 正文留下简短证据说明：针对性命令、修复前失败断言、修复后通过断言，以及证据在 macOS 还是 Windows 上收集。

## 验证

宣布修复完成前，运行：

- `git fetch origin dev && git rebase origin/dev`
- `git diff --quiet origin/dev...HEAD -- evals`
- `gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid`
- `cd tests/brainstorm-server && npm test`
- TDD 期间使用的相关针对性测试命令
- `git diff --check`
- 对改动 JavaScript 文件做 Node 语法检查
- 对改动 shell 文件做 shell lint
- 在 `ballmer` 上做 Windows 验证：完整可运行的 brainstorm-server 套件加独立 Windows 生命周期探测

手动/浏览器测试仅在自动化通过后进行。

## 验收标准

- PR #1720 干净变基到当前 `dev`。
- `evals` 不出现在 PR diff 中。
- 根屏幕服务无法通过符号链接或受支持的硬链接逃逸读取 `content/` 之外的内容。
- `/files/*` 限制保护保持不变。
- 回退服务器不使用可能与已占用首选端口服务器共享的令牌运行。
- 所有权证明缺失或模糊时，`stop-server.sh` 不向无关进程发送信号。
- `server-info` 或 `lsof` 不可用时，`stop-server.sh` 仍能停止实例 id 匹配的合法服务器。
- 每个回归记录针对性 RED/GREEN 证据。
- PR 正文记录 macOS 和 Windows 验证证据。
- PR 正文准确描述分支内容及外部收集的证据。