---
name: requesting-code-review
description: 完成任务、实现主要功能或合并前使用，以验证工作是否满足需求
---

# 请求代码审查

派发代码审查子代理，在问题扩散前发现它们。审查者获得精心构造的上下文用于评估——绝不使用你的会话历史。

**核心原则：** 尽早审查，经常审查。

## 何时请求审查

**强制：**
- 子代理驱动开发中每个任务完成后
- 完成主要功能后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（全新视角）
- 重构前（基线检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或：git merge-base origin/main HEAD
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派发代码审查子代理：**

派发一个 `general-purpose` 子代理，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` - 你构建内容的简要总结
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交

**3. 根据反馈行动：**
- 立即修复 Critical 问题
- 继续前修复 Important 问题
- 记录 Minor 问题留待后续
- 如果审查者有误，驳回（附理由）

## 示例

```
[刚完成任务 2：添加验证函数]

你：继续前让我请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | cut -d' ' -f1)
HEAD_SHA=$(git rev-parse HEAD)

[派发代码审查子代理]
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，含 4 种问题类型
  PLAN_OR_REQUIREMENTS: docs/superpowers/plans/deployment-plan.md 中的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[子代理返回]：
  优点：架构清晰，真实测试
  问题：
    Important：缺少进度指示器
    Minor：报告间隔使用魔法数字（100）
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 常见借口

| 借口 | 现实 |
|--------|---------|
| "我直接自己审查 diff 就行，不用派发审查者" | 你是协调者——内联审查 diff 会消耗你继续推进工作所需的上下文窗口。派发审查子代理：diff 和评估都在它的上下文中，只有发现结果返回给你。 |
| "审查者需要我的整个会话历史才能理解变更" | 交给它精心构造的上下文，绝不使用你的会话历史。这样审查者关注工作产物，而非你的思考过程。 |

## 危险信号

**绝不：**
- 因为"很简单"跳过审查
- 忽略 Critical 问题
- 带着未修复的 Important 问题继续
- 与有效的技术反馈争辩

**如果审查者有误：**
- 用技术理由驳回
- 展示证明其可行的代码/测试
- 请求澄清

参见模板：[code-reviewer.md](code-reviewer.md)