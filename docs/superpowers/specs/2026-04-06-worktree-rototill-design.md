# Worktree Rototill：检测并让位

**日期：** 2026-04-06
**状态：** 草稿
**工单：** PRI-974
**涵盖：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对 worktree 管理有强预设——特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理方式（`git worktree remove`）。而 Claude Code、Codex App、Gemini CLI 和 Cursor 都提供原生 worktree 支持，各有自己的路径、生命周期管理和清理方式。

这导致三种故障模式：

1. **重复**——在 Claude Code 上，技能做的事和 `EnterWorktree`/`ExitWorktree` 已经做的一样
2. **冲突**——在 Codex App 上，技能试图在已被管理的 worktree 内再创建 worktree
3. **幽灵状态**——技能在 `.worktrees/` 创建的 worktree 对 harness 不可见；harness 在 `.claude/worktrees/` 创建的 worktree 对技能不可见

对于没有原生支持的 harness（Codex CLI、OpenCode、Copilot 独立版），superpowers 填补了真实空白。技能不应消失——而是在原生支持存在时让位。

## 目标

1. 原生 harness worktree 系统存在时，让位给它
2. 继续为缺少原生支持的 harness 提供 worktree 支持
3. 修复 finishing-a-development-branch 中的三个已知 bug（#940、#999、#238）
4. 将 worktree 创建改为可选而非强制（#991）
5. 将硬编码的 `CLAUDE.md` 引用替换为平台中立表述（#1049）

## 非目标

- 按 worktree 的环境约定（`.worktree-env.sh`、端口偏移）——Phase 4
- 用于路径强制的 PreToolUse Hook——Phase 4
- 多仓库 worktree 文档——Phase 4
- 针对 worktree 的头脑风暴清单变更——Phase 4
- `.superpowers-session.json` 元数据跟踪（PR #997 的有趣想法，v1 不需要）
- 将 Hook 符号链接到 worktree（PR #965 的想法，独立问题）

## 设计原则

### 检测状态，而非平台

用 `GIT_DIR != GIT_COMMON` 判断"我是否已在 worktree 中"，而不是嗅探环境变量来识别 harness。这是稳定的 git 原语（自 git 2.5，2015 年起），在所有 harness 上通用，新 harness 出现时零维护。

### 声明意图，规定回退

技能描述目标（"确保工作在隔离工作区中进行"），有原生工具时让位。仅在 harness 无原生 worktree 支持时，才规定具体 git 命令作为回退。Step 1a 在前，明确列出原生工具名（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；Step 1b 在后，提供 git 回退。原始规范将 Step 1a 写得抽象（"你了解自己的工具集"），但 TDD 证明：Step 1a 过于模糊时，agent 会锚定 Step 1b 的具体命令。需要明确工具命名和同意-授权桥接，偏好才可靠。

### 基于来源的所有权

谁创建 worktree，谁负责清理。harness 创建的，superpowers 不碰。superpowers 创建的（通过 git 回退），superpowers 清理。启发式规则：worktree 位于 `.worktrees/` 或 `worktrees/` 下，归 superpowers 所有。其他位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`，或旧的用户全局 Superpowers 路径）归 harness 或用户所有，不动。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

技能在创建前新增三个步骤，并简化创建流程。

#### Step 0：检测现有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 操作 |
|------|------|------|
| `GIT_DIR == GIT_COMMON` | 普通仓库检出 | 进入 Step 0.5 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 已在链接 worktree 中 | 跳到 Step 3（项目设置）。报告："已在隔离工作区 `<path>`，分支 `<name>`。" |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 外部管理的 worktree（如 Codex App 沙箱） | 跳到 Step 3。报告："已在隔离工作区 `<path>`（分离 HEAD，外部管理）。" |

Step 0 不关心谁创建了 worktree 或运行哪个 harness。worktree 就是 worktree，不论来源。

**子模块防护：** `GIT_DIR != GIT_COMMON` 在 git 子模块内也为真。在得出"已在 worktree 中"结论前，确认不在子模块中：

```bash
# 如果返回路径，说明在子模块中，不是 worktree
git rev-parse --show-superproject-working-tree 2>/dev/null
```

若在子模块中，按 `GIT_DIR == GIT_COMMON` 处理（进入 Step 0.5）。

#### Step 0.5：征得同意

Step 0 未发现现有隔离（`GIT_DIR == GIT_COMMON`）时，创建前询问：

> "要我设置一个隔离 worktree 吗？这能保护你当前分支不受改动影响。（y/n）"

是，进入 Step 1。否，就地工作——跳到 Step 3，不创建 worktree。

Step 0 检测到现有隔离时完全跳过此步（已存在的东西无需再问）。

#### Step 1a：原生工具（首选）

> 用户已要求隔离工作区（Step 0 同意）。检查可用工具——是否有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令或 `--worktree` 标志？有：用户同意创建 worktree 即是你使用它的授权。立即使用，跳到 Step 3。

使用原生工具后，跳到 Step 3（项目设置）。

**设计说明——TDD 修订：** 原始规范故意将 Step 1a 写得简短抽象（"你了解自己的工具集——技能无需列出具体工具"）。TDD 验证推翻了这一点：agent 锚定 Step 1b 的具体 git 命令，忽略抽象指引（通过率 2/6）。三项修改解决了问题（GREEN 和 PRESSURE 测试合计 50/50 通过）：

1. **明确工具命名**——按名称列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`，将决策从解读（"我有原生工具吗？"）变为事实查找（"`EnterWorktree` 在我的工具列表里吗？"）。没有这些工具的平台上的 agent 只需检查、找不到、落入 Step 1b。未观察到误报。
2. **同意桥接**——"用户同意创建 worktree 即是你使用它的授权"直接回应 `EnterWorktree` 的工具级护栏（"仅在用户明确要求时"）。工具描述优先于技能指令（Claude Code #29950），因此技能必须将用户同意表述为工具所需的授权。
3. **红旗条目**——在 Red Flags 部分点名具体反模式（"有原生 worktree 工具时使用 `git worktree add`——这是头号错误"）。

