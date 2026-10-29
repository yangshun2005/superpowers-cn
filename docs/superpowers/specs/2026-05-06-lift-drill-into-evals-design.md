# 将 drill 提升为 superpowers 中的 `evals/` — 设计

## 背景

Drill 是一个 Python 技能合规性基准测试，位于独立仓库 `obra/drill`。它驱动真实的 tmux 会话，运行 LLM 演员作为模拟用户，对生成的对话记录运行 LLM 验证器，并按场景报告通过/失败。它支持 Claude Code、Codex、Gemini CLI，以及（根据近期提交）OpenCode 和 Copilot CLI。

Drill 实际上已经是 superpowers 的*事实上的*评估工具。drill 仓库中的 PRI-1397 提交系列将约 22 个 superpowers bash 测试提升为 drill 场景，而最近的 superpowers 提交（`a2292c5`）明确删除了一个冗余的 bash 测试，消息为*"replaced by drill behavioral coverage"*。迁移势头已经存在；本规范将其完成。

本工作将 drill 移入 superpowers 的 `evals/` 下，在逐文件验证 drill 场景覆盖后删除冗余的 bash 测试，并更新文档，使贡献者落到新结构上。

## 目标

1. `evals/` 是 superpowers 中的规范评估工具——完整的 drill 源代码、场景、fixtures、prompts、后端配置和测试。
2. `superpowers/tests/` 中经单独验证被 drill 场景 100% 覆盖的 bash 测试被删除；其余保留。
3. `tests/`（插件基础设施：bash + node + python 集成测试）与 `evals/`（带演员 + 验证器的 LLM 行为）之间的划分是有意义的，并有文档记录。
4. 顶层文档（`README.md`、`CLAUDE.md`、`docs/testing.md`）将贡献者指向正确的位置。
5. 独立的 `obra/drill` 仓库继续存在（本 PR 不触碰它），并在本 PR 合并后作为单独的手动步骤归档。

## 非目标

- **CI 集成。** 此处仅手动。自然的后续是"分层"：每个 PR 运行快速子集，每晚 + 按需运行完整扫描。这需要 API 预算决策、GitHub Actions secrets，以及安装了 `tmux` + `node` + `python` + `claude` / `codex` / `gemini` CLI 的 runner 镜像。超出范围。
- **场景与技能同置。** 场景保持集中在 `evals/scenarios/`。如果以后决定每个技能拥有自己的场景，那是一个查找路径并重命名的操作；YAML 格式不变。
- **重命名内部 Python 包**（`drill` → `evals`）。目录是 `evals/`（面向用户）；Python 包保留 `drill` 名称以保持 diff 小。`evals/README.md` 中的简短说明解释了这一点。
- **Drill 仓库归档。** 本 PR 不触碰 `obra/drill`。合并后，drill 仓库手动归档（GitHub 上只读，README 指向 `obra/superpowers/evals/`）。
- **将 `tests/claude-code/analyze-token-usage.py` 提升到 `evals/bin/`。** 有用的工具，不是测试代码。可以稍后移动；本 PR 不要求。

## 分支

从 `dev` 分出 `f/evals-lift`。本工作独立于开放的 `f/cross-platform` PR——除了可能的 `README.md` 外没有共享文件变更，后者足够小，如果冲突可在合并时解决。

## 移动后的架构

