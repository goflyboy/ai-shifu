# 仓库知识库

本文是 [`README.md`](README.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本仓库把版本化文件视为产品意图、工程规则和长期执行上下文的系统记录。

## 布局

- `../ARCHITECTURE.md`：产品表面和知识入口的顶层地图
- `../PLANS.md`：复杂工作的权威 ExecPlan 规范
- `engineering-baseline.md`：稳定的工程手册
- `QUALITY_SCORE.md`：当前质量等级和下一步清理动作
- `RELIABILITY.md`：当前校验环和可靠性约束
- `SECURITY.md`：harness 与诊断工作的仓库安全规则
- `design-docs/`：架构与实现决策记录
- `product-specs/`：产品流程和页面行为规格
- `references/`：长期有效的操作参考
- `exec-plans/active/`：当前进行中的 ExecPlan
- `exec-plans/completed/`：已归档的 ExecPlan
- `generated/`：生成的索引和清单文件
  包括 `doc-inventory.md`、`harness-health.md`，以及已提交的架构边界基线。

## 工作流

- 复杂工作必须从 `exec-plans/active/` 下的 ExecPlan 开始。
- `PLANS.md` 定义了 ExecPlan 的必要结构和维护规则。
- 生成的知识文档由脚本拥有，不得手工编辑。
- 架构边界规则位于 `references/architecture-boundaries.md`，已提交的基线由 `python scripts/check_architecture_boundaries.py` 检查。
- 历史扁平主题文档已退役；新文档应放到与其所有权和用途匹配的目录中。
