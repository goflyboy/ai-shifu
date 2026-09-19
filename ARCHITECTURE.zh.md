# AI-Shifu 架构地图

本文是 [`ARCHITECTURE.md`](ARCHITECTURE.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 目的

本文是面向人和编码代理的顶层地图。它说明仓库知识存放在哪里、主要产品表面如何彼此关联，以及哪些文件是设计、执行和校验的权威来源。

## 系统表面

- `src/api/`：Flask 后端、服务模块、持久化、供应商集成、共享后端测试，以及后端维护脚本。
- `src/web/`：面向学员流程、运营/管理工具、课程编写，以及共享前端测试的 Next.js 前端。
- `src/i18n/`：后端和前端共同使用的共享翻译清单。
- `docker/`：本地开发、latest 镜像，以及固定版本发布的运行时打包。
- `scripts/`：仓库维护、生成和校验脚本。
- `scripts/markdownflow-arena/`：手动调用的本地幻灯片对比工具，拥有独立依赖和测试；调用现有运行时组件，不启动应用，也不改动源码。
- `.github/`：CI、发布自动化、issue 模板，以及 Copilot 兼容指令。

## 仓库知识模型

- `AGENTS.md`：目录级入口和硬约束
- `PLANS.md`：权威 ExecPlan 规范
- `docs/engineering-baseline.md`：稳定的工程手册
- `docs/design-docs/`：实现与架构决策
- `docs/product-specs/`：产品和流程行为规格
- `docs/references/`：长期有效的参考文档和操作指南
- `docs/exec-plans/active/`：当前以活文档 ExecPlan 跟踪的复杂工作
- `docs/exec-plans/completed/`：已归档的执行计划
- `docs/generated/`：生成的知识索引和清单
- `docs/generated/harness-health.md`：生成的 harness 资产摘要和边界基线状态

## 运行时流程

### 后端

- HTTP 流量从 `src/api/app.py` 进入 Flask 应用。
- 共享请求日志和 request-id 传递位于 `src/api/flaskr/common/log.py`。
- Langfuse tracing 辅助位于 `src/api/flaskr/api/langfuse.py`。
- 业务行为按模块分组在 `src/api/flaskr/service/<module>/`。

### 前端

- 路由入口位于 `src/web/src/app/**/page.tsx`、`layout.tsx` 和 `route.ts`。
- 共享请求传输和业务码处理位于 `src/web/src/lib/request.ts` 和 `src/web/src/lib/api.ts`。
- 学员和老师代码共用按职责划分的源码目录：`api`、`assets`、`components`、`constants`、`hooks`、`lib`、`store` 和 `types`。
- 共享分析使用 `hooks/useTracking.ts` 和 `lib/tracking.ts`；课程编排辅助位于 `lib/shifu`。

## Harness 模型

- 仓库知识必须能从已版本化的文件中发现，而不是依赖聊天历史或未成文约定。
- 复杂工作用 ExecPlan 跟踪，而不是临时的分支笔记。
- 生成索引和校验脚本让漂移可见，并尽早失败。
- 架构边界漂移被冻结在已提交的基线中，并由 `scripts/check_architecture_boundaries.py` 检查。
- 浏览器冒烟测试加上 request-id 诊断，构成 UI 工作的最低代理校验循环。
- 默认 Docker 开发栈现在包含本地可观测性服务，因此运行时失败可以在日志、trace 和指标之间关联。