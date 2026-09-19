---
title: Agent-First Harness 第二阶段
status: implemented
owner_surface: repo
last_reviewed: 2026-04-17
canonical: false
---

# Agent-First Harness 第二阶段

本文是 [`agent-first-harness-phase-2.md`](agent-first-harness-phase-2.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 摘要

把仓库从第一波 agent-first harness 扩展到第二波：仓库治理、可执行架构边界、默认本地可观测性，以及周期性 harness 养护，都要版本化并被强制执行。

## 目标

- 把 repo-harness 校验从仅本地卫生提升为 CI 优先门禁。
- 新增感知基线的架构检查器，覆盖前端和后端边界漂移。
- 把默认 Docker 开发栈升级为带可观测性的 harness。
- 新增周期性文档养护和债务养护自动化，以及生成的健康报告。

## 约束

- 保持公开 HTTP API、业务数据库契约和共享 i18n 契约不变。
- 保持 `AGENTS.md` 导航优先；新的长期知识必须放在 `docs/` 和生成产物中。
- 保持 `docker-compose.latest.yml` 和钉死的 release compose 不变。
- 保留现有 Langfuse 流程，并增加互补的运行时可见性，而不是替换它们。

## 决策

### Harness 治理

- 为第二阶段新增一份活跃 ExecPlan，并把它当作实现的事实来源。
- 用已提交的 harness 健康报告扩展生成知识系统。
- 让 `scripts/check_repo_harness.py` 感知新的 workflow、文档和生成 harness 资产。

### 架构边界

- 用单个 Python 检查器覆盖后端和前端边界。
- 使用已提交基线文件冻结现有债务，只有新增违规才会让常规检查失败。
- 第一波后端允许列表有意保持很小：`common`、`config`，以及稳定的 DTO/model/const 入口。
- 第一波前端路由规则有意保持很窄：禁止 `src/app/**` 中新增的路由内部耦合，并阻断新增的 `src/components/** -> src/app/**` 依赖。

### 运行时可观测性

- 默认本地开发包含 Grafana、Loki、Tempo、Prometheus、OTEL collector 和日志投递器。
- 后端运行时 traces 是请求级的，并把 `X-Request-ID` 作为稳定属性携带。
- HTTP 指标由后端通过本地 metrics 端点导出，并由 Prometheus 抓取。
- 现有文件日志保持原位，并获得稳定的 request/trace/status/duration 字段，以便 Promtail 可以投递它们，且不改变本地工作流。
- 无论共享本地 `.env` 文件中是否有可选 OAuth 覆盖，默认开发 compose 都必须钉死手机登录，以保证确定性冒烟运行。
- 默认 API 启动必须在重新运行 Alembic 之前修复已知的遗留迁移残留，使 harness 能从非事务 schema 漂移中恢复。

### 养护

- 新增定时/手工 GitHub workflow，扫描过期文档、退役 workflow 术语，以及过期边界基线条目。
- 在养护运行期间重新生成 harness 健康报告，并在检测到漂移时打开 GitHub issue。

## 交付物

- 活跃 ExecPlan 和第二阶段设计/参考文档
- `scripts/check_architecture_boundaries.py` 加上 fixture 和基线
- 生成的 `docs/generated/harness-health.md`
- 默认开发可观测性栈和扩展后的后端诊断
- Repo-harness、runtime-harness 和 harness-gardening GitHub workflow
