# 平台中立 README 排序 — 阶段 C 设计

## 背景

阶段 A 和 B（见 `2026-05-05-platform-neutral-prose-design.md` 和 `2026-05-05-platform-neutral-config-refs-design.md`）已中立化 README 中的通用 Claude 文案和配置文件引用。剩余的平台倾向信号是布局：README 中的两处平台列表都把 Claude Code 放在首位，其余部分也并非严格按字母排序。

本阶段修正排序。不改文案。

## 范围内

1. **快速开始平台列表**（`README.md:7`）——受支持 harness 的内联链接列表
2. **安装章节排序**（`README.md:35–152`）——各 harness 的安装子章节

## 范围外

- 文案、市场名称、插件 ID、URL——均事实正确，保持原样。
- Claude Code 章节的视觉权重（该章节有两个子章节——Anthropic 官方市场和 Superpowers 市场）。两者都是真实的安装路径；合并会隐藏准确信息。
- 各安装块内的章节标题和内容——仅改变块的顺序。

## 替换

两处列表均重排为严格字母序：

| 旧顺序 | 新顺序 |
|-----------|-----------|
| Claude Code | Claude Code |
| Codex CLI | Codex App |
| Codex App | Codex CLI |
| Factory Droid | Cursor |
| Gemini CLI | Factory Droid |
| OpenCode | Gemini CLI |
| Cursor | GitHub Copilot CLI |
| GitHub Copilot CLI | OpenCode |

三处移动：Codex App 与 Codex CLI 互换；Cursor 上移两位；GitHub Copilot CLI 上移一位。

Claude Code 因字母顺序巧合仍居首位（`Cl…` 先于 `Co…`）。

## 提交计划

一个原子提交覆盖两处列表，因为只改一处而不改另一处会导致快速开始与安装章节不一致。

## 验证

- 快速开始锚点（`#claude-code`、`#codex-app` 等）仍能解析到现有的 `### …` 标题——未重命名任何标题。
- 每个安装子章节的正文在改动前后逐字节一致；仅位置变化。
- `git diff README.md` 仅显示章节移动，无内容编辑。