文件拆分（Step 1b 放入独立技能）经测试证明不必要。锚定问题靠 Step 1a 文本质量解决，而非物理分离 git 命令。完整 240 行技能（所有 git 命令可见）的对照测试 20/20 通过。

#### Step 1b：Git Worktree 回退

无原生工具时，手动创建 worktree。

**目录选择**（优先级顺序）：
1. 检查项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）中的 worktree 目录偏好。
2. 检查现有 `.worktrees/` 或 `worktrees/` 目录——找到则使用。两者都存在时，`.worktrees/` 优先。
3. 默认 `.worktrees/`。

无交互式目录选择提示。不检测也不提供旧的用户全局 Superpowers worktree 路径；新的手动 worktree 为项目本地，除非用户明确指定其他位置。

**安全验证**（仅项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

未被忽略则添加到 `.gitignore` 并提交后再继续。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**Hook 感知：** Git worktree 不继承父仓库的 Hook 目录。通过 1b 创建 worktree 后，若主仓库存在 Hook 目录，则符号链接：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这防止工作移到 worktree 后，pre-commit 检查、linter 等 Hook 静默停止。（想法来自 PR #965。）

**沙箱回退：** `git worktree add` 因权限错误失败时，视为受限环境。跳过创建，在当前目录工作，进入 Step 3。

**步骤编号说明：** 当前技能 Steps 1-4 为平铺列表。本设计使用 0、0.5、1a、1b、3、4。没有 Step 2——它是旧的单体"Create Isolated Workspace"，现拆分为 1a/1b 结构。实现时应干净重编号（如 0 → "Step 0: Detect"，0.5 → 并入 Step 0 流程，1a/1b → "Step 1"，3 → "Step 2"，4 → "Step 3"），或保留当前编号加注释。实现者选择。

#### Steps 3-4：项目设置与基线测试（不变）

无论哪条路径创建工作区（Step 0 检测到现有、Step 1a 原生工具、Step 1b git 回退，或无 worktree），执行汇合：

- **Step 3：** 自动检测并运行项目设置（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **Step 4：** 运行测试套件。测试失败则报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

finishing 技能新增环境检测并修复三个 bug。

#### Step 1：验证测试（不变）

运行项目测试套件。测试失败则停止。不提供完成选项。

#### Step 1.5：检测环境（新增）

重新运行与创建时 Step 0 相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 需清理 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 标准 4 选项 | 基于来源（见 Step 5） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 精简菜单：作为新分支推送 + PR、保持原样、丢弃 | 无合并选项（无法从分离 HEAD 合并） |

#### Step 2：确定基础分支（不变）

#### Step 3：呈现选项

**普通仓库和具名分支 worktree：**

1. 本地合并回 `<base-branch>`
2. 推送并创建 Pull Request
3. 保持分支原样（稍后自行处理）
4. 丢弃此工作

**分离 HEAD：**

1. 作为新分支推送并创建 Pull Request
2. 保持原样（稍后自行处理）
3. 丢弃此工作

#### Step 4：执行选择