```
superpowers/
  evals/                              ← 新增（完整 drill 副本）
    pyproject.toml                    (Python 3.11, uv 管理)
    uv.lock
    .gitignore                        (drill 自己的；results/, .venv/, .env)
    README.md                         (原 drill 的 README；安装说明已更新)
    CLAUDE.md                         (原 drill 的 CLAUDE.md；路径已更新)
    docs/
      design.md                       (drill 的设计 — 逐字保留，从本规范交叉链接)
      manual-testing.md
      pressure-and-red-testing.md
    drill/                            (Python 包；名称保留；cli、engine、actor、verifier 等)
    backends/                         (claude-*.yaml, codex.yaml, gemini.yaml)
    scenarios/                        (32+ YAML 场景)
    setup_helpers/                    (15 个 Python 辅助脚本；create_base_repo、sdd_*、spec_*、worktree 等)
    fixtures/                         (template-repo, sdd-go-fractals, sdd-svelte-todo)
    prompts/                          (actor.md, verifier.md)
    bin/                              (断言辅助脚本：tool-called、tool-count 等)
    tests/                            (drill 自己的 pytest 套件)

  tests/                              ← bash 测试默认保留
    brainstorm-server/                ← 保留（brainstorm-server JS 代码的 node 测试）
    opencode/                         ← 保留（插件加载测试）
    codex-plugin-sync/                ← 保留（同步验证）
    claude-code/                      ← 大部分保留 — 见删除门禁
    explicit-skill-requests/          ← 保留，除非验证已被替换
    skill-triggering/                 ← 保留，除非验证已被替换
    subagent-driven-dev/              ← 保留，除非验证已被替换

  docs/
    testing.md                        ← 已更新（拆分为"插件测试" + "技能行为评估"）
    superpowers/
      specs/
        2026-05-06-lift-drill-into-evals-design.md   ← 本规范

  README.md                           ← 贡献部分的小指针指向 evals/
  CLAUDE.md                           ← 一行"评估工具位于 evals/"指针
```

本 PR 之后，`tests/` 和 `evals/` 目录服务于明显不同的角色：

- **`tests/`** — 插件的非 LLM 代码能工作吗？brainstorm-server JS 代码、OpenCode 插件加载、codex-plugin-sync 同步验证的单元和集成测试。Bash + node + python。
- **`evals/`** — 代理在真实 LLM 会话中行为正确吗？带演员 + 验证器的 drill 场景。仅 Python，运行真实 tmux 会话。

## 删除门禁（每个 bash 测试）

一个 bash 测试*仅当*某个 drill 场景可验证地覆盖其做出的每一个断言时才被删除。实现计划记录每个文件的此验证：阅读 bash 测试，列出其检查项，找到 drill 场景，确认每个检查项都有匹配的 `verify.assertions` 或 `verify.criteria` 条目。如果哪怕一个检查项缺失，选项是扩展 drill 场景或保留 bash 测试。默认保留。

**暂定覆盖映射**（基于提交消息；任何删除前需要逐文件验证）：

