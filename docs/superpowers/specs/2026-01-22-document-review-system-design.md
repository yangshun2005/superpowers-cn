# 文档审查系统设计

## 概述

为 superpowers 工作流新增两个审查阶段：

1. **规格文档审查** — 头脑风暴之后、编写计划之前
2. **计划文档审查** — 编写计划之后、实施之前

两者均采用实施审查所用的迭代循环模式。

## 规格文档审查器

**目的：** 验证规格完整、一致，且已准备好进入实施规划。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 关注点 |
|----------|------------------|
| 完整性 | TODO、占位符、"TBD"、未完成的章节 |
| 覆盖度 | 缺失的错误处理、边界情况、集成点 |
| 一致性 | 内部矛盾、需求冲突 |
| 清晰度 | 需求含糊 |
| YAGNI | 未被要求的功能、过度设计 |

**输出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**审查循环：** 发现问题 -> 头脑风暴 agent 修复 -> 重新审查 -> 重复直至通过。

**调度机制：** 使用 Task 工具，`subagent_type: general-purpose`。审查器提示词模板提供完整提示词。由头脑风暴技能的控制器调度审查器。

## 计划文档审查器

**目的：** 验证计划完整、与规格一致，且任务分解合理。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 关注点 |
|----------|------------------|
| 完整性 | TODO、占位符、未完成的任务 |
| 规格对齐 | 计划覆盖规格需求，无范围蔓延 |
| 任务分解 | 任务原子化、边界清晰 |
| 任务语法 | 任务和步骤使用复选框语法 |
| 分块大小 | 每个分块不超过 1000 行 |

**分块定义：** 分块是计划文档内任务的逻辑分组，由 `## Chunk N: <name>` 标题分隔。编写计划技能根据逻辑阶段（如"基础"、"核心功能"、"集成"）创建这些边界。每个分块应足够自包含，以便独立审查。

**规格对齐验证：** 审查器接收两项内容：
1. 计划文档（或当前分块）
2. 规格文档路径，供参考

审查器阅读两者并比对需求覆盖情况。

**输出格式：** 与规格审查器相同，但范围限于当前分块。

**审查流程（逐块进行）：**
1. 编写计划创建分块 N
2. 控制器携带分块 N 内容和规格路径调度计划文档审查器
3. 审查器阅读分块和规格，返回结论
4. 若有问题：编写计划 agent 修复分块 N，回到步骤 2
5. 若通过：继续分块 N+1
6. 重复直至所有分块通过

**调度机制：** 与规格审查器相同 — Task 工具，`subagent_type: general-purpose`。

## 更新后的工作流

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**规格审查循环：**
1. 规格完成
2. 调度审查器
3. 若有问题：修复 -> 回到 2
4. 若通过：继续

**计划审查循环：**
1. 分块 N 完成
2. 为分块 N 调度审查器
3. 若有问题：修复 -> 回到 2
4. 若通过：下一分块或进入实施

## Markdown 任务语法

任务和步骤使用复选框语法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 错误处理

**审查循环终止：**
- 无硬性迭代上限 — 循环持续至审查器通过
- 若循环超过 5 次迭代，控制器应向人类上报以获取指导
- 人类可选择：继续迭代、带已知问题通过、或中止

**分歧处理：**
- 审查器为建议性质 — 它们标记问题但不阻塞
- 若 agent 认为审查反馈有误，应在修复中说明理由
- 若同一问题经过 3 次迭代后分歧仍存在，上报人类

**审查器输出格式错误：**
- 控制器应验证审查器输出包含必需字段（Status、Issues 如适用）
- 若格式错误，携带预期格式说明重新调度审查器
- 2 次格式错误响应后，上报人类

## 需变更的文件

**新增文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改文件：**
- `skills/brainstorming/SKILL.md` — 在规格编写完成后添加审查循环
- `skills/writing-plans/SKILL.md` — 添加逐块审查循环，更新任务语法示例