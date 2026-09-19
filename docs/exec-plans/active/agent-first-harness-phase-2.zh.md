# Agent-First Harness 第二阶段

本文是 [`agent-first-harness-phase-2.md`](agent-first-harness-phase-2.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本 ExecPlan 是一份活文档，必须与 `PLANS.md` 保持对齐。

## 目的 / 全局图景

把仓库从第一波 agent-first harness 升级到第二波：仓库治理、架构边界、默认本地可观测性、浏览器冒烟校验，以及周期性 harness 养护，都要明确、版本化，并被机械强制。

## 进度

- [x] 2026-04-17 18:40 CST：审计当前 harness 状态，确认现有知识/文档布局，并识别主要剩余缺口：缺少 CI 优先的 harness 门禁、没有可执行的架构边界检查器、没有默认本地可观测性栈，也没有定时文档/债务养护。
- [x] 2026-04-17 18:55 CST：新增第二阶段知识文档、边界规则，以及感知基线的架构检查器。
- [x] 2026-04-17 19:20 CST：用可观测性服务和后端埋点升级默认 Docker 开发栈。
- [x] 2026-04-17 19:45 CST：扩展冒烟诊断、新增 CI workflow，并生成 harness 健康报告。
- [x] 2026-04-17 20:10 CST：运行生成器/检查/测试，更新知识索引，并记录最终结果。

## 意外发现

- 观察：仓库已经完成了有意义的第一阶段迁移，包括 ExecPlan、生成的知识索引、Playwright 冒烟测试，以及 request-id 诊断。
  证据：`docs/design-docs/agent-first-harness.md`、`docs/exec-plans/completed/agent-first-harness-migration.md`、`scripts/check_repo_harness.py` 和 `src/web/e2e/smoke.spec.ts`。
- 观察：控制面强于运行时面。知识文档、生成镜像和 `pre-commit` 已经存在，但默认 CI workflow 还没有把 repo-harness 和 runtime-harness 当作一等作业。
  证据：`.github/workflows/` 包含后端/契约/格式/翻译作业，但没有 repo-harness 或 runtime-harness workflow。
- 观察：默认 Docker 开发栈仍会在冒烟套件变绿之前失败，因为迁移 `6b603528dac8_add_system_profile.py` 导入了已不存在的运行时 helper。
  证据：该迁移导入 `add_profile_i18n`，而它已不在 `src/api/flaskr/service/profile/profile_manage.py` 中。
- 观察：当前架构指引大多仍是文档性的。仓库有按路径作用域的 AI 指令，但没有针对前端路由耦合、后端跨服务导入，或新增直接请求路径的可执行边界检查器。
  证据：前端 ESLint 配置只有通用规则，后端在 `scripts/` 或 CI 中也没有架构断言脚本。
- 观察：即使加入了可观测性栈，OTEL traces 最初仍会失败，因为 collector 默认使用容器内的 `localhost` 监听。
  证据：collector 日志显示 `endpoint: "localhost:4318"`，在接收端被钉到 `0.0.0.0` 之前，Tempo 对新鲜请求 traces 返回 404。
- 观察：默认开发 `.env` 会漂向仅 Google 登录，这会破坏确定性冒烟鉴权，尽管 harness 期望手机登录加上通用验证码。
  证据：`docker/.env` 覆盖 `LOGIN_METHODS_ENABLED=google`，登录页只渲染 Google 按钮，直到 `docker-compose.dev.yml` 为开发 harness 钉死手机登录。
- 观察：一旦恢复手机登录，浏览器冒烟套件仍会抖动，因为重复的 `send_sms_code` 请求会撞上现有 SMS 频率限制，而通用验证码仍然有效。
  证据：`pw-1776419058232-admin-operations-page-loads` 和 `pw-1776419075062-learner-chat-shell-renders-and-c` 的请求级诊断捕获到 `/api/user/send_sms_code` 响应 `{"code":9999,"message":"SMS sent too frequently"}`。

## 决策记录

- 决策：通过用 Python 实现 AST 和静态导入扫描，让第二阶段边界检查器保持语言无关，并且只存在于仓库本地。
  理由：这让规则对代理可读，避免再加一套语言特定 lint 工具，并允许单个基线文件同时治理后端和前端。
  日期/作者：2026-04-17 / Codex
- 决策：保留迁移历史，并通过当前运行时 helper 的兼容行为修复失败的遗留迁移，而不是改写迁移文件本身。
  理由：现有迁移历史可能已在其他地方应用；更安全的兼容做法是恢复期望的遗留 helper 行为。
  日期/作者：2026-04-17 / Codex
- 决策：让 Langfuse 继续负责 LLM 特定 traces，并另加一套本地运行时可观测性栈，用于通用日志、指标和 HTTP traces。
  理由：仓库已经依赖 Langfuse 语义来服务学习流程；第二阶段需要独立的运行时诊断，而不能把 LLM 追踪栈当作通用 HTTP 可观测性的替代。
  日期/作者：2026-04-17 / Codex
- 决策：让默认开发栈包含可观测性服务，但保持 release/latest compose 文件不变。
  理由：用户明确选择了完整闭环路径，而第二阶段范围是本地开发和 harness 诊断，不是生产部署。
  日期/作者：2026-04-17 / Codex
- 决策：在 `docker-compose.dev.yml` 中把开发 harness 钉到手机登录，而不是依赖可变的共享 `.env`。
  理由：浏览器冒烟套件依赖确定性鉴权，开发 harness 不应继承共享本地配置中与 OAuth 相关的覆盖。
  日期/作者：2026-04-17 / Codex
- 决策：把 SMS 频率限制视为手机登录中的可恢复 UI 状态，并在后端报告 `smsSendTooFrequent` 时继续保留 OTP 输入路径。
  理由：先前验证码仍然有效，通用验证码在开发中仍可用；禁用 OTP 输入会让冒烟自动化和真实用户重试流程都变得不必要地脆弱。
  日期/作者：2026-04-17 / Codex

## 结果与回顾

- 新增第二阶段控制面资产：`docs/design-docs/agent-first-harness-phase-2.md`、`docs/references/architecture-boundaries.md`、`docs/generated/harness-health.md`、`docs/generated/harness-gardening-summary.md`，以及这份活跃 ExecPlan。
- 新增 `scripts/check_architecture_boundaries.py`、fixture 覆盖、已提交基线、`pre-commit` 接线，以及新的 repo/runtime/gardening workflow。
- 把 `docker/docker-compose.dev.yml` 升级为默认可观测性 harness，包含 Grafana、Loki、Tempo、Prometheus、OTEL collector 和 Promtail，同时保持 release/latest compose 文件不变。
- 新增后端请求级 traces、HTTP 指标、稳定结构化日志字段，以及更丰富的 `harness_diagnostics.py` 摘要，包含 Loki/Tempo/Prometheus/Grafana 提示。
- 通过新增自愈迁移残留修复步骤，以及遗留 profile 迁移路径的兼容行为，恢复默认开发启动。
- 在活动开发栈中，用 `X-Request-ID`、`trace_id`、Loki 匹配、Tempo trace 查找和 Prometheus 请求指标验证端到端请求关联。
- 通过在开发 harness 中钉死手机登录，并在 SMS 发送被限流时继续允许 OTP 输入，恢复确定性浏览器冒烟。

## 上下文与定位

第一阶段已经建立了仓库知识模型、根架构地图、ExecPlan 工作流、Playwright 冒烟套件，以及 request-id 诊断。第二阶段必须在不改变公开产品契约的前提下，补上四套缺失的控制系统：

- CI 优先的仓库 harness 治理
- 感知基线的可执行边界检查
- 开发栈的默认本地可观测性
- 定时 harness 养护和健康报告

最脆弱的运行时阻塞点是后端启动路径中失败的遗留迁移，它目前会阻止冒烟套件在默认 Docker 开发栈中变绿。

## 工作计划

1. 新增第二阶段设计文档、参考文档，以及生成健康报告的管道，让这次变更有版本化的事实来源。
2. 实现感知基线的架构检查器，并把它接到 `pre-commit`、repo-harness 校验和 CI。
3. 新增后端可观测性原语，以及 Docker 开发可观测性栈。
4. 升级冒烟诊断，并新增 repo/runtime/gardening GitHub workflow。
5. 重新生成文档、运行定向校验，并用最终结果更新这份 ExecPlan。

## 具体步骤

- 新增 `docs/design-docs/agent-first-harness-phase-2.md`。
- 在 `docs/references/` 下新增长期有效的边界参考。
- 新增 `scripts/check_architecture_boundaries.py`、fixture 数据，以及 `docs/generated/` 下已提交的基线文件。
- 扩展 `scripts/build_repo_knowledge_index.py` 和 `scripts/check_repo_harness.py`。
- 新增后端兼容修复、指标/追踪支持，以及扩展诊断。
- 在 `docker/observability/` 下新增可观测性配置文件。
- 更新 `docker/docker-compose.dev.yml`、冒烟诊断和 GitHub workflow。

## 校验与验收

- `python scripts/generate_ai_collab_docs.py`
- `python scripts/build_repo_knowledge_index.py`
- `python scripts/check_repo_harness.py`
- `python scripts/check_architecture_boundaries.py --run-fixture-tests`
- `cd docker && docker compose -f docker-compose.dev.yml config`
- `cd src/web && npm run test:e2e`

## 幂等与恢复

知识生成器、边界基线和健康报告，对固定仓库状态必须是确定性的。如果默认运行时 harness 在兼容修复后仍不能变绿，最终摘要必须说明阻塞点是迁移相关、环境相关，还是由新的可观测性栈引起。

最终已校验状态是：

- 默认开发栈中，同一请求的请求级日志、指标和 traces 可以解析；
- `python scripts/check_repo_harness.py` 通过；
- `python scripts/check_architecture_boundaries.py --run-fixture-tests` 通过；
- `cd src/web && npm run test:e2e` 针对默认开发栈通过。

## 接口与依赖

- `docs/generated/architecture-boundary-baseline.json` 成为现有边界违规的已提交基线。
- `docs/generated/harness-health.md` 成为 harness 控制面的生成高层健康报告。
- `scripts/check_architecture_boundaries.py` 成为新的仓库级检查。
- `docker/docker-compose.dev.yml` 仅增加本地可观测性服务。
- `.github/workflows/repo-harness.yml`、`.github/workflows/runtime-harness.yml` 和 `.github/workflows/harness-gardening.yml` 成为新的 harness workflow。
