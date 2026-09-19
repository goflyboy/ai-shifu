# AI-Shifu ExecPlan

本文是 [`PLANS.md`](PLANS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本仓库使用执行计划（“ExecPlan”）处理复杂功能、架构变更和耗时数小时的重构。`PLANS.md` 是这些计划如何撰写和维护的权威规范。

## 何时使用 ExecPlan

在工作满足以下情况时使用 ExecPlan：

- 跨越多个模块或工程表面
- 改变仓库结构、架构或共享工作流
- 需要分阶段上线、验证，或必须在当前对话结束后仍然可读的设计决策

把进行中的计划存放在 `docs/exec-plans/active/<slug>.md`。只有在实现和验证都完成后，才把它们移到 `docs/exec-plans/completed/<slug>.md`。

## 必要结构

每份 ExecPlan 都必须自包含，并且必须包含这些小节：

- `## Purpose / Big Picture`
- `## Progress`
- `## Surprises & Discoveries`
- `## Decision Log`
- `## Outcomes & Retrospective`
- `## Context and Orientation`
- `## Plan of Work`
- `## Concrete Steps`
- `## Validation and Acceptance`
- `## Idempotence and Recovery`
- `## Interfaces and Dependencies`

## 工作规则

- 把该计划视为该主题的实现权威来源。
- 在工作进行期间，保持 `Progress`、`Surprises & Discoveries`、`Decision Log` 和 `Outcomes & Retrospective` 为最新。
- 把计划写到无状态编码代理或新工程师只靠仓库和这份 ExecPlan 文件就能继续任务。
- 把已解决的歧义记录进计划，而不是留在聊天里。
- 把验证表述为可观察行为，而不仅仅是内部代码编辑。

## Progress 格式

`## Progress` 必须使用带时间戳的 Markdown 复选框。

示例：

- [x] 2026-04-17 15:00 CST: Added the knowledge index generator.
- [ ] 2026-04-17 15:10 CST: Wire the generated inventory into the repository
  harness checker.

## 与其他文档的关系

- `ARCHITECTURE.md` 说明仓库知识和运行时表面位于何处。
- `docs/engineering-baseline.md` 仍然是稳定的工程手册。
- `AGENTS.md` 文件把贡献者路由到正确的本地规则和来源文档。

## 已退役工作流

仓库根目录的 `tasks.md` 已退役。不要再在那里创建新的任务清单。改用 `docs/exec-plans/` 下的 ExecPlan。