| Bash 测试 | 声称的 drill 替代 | 覆盖状态 |
|-----------|---------------------------|-----------------|
| `tests/skill-triggering/prompts/*`（6 个 prompt 文件） | `triggering-*.yaml`（6 个场景） | 候选 — 删除前逐 prompt 验证 |
| `tests/skill-triggering/run-test.sh`、`run-all.sh` | 不适用（运行器，非测试） | **保留** — 运行器脚本 |
| `tests/explicit-skill-requests/prompts/please-use-brainstorming.txt` | 需要验证 — drill 尚无明显的对应物 | 可能**保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` | 需要验证 — drill 尚无明显的对应物 | 可能**保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/run-claude-describes-sdd.sh` | 部分 → `mid-conversation-skill-invocation.yaml` | 候选 — 逐脚本验证 |
| `tests/explicit-skill-requests/run-haiku-test.sh` | 没有 drill 场景覆盖 Haiku 特定行为 | **保留** |
| `tests/explicit-skill-requests/run-multiturn-test.sh`、`run-extended-multiturn-test.sh` | 没有 drill 场景覆盖多轮构建 | **保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/run-test.sh`、`run-all.sh` | 不适用（运行器） | **保留** |
| `tests/subagent-driven-dev/go-fractals/`、`tests/subagent-driven-dev/svelte-todo/` | `sdd-go-fractals.yaml`、`sdd-svelte-todo.yaml` | 候选 — 删除前验证（这些包含关于测试套件通过的真实断言） |
| `tests/claude-code/test-document-review-system.sh` | `spec-reviewer-catches-planted-flaws.yaml` | 候选 — 删除前验证 |
| `tests/claude-code/test-requesting-code-review.sh` | `code-review-catches-planted-bugs.yaml` | 候选 — 删除前验证 |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | `sdd-rejects-extra-features.yaml`（YAGNI 子集） | **部分** — bash 测试还断言 ≥3 次提交 / `npm test` 通过 / 运行 `analyze-token-usage.py`。Drill 场景断言 forbidden-exports + reviewer-as-gate。大部分不相交 — 几乎肯定**保留 + 扩展 drill 场景**。 |
| `tests/claude-code/test-subagent-driven-development.sh` | 元/文档测试（要求代理*描述* SDD）；没有 drill 场景覆盖描述测试 | **保留**，除非添加 drill 场景 |
| `tests/claude-code/test-worktree-native-preference.sh` | `worktree-creation-under-pressure.yaml` | 候选 — 删除前验证 |
| `tests/claude-code/test-helpers.sh`、`run-skill-tests.sh`、`analyze-token-usage.py` | 不适用（工具，非测试） | **保留** — 库/工具 |

## 验证协议（子代理门禁）

实现计划中的每个变更在提交前都由独立子代理交叉检查。

| 变更类别 | 子代理验证 |
|----------------|----------------------|
| 每个 bash 测试删除 | 派发子代理，提供：(a) bash 测试文件内容，(b) 候选 drill 场景 YAML，(c) 提示：*"列出 bash 测试做出的每一个断言。列出 drill 场景中的每一个 verify 条目。对于每个 bash 断言，找到匹配的 drill 检查项或报告为未匹配。输出逐断言表格。"* 子代理的输出是门禁——仅当每个 bash 断言都有匹配时才删除。 |
| 初始 `evals/` 复制 | 子代理验证：(a) 正在复制的 drill SHA 记录在提升提交消息中，以便来源可审计；(b) **逐文件 SHA-256 校验和**与 drill 仓库匹配每个文件（不仅是文件计数）；(c) 排除的路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、任何 `.private-journal/`）在 `evals/` 中不存在；(d) 所有后端 YAML 引用的路径在移动后存在；(e) `pyproject.toml`、`uv.lock`、`.gitignore` 完好。 |
| Drill 自己的 pytest 套件 | 子代理在路径默认值更改后运行 `cd evals && uv run pytest`。Drill 在 `evals/tests/` 附带自己的 pytest 套件，包括 `test_backend.py`，它测试 `SUPERPOWERS_ROOT` 环境变量行为——这些测试必须更新以匹配辅助脚本并继续通过。 |
| 删除后的引用清理 | 子代理在整个 superpowers 树中 grep（排除 `node_modules/`、`.venv/` 和 `evals/`）对已删除 bash 测试路径的引用。搜索目标：`docs/`、`docs/superpowers/plans/`、`RELEASE-NOTES.md`、`CLAUDE.md`、`GEMINI.md`、`AGENTS.md`、`README.md`、`.github/`、`scripts/`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`lefthook.yml`。任何命中要么更新，要么暴露遗漏的依赖。 |
| 路径默认值更改（`SUPERPOWERS_ROOT` 默认值） | 子代理在路径更改后运行至少一个廉价的 drill 场景（例如 `triggering-test-driven-development`）并确认它仍然通过。真实验证，不仅仅是代码审查。 |
| 最终 PR 前对抗性审查 | 两个子代理并行，"找到最多合理问题的人得 5 分"框架——与 cross-platform PR 使用的相同协议。验证源代码和行为。 |

每个子代理任务在实现计划中都有自己的条目，带有明确的输入和通过标准。子代理的输出在相关提交消息中总结（"Subagent verification: …"），以便轨迹可审计。

## 具体路径/配置编辑

**在编写本规范之前已验证。** `drill/cli.py` 定义 `PROJECT_ROOT = Path(__file__).parent.parent`。移动后，`cli.py` 位于 `evals/drill/cli.py`，因此 `PROJECT_ROOT` 解析为 `evals/`，`PROJECT_ROOT.parent` 解析为 superpowers 仓库根。这就是 `SUPERPOWERS_ROOT` 默认应取的值。

**YAML 替换审计。** 只有四个 `claude*.yaml` 后端配置将 `${SUPERPOWERS_ROOT}` 插值到 `args` 中（用于 `--plugin-dir` 标志）；`codex.yaml` 和 `gemini.yaml` 仅在 `required_env` 中列出 `SUPERPOWERS_ROOT`（由 `engine.py:233` / `setup.py:25` 的 `os.environ["SUPERPOWERS_ROOT"]` 查找在运行前/后钩子中消费）。辅助脚本的 `os.environ` 变更覆盖两条代码路径。

| 文件 | 当前 | 之后 |
|------|---------|-------|
| `drill/cli.py` | 模块导入时 `load_dotenv(PROJECT_ROOT / ".env")`；没有关于 `SUPERPOWERS_ROOT` 的内容 | 在 `load_dotenv` 之后，调用新辅助函数 `_set_superpowers_root_default()`，当且仅当尚未设置时将 `os.environ["SUPERPOWERS_ROOT"]` 设为 `str(PROJECT_ROOT.parent)`。顺序：`load_dotenv` → 设置默认值 → click 组定义。 |
| `drill/engine.py:233`、`drill/setup.py:25` | 直接 `os.environ["SUPERPOWERS_ROOT"]` 访问（未设置时 KeyError） | 不变。CLI 启动钩子保证在 engine/setup 执行时环境变量已设置。 |
| `backends/claude*.yaml`（5 个文件） | `${SUPERPOWERS_ROOT}` 在 `args` 中替换用于 `--plugin-dir` | 不变。YAML 替换在后端加载时读取 `os.environ`，这在 CLI 启动之后。 |
| `backends/codex.yaml`、`backends/gemini.yaml` | `SUPERPOWERS_ROOT` 仅在 `required_env` 中 | 从 `required_env` 中移除（辅助脚本提供它）。`claude*.yaml` 保留 `required_env` 以向后兼容（环境变量作为覆盖有效）。 |
| `evals/tests/test_backend.py` | 测试断言 `SUPERPOWERS_ROOT` 在 `required_env` 列表中，加上路径解析测试 | 更新测试以匹配新契约：辅助脚本提供的默认值、环境覆盖仍然有效、codex/gemini 不再需要 `required_env`。 |
| `evals/README.md` | "export SUPERPOWERS_ROOT=/path/to/superpowers" | 删除 export 行；注明环境变量自动默认为 `evals/` 的父目录；提及唯一需要的设置是 `ANTHROPIC_API_KEY`（或 `OPENAI_API_KEY` / Gemini 认证）。 |
| `evals/CLAUDE.md` | 同上 | 同上 |
| `evals/.gitignore` | drill 现有的模式（`results/`、`.venv/`、`__pycache__/`、`.env`、`*.pyc`、`*.egg-info/`、`dist/`、`build/`、`.claude/`） | 逐字复制。模式相对于文件位置，因此在 `evals/` 下正确应用。 |
| `evals/lefthook.yml` | drill 附带 `lefthook.yml`，定义 `pre-commit: uv run ruff check && uv run ty check` | 移动到 `evals/lefthook.yml`。要么 (a) 在 superpowers 根安装 lefthook 并让它联邦到 `evals/lefthook.yml`，要么 (b) 记录贡献者手动运行 `cd evals && lefthook run pre-commit`。**实现中的决定：选项 (b) 以简化**——superpowers 的顶层工作流不变。 |

`.env` 放置：保留 `evals/.env`（gitignored）。贡献者从那里 source 它或在 shell 环境中设置 `ANTHROPIC_API_KEY`。

**需要小添加的顶层 superpowers 文件：**

- `superpowers/.gitignore`：添加 `evals/results/`、`evals/.venv/`、`evals/.env`（双保险；evals/.gitignore 已在本地覆盖这些）。
- `superpowers/CLAUDE.md`：添加一行指针"评估工具位于 `evals/` — 见 `evals/README.md`"，以便代理发现它。
- `superpowers/docs/testing.md`：拆分为"## 插件测试"（现有 tests/ 内容，删除的测试引用被修剪）和"##