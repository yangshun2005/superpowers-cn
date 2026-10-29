# Hermes 版本号升级接线设计

**日期：** 2026-08-05
**修订：** 2026-08-06
**状态：** 已批准

## 目标

让 `.hermes-plugin/plugin.yaml` 与仓库版本保持同步：在 `.version-bump.json` 中注册该文件，并让 `scripts/bump-version.sh` 能处理 YAML，而无需在 Bash 中实现 YAML 解析器。

## 设计

- 在 `.version-bump.json` 中添加 `{ "path": ".hermes-plugin/plugin.yaml", "field": "version" }`。
- `.json` 走现有的 `jq` 辅助函数，`.yaml` 走 Mike Farah `yq` v4。YAML 的键和值作为数据传入，不插值进表达式。
- 仅支持存在的顶层 YAML 字符串字段。嵌套字段和 `.yml` 不在范围内。
- `--check`、`--audit` 和版本更新都走同一个小的读/写分发器。
- 版本升级写入任何清单之前，先运行一次只读预检：校验所需工具，并通过分发器读取每个存在的已声明清单。这样可避免确定性的 YAML 失败发生在较早的 JSON 文件已更新之后。文件缺失行为不变，`--help` 仍可在没有 `jq` 或 `yq` 时使用。

预检是唯一的可靠性增强。它不会让脚本具备事务性，也不重新设计现有的审计和错误状态行为。

## 测试

三个聚焦的行为测试在隔离的临时夹具上运行真实脚本，证明：

- 对齐的 JSON 和 YAML 能通过 `--check` 和 `--audit`，且一次升级会同时更新两种格式；
- 当 JSON 声明在前、后续 YAML 清单的顶层 `version` 不是字符串时，实际升级会以非零状态退出，且每个清单逐字节保持不变；
- 真实的 `.version-bump.json` 注册了 Hermes 清单。

验证还会对仓库运行 shell lint 和 `scripts/bump-version.sh --check`。

## 非目标

- 不手写 YAML 解析器。
- 不支持 `.yml` 或嵌套 YAML。
- 不修改 Hermes 运行时。
- 不做回滚框架、通用配置 Schema 层、审计/状态重构或穷举失败矩阵。
- 不修改评审中发现的独立版本校验和 JSON 表达式问题。