**选项 1（本地合并）：**

```bash
# 获取主仓库根目录以保证 CWD 安全（Bug #238 修复）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并，确认成功后再移除任何东西
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# 仅在合并成功后：移除 worktree，再删除分支（Bug #999 修复）
git worktree remove "$WORKTREE_PATH"  # 仅当 superpowers 拥有它
git branch -d <feature-branch>
```

顺序至关重要：合并 → 验证 → 移除 worktree → 删除分支。旧技能在移除 worktree 前删除分支（失败，因为 worktree 仍引用该分支）。先移除 worktree 的朴素修复也是错的——若合并随后失败，工作目录已消失，改动丢失。

**选项 2（创建 PR）：**

推送分支，创建 PR。不要清理 worktree——用户需要它进行 PR 迭代。（Bug #940 修复：删除矛盾的"然后：清理 worktree"文字。）

**选项 3（保持原样）：** 无操作。

**选项 4（丢弃）：** 要求输入"discard"确认。然后移除 worktree（若 superpowers 拥有），强制删除分支。

#### Step 5：清理（更新）

```
if GIT_DIR == GIT_COMMON:
    # 普通仓库，无 worktree 需清理
    done

if worktree 路径在 .worktrees/ 或 worktrees/ 下:
    # Superpowers 创建的——我们负责清理
    cd 到主仓库根目录       # Bug #238 修复
    git worktree remove <path>

else:
    # Harness 创建的——不碰
    # 若平台提供工作区退出工具，使用它
    # 否则，保留 worktree 原样
```

清理仅在选项 1 和 4 时运行。选项 2 和 3 始终保留 worktree。（Bug #940 修复。）

**陈旧 worktree 修剪：** 每次 `git worktree remove` 后，运行 `git worktree prune` 作为自愈步骤。Worktree 目录可能被带外删除（如 harness 清理、手动 `rm` 或 `.claude/` 清理），留下陈旧注册导致令人困惑的错误。一行命令，防止静默腐化。（想法来自 PR #1072。）

### 3. 集成更新

#### `subagent-driven-development` 和 `executing-plans`

两者当前在集成部分将 `using-git-worktrees` 列为必需。改为：

> `using-git-worktrees` — 确保隔离工作区（创建或验证现有）

技能本身现在处理同意（Step 0.5）和检测（Step 0），调用技能无需再设门槛或提示。

#### `writing-plans`

删除过时说法"应在专用 worktree 中运行（由头脑风暴技能创建）"。头脑风暴是设计技能，不创建 worktree。worktree 提示在执行时通过 `using-git-worktrees` 发生。

### 4. 平台中立的指令文件引用

worktree 相关技能中所有硬编码 `CLAUDE.md` 替换为：

> "你项目的 agent 指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules 或等效文件）"

适用于 Step 1b 中的目录偏好检查。

## Bug 修复（捆绑）

| Bug | 问题 | 修复 | 位置 |
|-----|------|------|------|
| #940 | 选项 2 文字说"然后：清理 worktree（Step 5）"，但快速参考说保留。Step 5 说"选项 1、2、4"，但 Common Mistakes 说"仅选项 1 和 4"。 | 从选项 2 移除清理。Step 5 仅适用于选项 1 和 4。 | finishing SKILL.md |
| #999 | 选项 1 在移除 worktree 前删除分支。`git branch -d` 可能失败，因为 worktree 仍引用分支。 | 重排为：合并 → 验证测试 → 移除 worktree → 删除分支。合并必须先成功，再移除任何东西。 | finishing SKILL.md |
| #238 | CWD 在待移除 worktree 内时，`git worktree remove` 静默失败。 | 添加 CWD 防护：`git worktree remove` 前 `cd` 到主仓库根目录。 | finishing SKILL.md |

## 已解决的 Issue

| Issue | 解决方式 |
|-------|---------|
| #940 | 直接修复（Bug #940） |
| #991 | Step 0.5 可选同意 |
| #918 | Step 0 检测 + Step 1.5 finishing 检测 |
| #1009 | 由 Step 1a 解决——agent 使用原生工具（如 `EnterWorktree`），在 harness 原生路径创建。依赖 Step 1a 生效；见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立的指令文件引用 |
| #279 | 由检测并让位解决——尊重原生路径，因为我们不覆盖它们 |
| #574 | **推迟。** 本规范未触及 bug 所在的头脑风暴技能。完整修复（在头脑风暴清单中添加 worktree 步骤）为 Phase 4。 |

## 风险

### Step 1a 是承重假设——已解决

Step 1a——agent 优先使用原生 worktree