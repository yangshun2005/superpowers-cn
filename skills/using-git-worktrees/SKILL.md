---
name: using-git-worktrees
description: 在开始需要与当前工作区隔离的功能开发时，或在执行实施计划之前使用——通过原生工具或 git worktree 回退方案确保存在隔离的工作区
---

# 使用 Git Worktrees

## 概述

确保工作在隔离的工作区中进行。优先使用平台原生的 worktree 工具。仅在没有原生工具可用时，才回退到手动 git worktree。

**核心原则：** 先检测是否已隔离。然后使用原生工具。最后回退到 git。绝不与运行环境对抗。

**开始时声明：** "我正在使用 using-git-worktrees 技能来设置隔离的工作区。"

## 步骤 0：检测现有隔离

**在创建任何东西之前，检查你是否已在隔离的工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块防护：** 在 git 子模块内部，`GIT_DIR != GIT_COMMON` 同样为真。在得出"已在 worktree 中"的结论之前，先确认你不在子模块中：

```bash
# 如果返回路径，说明你在子模块中，而非 worktree——按普通仓库处理
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已在链接的 worktree 中。跳到步骤 2（项目设置）。不要创建另一个 worktree。

报告分支状态：
- 在分支上："已在隔离工作区 `<path>`，分支 `<name>`。"
- 分离 HEAD："已在隔离工作区 `<path>`（分离 HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你在普通仓库检出中。

用户是否已在指令中表明 worktree 偏好？如果没有，在创建 worktree 前征求同意：

> "要我设置一个隔离的 worktree 吗？它可以保护你当前的分支不受改动影响。"

尊重任何已声明的偏好，无需询问。如果用户拒绝同意，就地工作并跳到步骤 2。

## 步骤 1：创建隔离工作区

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已要求隔离工作区（步骤 0 同意）。你是否已有创建 worktree 的方式？可能是名为 `EnterWorktree`、`WorktreeCreate` 的工具、`/worktree` 命令或 `--worktree` 标志。如果有，使用它并跳到步骤 2。

原生工具自动处理目录放置、分支创建和清理。在有原生工具时使用 `git worktree add` 会产生运行环境无法看到或管理的幻影状态。

仅在没有原生 worktree 工具可用时，才继续步骤 1b。

### 1b. Git Worktree 回退

**仅在步骤 1a 不适用时使用**——你没有原生 worktree 工具可用。使用 git 手动创建 worktree。

#### 目录选择

遵循此优先级顺序。用户的明确偏好始终优先于观察到的文件系统状态。

1. **检查指令中是否声明了 worktree 目录偏好。** 如果用户已指定，直接使用，无需询问。

2. **检查是否存在项目本地 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，使用它。如果两者都存在，`.worktrees` 优先。

3. **如果没有其他指引可用**，默认使用项目根目录下的 `.worktrees/`。

#### 安全验证（仅限项目本地目录）

**创建 worktree 前必须验证目录已被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，提交更改，然后继续。

**为何关键：** 防止意外将 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# 根据所选位置确定路径
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，你将在当前目录中工作。然后就地运行设置和基线测试。

## 步骤 2：项目设置

自动检测并运行适当的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 步骤 3：验证干净的基线

运行测试以确保工作区初始状态干净：

```bash
# 使用项目适用的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree 就绪于 <full-path>
测试通过（<N> 个测试，0 失败）
准备实施 <feature-name>
```

## 快速参考

| 情况 | 操作 |
|-----------|--------|
| 已在链接的 worktree 中 | 跳过创建（步骤 0） |
| 在子模块中 | 按普通仓库处理（步骤 0 防护） |
| 有原生 worktree 工具可用 | 使用它（步骤 1a） |
| 无原生工具 | Git worktree 回退（步骤 1b） |
| `.worktrees/` 存在 | 使用它（验证已忽略） |
| `worktrees/` 存在 | 使用它（验证已忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 两者都不存在 | 检查指令文件，然后默认 `.worktrees/` |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，就地工作 |
| 基线测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见合理化借口

| 借口 | 现实 |
|--------|---------|
| "我显然不在 worktree 中——无需检查" | 运行步骤 0。运行环境创建的隔离和子模块都会骗过肉眼；检测命令才能定论。 |
| "`git worktree add` 比寻找原生工具更快" | 原生工具（如 `EnterWorktree`）负责放置、分支和清理。绕过它是头号错误——它会产生运行环境无法看到或管理的幻影状态。 |
| "worktree 目录肯定已被忽略" | 运行 `git check-ignore`。未忽略的 worktree 目录会把整棵树提交到仓库。 |
| "任何目录名都行" | 明确指令优先于现有项目本地目录，后者优先于 `.worktrees/` 默认值。 |
| "工作区是全新的——基线测试可以等" | 脏基线会让之后每次失败都难以归因。现在就运行测试；在失败情况下继续是你的搭档的决定。 |