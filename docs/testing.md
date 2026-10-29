# 测试 Superpowers

Superpowers 有两类不同的测试，各自放在独立目录：

- **`tests/`** — 插件的非 LLM 代码能否正常工作？Bash + node + python 集成测试，覆盖 brainstorm-server JS、OpenCode 插件加载、codex-plugin 同步和分析工具。
- **`evals/`** — 智能体在真实 LLM 会话中行为是否正确？Python 测试框架驱动真实的 Claude Code / Codex / Gemini CLI tmux 会话，由 LLM 演员和验证器评判技能合规性。

## 插件测试

位于 `tests/`。目前包括：

- `tests/brainstorm-server/` — brainstorm server JS 代码的 node 测试套件。
- `tests/opencode/` — OpenCode 插件加载、bootstrap 缓存和工具注册的 bash 测试。
- `tests/codex-plugin-sync/` — bash 同步验证。
- `tests/kimi/` — Kimi 插件清单接线的 bash/Python 检查。
- `tests/claude-code/test-helpers.sh`、`analyze-token-usage.py` — 其余 bash 测试使用的工具。
- `tests/claude-code/test-subagent-driven-development.sh` — 智能体能否描述 SDD 的测试（quorum 无对应项；测试描述回忆，而非行为）。
- `tests/claude-code/test-subagent-driven-development-integration.sh` — 扩展的 SDD 集成测试，含 token 分析（quorum 覆盖 YAGNI 子集；bash 额外断言提交数、Claude Code 任务跟踪和 token 遥测）。
- `tests/claude-code/test-worktree-native-preference.sh` — worktree 技能的 RED-GREEN-REFACTOR 验证（quorum 覆盖 PRESSURE 阶段；bash 还覆盖 RED/GREEN 基线）。
- `tests/explicit-skill-requests/` — quorum 未覆盖的 Haiku 专用、多轮和技能名提示测试。
- `tests/diagnosing-superpowers/test-skill-structure.sh` — diagnosing-superpowers 技能的结构检查（frontmatter、引用文件、泄漏扫描、字数预算）；行为场景评估记录由维护者保存在仓库外。

通过相关目录的 `run-*.sh` 或 `npm test` 运行插件测试。

## 技能行为评估

位于 `evals/`（即 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 评估实验室，原名 Drill）。Quorum 是测试框架 CLI——系统的一部分：它通过 Gauntlet QA 智能体驱动真实编码智能体 CLI，并按每个场景的验收标准及确定性后置检查评分。场景位于 `evals/scenarios/<name>/`。安装、容器运行时和安全模型见 `evals/README.md`。快速开始（本地紧急运行）：

```bash
cd evals
bun install
export SUPERPOWERS_ROOT=/path/to/superpowers
bun run quorum run scenarios/triggering-test-driven-development --coding-agent claude
bun run quorum show <run-dir>
```

Quorum 场景很慢（每个 3-30+ 分钟），且在宽松模式下运行真实 LLM 会话——请先阅读 `evals/README.md` 的 Live Eval Risk 部分。只有静态门禁（`bun run check`、`bun run quorum check`）可安全用于公共 CI；自然的后续仍是分层模型（PR 上跑静态门禁，每晚 + 按需跑实时扫描）。