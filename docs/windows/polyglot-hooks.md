# Claude Code 的跨平台多语言 Hook

Claude Code 插件需要能在 Windows、macOS 和 Linux 上运行的 Hook。本文档介绍 `hooks/run-hook.cmd` 中使用的单一通用分发器模式。

> **权威来源：** `hooks/run-hook.cmd` 是规范实现。当本文档与代码不一致时，以代码为准。

## 问题

Claude Code 通过 shell 运行 Hook 命令：
- **macOS/Linux**：bash 或 sh
- **安装了 Git Bash 的 Windows**：Git Bash
- **未安装 Git Bash 的 Windows**：PowerShell（旧版本使用 CMD.exe）

Windows 的两个回退 shell 都无法解析我们的命令字符串：PowerShell 会把开头带引号的路径当作字符串表达式，并在下一个裸词处报错；而 CMD.exe 的 `/c` 引号规则会在路径包含 `(` 等元字符时剥掉外层引号。因此我们的 Hook 声明了 `"shell": "bash"`（自 Claude Code 2.1.81 起支持；旧版本会忽略该键），这会强制走 Git Bash 路径；当 Git Bash 不存在时，会给出可操作的"安装 Git for Windows"错误，而不是 shell 解析器失败。

这带来几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：`$VAR` 语法在 CMD 中无效
4. **`.sh` 自动前置**：Windows 上的 Claude Code 会自动给路径中包含 `.sh` 的任何命令前置 `bash`——如果脚本带扩展名，这会干扰分发器

## 解决方案：无扩展名脚本 + 单一通用分发器

仓库为所有 Hook 使用一个通用 `run-hook.cmd` 分发器。Hook 脚本**无扩展名**（`session-start`，而非 `session-start.sh`）。这是有意为之：它能防止 Claude Code 的 Windows 自动检测给分发器命令前置 `bash` 而将其破坏。

### 文件结构

```
hooks/
├── hooks.json          # 指向 run-hook.cmd，使用无扩展名脚本名
├── run-hook.cmd        # 跨平台分发器（多语言包装器）
└── session-start       # 实际 Hook 逻辑——无扩展名 bash 脚本
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "shell": "bash",
            "async": false
          }
        ]
      }
    ]
  }
}
```

路径加引号是因为 `${CLAUDE_PLUGIN_ROOT}` 可能包含空格。

## `run-hook.cmd` 的工作原理概述

`run-hook.cmd` 是一个多语言脚本：Windows 将第一个块视为批处理命令，而 Unix shell 将该块视为空操作 heredoc 并继续执行后面的内容。

不要从本文档复制实现。修改分发器时请直接阅读 `hooks/run-hook.cmd`，之后运行 `tests/hooks/test-session-start.sh`。

### 在 Windows（CMD.exe）上的工作方式

1. 批处理部分校验脚本名，并从分发器自身位置解析 Hook 目录。
2. 按顺序在四个位置尝试 bash：
   - `C:\Program Files\Git\bin\bash.exe`
   - `C:\Program Files (x86)\Git\bin\bash.exe`
   - `%LOCALAPPDATA%\Programs\Git\bin\bash.exe`（按用户安装的 Git for Windows），仅在设置了 `LOCALAPPDATA` 时尝试
   - `PATH` 上的 `bash`（MSYS2、Cygwin 或非默认 Git 安装），通过运行 `%SystemRoot%\System32\where.exe $PATH:bash` 查找，因此 `where` 和 `bash` 都不会从当前目录获取。WSL 启动器（`System32`、`Sysnative` 或 `%LOCALAPPDATA%\Microsoft\WindowsApps` 中的 `bash.exe`）和无扩展名匹配会被跳过，因为未安装 Linux 发行版时这些启动器会失败。
3. 如果找到 bash，则从 hooks 目录运行指定的无扩展名 Hook 脚本，并以该脚本的退出码退出。
4. 如果未找到 bash，分发器静默退出 `0`——插件继续工作，只是跳过该 Hook。
5. `exit /b` 在 CMD 到达 Unix 部分之前将其停止。

### 在 Unix（bash/sh）上的工作方式

1. `: << 'CMDBLOCK'` 在空操作命令上打开一个 heredoc。
2. 整个 CMD 批处理块被 heredoc 消费并忽略。
3. 在 `CMDBLOCK` 之后，bash 解析脚本目录并直接 `exec` 指定的无扩展名脚本。

### 关键设计决策

| 决策 | 原因 |
|----------|-----|
| 无扩展名脚本 | 防止 Claude Code 的 Windows `.sh` 自动前置干扰分发器命令 |
| 不使用 `-l`（登录 shell） | 不需要；Hook 脚本应自包含，不依赖登录 shell 的 PATH 设置 |
| 不使用 `cygpath` | bash 直接接收 Windows 路径并能正确处理；`cygpath` 是旧的 `-c "..."` 调用模式所需要的，直接 exec 不需要 |
| 无 bash 时静默退出 | 避免破坏未安装 Git for Windows 的用户的插件；Hook 上下文注入会被优雅跳过 |

## 编写跨平台 Hook 脚本

Hook 逻辑放在无扩展名脚本文件中。几个可移植模式：

### 推荐
- 尽可能使用纯 bash 内建命令
- 使用 `$(command)` 而非反引号
- 所有变量展开都加引号：`"$VAR"`

### 避免
- 依赖 PATH 中的工具且无回退（Hook 运行时没有 `-l`，因此登录 shell 的 PATH 未设置）
- 给脚本加 `.sh` 扩展名——这会触发 Claude Code 的 Windows 自动前置

### 示例：不使用外部工具的 JSON 转义

```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 故障排查

### "bash is not recognized"

CMD 在分发器尝试的四个位置中都找不到 bash。分发器会静默退出（0）而非报错，因此 Hook 被跳过。安装 Git for Windows（系统级或按用户），或确保 `PATH` 上有非 WSL 的 `bash`。

### Hook 在 Unix 上运行但在 Windows 上无反应

检查 `hooks.json` 中的脚本文件名是否**无扩展名**。像 `run-hook.cmd session-start.sh` 这样的命令可能触发 Claude Code 的 `.sh` 自动检测并绕过预期的 CMD 分发器路径，或者只是尝试运行不存在的 `session-start.sh` 脚本。

### Hook 完全不触发

验证 `hooks.json` 中的 `matcher` 是否匹配你的 harness 发出的事件类型。Claude Code 使用 `startup|clear|compact`；Cursor 使用 `sessionStart`。查看 `hooks-cursor.json` 了解 Cursor 变体。

## 相关 Issue

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) — `.sh` 脚本在 Windows 上于编辑器中打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) — Hook 在 Windows 上不工作