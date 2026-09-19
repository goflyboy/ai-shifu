# ExecPlan：计费积分通知

本文是 [`billing-credit-notifications.md`](billing-credit-notifications.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 目的 / 全局图景

实现积分通知中心 v1，让老师在积分到账、积分即将过期、余额较低时收到可运营配置的通知。v1 首个渠道是短信，但实现必须按通知中心抽象建设，保留后续站内信、邮件、飞书等渠道扩展空间。

本计划的来源文档是：

- `docs/billing-credit-notifications.md`
- `docs/billing-credit-notifications-technical-design.md`

核心边界：积分通知只记录和投递通知事实，不改变积分发放、扣减、过期或余额事实。账务真相仍以 `credit_ledger_entries`、`credit_wallet_buckets`、`credit_wallets` 为准。

## 进度

- [x] 2026-05-21 16:21 CST：根据需求和技术设计文档创建活跃 ExecPlan。
- [x] 2026-05-21 17:07 CST：新增 `notification_records` 数据模型、迁移、常量和默认禁用的配置种子。
- [x] 2026-05-21 17:07 CST：实现计费通知暂存、扫描、投递、重新入队、策略校验、dry-run、成本估算和指标事件。
- [x] 2026-05-21 17:07 CST：为积分发放、即将过期积分、低余额和短信投递接入 Celery 任务；保留 `billing.send_low_balance_alert` 作为兼容入口。
- [x] 2026-05-21 17:07 CST：新增运营 API 和前端页面，覆盖通知记录、策略配置、dry-run 和重试。
- [x] 2026-05-21 17:07 CST：在前端和后端预览/调试路径上，新增面向老师的余额状态以及 softlimit 调试阻断。
- [x] 2026-05-21 17:07 CST：新增定向后端/任务/前端测试并运行校验；架构边界检查仅被工作区中无关的既有未跟踪路由支持文件阻塞。
- [x] 2026-05-22 18:35 CST：用基于已完成日账本消耗的预估天数阈值扩展低余额通知，并补充结构化运营表单字段、dry-run 详情和定向测试。
- [x] 2026-06-19 21:20 CST：`refactor/billing-credit-notifications` 上的跟进工作清理了记录/配置的请求状态处理、重新入队刷新行为，以及配置侧体验，且未改变通知中心核心模型。
- [x] 2026-06-19 21:36 CST：把下一步的 `CreditNotificationConfigTab.tsx` 拆分（dry-run、模板同步、托管列表对话框状态隔离）推迟到单独的跟进 PR，使请求状态打磨分支保持可评审。
- [x] 2026-06-21 16:10 CST：后续运营细化先拆分记录总览/筛选 UI，再把 `CreditNotificationConfigTab` 的本地模板/输入/托管列表状态移进专用 hook，使配置 tab 保持编排导向，且不改变后端契约。
- [x] 2026-06-21 16:24 CST：在完成本地状态提取后结束本轮跟进；`dry-run` 和模板同步的更深 hook 拆分仍明确推迟到后续积分通知 PR，以保持范围可评审。
- [x] 2026-06-21 17:05 CST：一次配置 tab 跟进把 `dry-run` 和模板同步状态提取到专用前端 hook，使它们的错误留在配置区域本地，而不是复用页面级状态。
- [x] 2026-08-31 17:58 CST：模板管理评审整改保留阿里云页面请求 ID，在 UI 中串行化手工库刷新，新增提供商模板工作流分析，并补齐泰语详情标签。
- [x] 2026-08-31 18:56 CST：关闭模板管理数据完整性缺口：当通知配置加载失败时，绑定关系现在会报告不可用；提供商回退模板列表会返回每一条已同步的本地模板，而不是静默把结果限制为 100 条。
- [x] 2026-09-01 16:10 CST：用托管通知规则列表替换固定三行通知类型配置。运营可以创建、编辑、启用、禁用和删除命名短信规则，规则包含受支持的触发事件、模板和事件特定条件；全局投递保护保持独立。

### 托管通知规则 PR 拆分

- **PR 1：规则运行时和 API 契约** 留在 `feat/notification-rule-management`。它负责 `rules[]` 校验、遗留 `types` 兼容、感知规则的模板校验、对 `credit_granted`、`credit_expiring` 和 `low_balance` 的多规则匹配、规则作用域去重键、通知记录规则快照，以及定向后端回归覆盖。它不改运营配置 UI。
- **PR 2：运营规则管理 UI** 只在 PR 1 合入 `main` 后开始。它用规则列表和创建/编辑/启用/删除交互替换固定通知类型编辑器，同时保留独立的全局投递保护控件。它包含 i18n、分析和定向前端覆盖。
- 每个 PR 在推送前保持为一个打磨过的功能提交。不要推送中间实现检查点。

## 意外发现

- 现有订阅购买短信已经在 `src/api/flaskr/service/billing/notifications.py` 使用异步计费通知模式，但它把状态存在 `BillingOrder.metadata` 中；本功能需要更通用的 `notification_records` 表，因为 `credit_expiring` 和 `low_balance` 不属于单个订单。
- 现有 `billing.send_low_balance_alert` 当前会产出告警候选。它应作为兼容入口保留，同时委托给新的低余额通知扫描。
- `docs/exec-plans/index.md` 由 `scripts/build_repo_knowledge_index.py` 生成；新增本计划时不要手工编辑它。
- 第一版过期桶扫描草稿使用 `effective_to <= now + window`，导致还有 1 天到期的桶会在同一次扫描中匹配 7d/3d/1d 窗口。最终扫描器使用按窗口的日区间，因此每个桶在该次运行中只匹配预期提醒窗口。
- `python scripts/check_architecture_boundaries.py` 仍会报告本工作区中已存在的无关未跟踪 `learn/http`、`shifu/http` 和 `route_support.py` 边界违规；积分通知导入已调整，避免新增边界漂移。

## 决策记录

- 使用 `notification_records` 作为表名，而不是积分特定名称，以便未来非积分通知类型可以复用该表。
- v1 渠道是 `sms`；表仍保留 `channel` 列。
- 把 v1 策略存在 `sys_configs` 的 `BILL_CREDIT_NOTIFICATION_SMS_CONFIG` 下，默认禁用。
- 把阿里云提供商凭据和签名密钥保留在现有 env/config 路径；运营配置只存储通知策略和模板绑定。
- 使用固定去重键：
  - `credit_expiring:{wallet_bucket_bid}:{window}`
  - `credit_granted:{ledger_bid}`
  - `low_balance:{creator_bid}:{threshold}:{date}`
  - `low_balance:{creator_bid}:estimated_days:{days}:lookback:{lookback_days}:{date}`
- `low_balance` 支持 `fixed` 和 `estimated_days` 阈值策略；`estimated_days` 只读取 `bill_daily_ledger_summary` 中完整的先前自然日。
- v1 中 `softlimit.threshold` 仍只支持固定值，因此日消耗估算不能改变调试阻断行为。
- 不要复用 `/api/user/send_sms_code`、`/api/user/console_send_sms_code` 或验证码模板。
- softlimit 在前端和后端都强制：前端禁用老师调试入口，后端调试/预览 API 校验 `debug_allowed`。
- hardlimit 仍是计费准入/运行时边界，不受通知策略控制。
- 把 `credit_granted`、`credit_expiring` 和 `low_balance` 视为固定 **触发事件**，而不是运营自定义通知类型。运营管理命名 **通知规则**，每条规则选择一个受支持的触发事件。新的任意触发事件需要单独的后端实现，不能由配置 UI 暗示出来。
- 把全局投递保护（频率、免打扰时段、黑名单、退订和日预算）保留在共享通知策略中。规则级配置拥有渠道、提供商模板、启用状态，以及事件特定触发条件。
- 把初始托管规则列表作为结构化 `rules[]` 数据存在 `BILL_CREDIT_NOTIFICATION_SMS_CONFIG` 中。这是运行时配置变更，不是 schema 迁移。给每条规则一个稳定生成的业务 ID，并在现有值被表示为规则之前，把遗留 `types` 策略保留为可读兼容来源。
- 通知记录必须在 `policy_snapshot_json` 中快照匹配规则 ID 和显示名。每条通知去重键必须包含规则 ID，这样同一源事件上两条已启用规则都能投递，而不会被误判为重复。
- 国内首发支持带已审核阿里云模板的 `sms`。规则模型必须保留 `channel` 和提供商/模板引用，以便海外邮件模板跟进可以复用同一套配置和触发模型。

## 结果与回顾

已实现积分通知中心 v1：

- 持久化 `notification_records` 表，以及默认禁用的 `BILL_CREDIT_NOTIFICATION_SMS_CONFIG` 策略。
- 仅短信的通知服务，包含策略校验、去重、暂存、投递、跳过状态、失败提供商重新入队、dry-run 和短信成本估算。
- 在付费/手工/试用发放流程中接入 `credit_granted` 钩子，并为 `credit_expiring` 和 `low_balance` 增加定时扫描任务。
- 运营 API 和前端页面，覆盖记录搜索、结构化策略配置、dry-run 和失败提供商重新入队。
- 低余额提醒可以按预估剩余天数可选触发，使用已完成的日消耗汇总。固定阈值回退只在存在部分有效消耗历史时应用；缺失或为零的日消耗汇总不会发送预估天数提醒。
- 计费总览暴露 `credit_status`、`debug_allowed` 和 `softlimit_threshold`；预览/调试路径在前端和后端都强制 softlimit。
- 定向后端、任务和前端测试覆盖暂存、去重、跳过、提供商重试、扫描窗口、softlimit、Celery 调度/配置、运营页面渲染，以及前端预览阻断。
- 后续运营细化现在会分别限定记录/配置/dry-run 错误范围，在重新入队后并行刷新记录和总览，通过把本地编辑器和托管列表状态隔离到专用前端 hook 保持积分通知配置 tab 可维护，并把 `dry-run` 加模板同步状态移进专用前端 hook，使配置动作留在配置体验本地，且不改变契约或记录 tab 行为。

## PR2B 模板管理分析契约

- 业务问题：运营是否发现国内短信模板库，并成功完成提供商同步。
- 指标定义：在每个运营会话中，计数一次符合条件的模板库曝光、每一次被接受的手工同步尝试，以及每次尝试的一个终态结果。
- 事件名：`operator_notification_template_library_viewed`、`operator_notification_template_sync_attempt`、`operator_notification_template_sync_result`、`operator_notification_template_filter_applied` 和 `operator_notification_template_detail_opened`。
- 行动者与表面：已认证运营，位于积分通知模板管理 tab；排除邮件渠道占位视图。
- 触发与去重：曝光在每次已挂载的符合条件 tab 视图上触发一次；同步尝试在刷新守卫接受点击后触发；结果在该请求结束后触发一次；筛选和详情事件不去重，因为重复运营动作是有意义的。
- payload 允许列表：`channel`（`sms`）、`provider`（`aliyun`）、`source`（`provider` 或 `local`）、`outcome`（`success` 或 `failed`）和 `filter`（`keyword` 或 `status`）。不要发出模板内容、模板码、名称、用户标识符、联系信息或提供商请求 ID。
- 消费者：运营通知中心采纳，以及提供商同步可靠性报告。这是一组新的增量事件族。

## PR2 托管规则分析契约

- 业务问题：运营是否在配置和维护托管短信通知规则。
- 事件名：`operator_notification_rule_action`。
- 行动者与表面：已认证运营，位于积分通知配置 tab。
- 触发：在运营于本地策略草稿中创建、编辑、删除或改变规则启用状态后发出。该事件不代表已持久化保存。
- payload 允许列表：`channel`（`sms`）、`action`（`created`、`edited`、`deleted` 或 `toggled`）和 `trigger_event`（三个受支持事件之一）。不要发出规则 ID、规则名、模板码或内容、用户标识符、手机号或策略条件。
- 消费者：运营配置采纳报告。这是增量的，且绝不能阻塞配置工作流。
- 校验：定向前端测试覆盖符合条件的曝光、被接受的同步尝试和结果、筛选/详情事件、payload 允许列表，以及不改变用户工作流的分析失败。

## 上下文与定位

相关来源文档：

- 需求：`docs/billing-credit-notifications.md`
- 技术设计：`docs/billing-credit-notifications-technical-design.md`
- 现有购买短信设计：`docs/billing-subscription-purchase-sms.md`
- 手工发放语义：`docs/operator-user-points-grant.md`
- 计费钱包和桶设计：`docs/billing-subscription-design.md`

可能的后端表面：

- 计费模型和常量：`src/api/flaskr/service/billing/models.py`、`src/api/flaskr/service/billing/consts.py`
- 计费通知先例：`src/api/flaskr/service/billing/notifications.py`
- 计费任务：`src/api/flaskr/service/billing/tasks.py`
- Celery beat 调度：`src/api/flaskr/common/celery_app.py`
- 配置读写：`src/api/flaskr/service/config`
- 短信提供商 helper：`src/api/flaskr/api/sms/aliyun.py`
- 运营 API：`src/api/flaskr/service/shifu/admin.py`、`src/api/flaskr/service/shifu/admin_dtos.py`、`src/api/flaskr/service/shifu/route.py`

可能的前端表面：

- 运营菜单：`src/web/src/app/admin/admin-menu.tsx`
- 运营页面和类型：`src/web/src/app/admin/operations`
- 前端 API 映射：`src/web/src/api/api.ts`
- 面向用户字符串：`src/i18n/`

## 工作计划

1. 新增持久化通知模型和默认禁用的策略配置。
2. 构建计费通知服务，负责策略解析、去重、暂存、跳过决策、投递、dry-run 和重新入队。
3. 为 `credit_granted`、`credit_expiring` 和 `low_balance` 接入事件和扫描触发。
4. 暴露记录、配置、dry-run 和重试的运营 API。
5. 新增带 i18n 标签的运营前端页面。
6. 新增面向老师的限额状态和 softlimit 调试阻断。
7. 新增定向后端、任务、前端和仓库校验覆盖。
8. 按禁用、dry-run、小流量和扫描启用阶段上线。

## 具体步骤

1. 在计费模型层新增 `notification_records`，包含：
   - `notification_bid`
   - `notification_type`
   - `channel`
   - `creator_bid`
   - `target_user_bid`
   - `mobile_snapshot`
   - `source_type` / `source_bid`
   - `dedupe_key`
   - `status`
   - 模板、策略、提供商响应、错误和时间戳字段
   - `notification_bid` 和 `dedupe_key` 的唯一索引
   - 状态/类型/时间、创作者/时间，以及来源查找的查询索引
2. 新增通知常量，并以 `enabled=false` 种子化 `BILL_CREDIT_NOTIFICATION_SMS_CONFIG`。
3. 新增 `src/api/flaskr/service/billing/credit_notifications.py`，包含：
   - 策略加载/校验 helper
   - 去重键构建器
   - `stage_credit_granted_notification`
   - `scan_credit_expiring_notifications`
   - `scan_low_balance_notifications`
   - `deliver_credit_notification`
   - `requeue_credit_notification`
   - `dry_run_credit_notifications`
   - `resolve_creator_limit_state`
4. 新增 Celery 任务：
   - `billing.scan_credit_expiring_notifications`
   - `billing.scan_low_balance_notifications`
   - `billing.send_credit_notification`
   - 来自 `billing.send_low_balance_alert` 的兼容包装或委托
5. 只在账本和桶事实提交后，把 `credit_granted` 接入成功的积分发放流程。
6. 新增运营 API DTO 和路由，覆盖：
   - 列出/筛选通知记录
   - 获取/更新策略配置
   - dry-run
   - 重新入队 `failed_provider`
7. 新增运营前端页面：
   - 菜单入口“积分通知”
   - 记录表和筛选
   - 失败详情
   - 单条记录重新入队
   - 策略配置表单
   - dry-run 结果面板
8. 新增面向老师的余额状态：
   - 暴露 `normal`、`softlimit`、`hardlimit`
   - 暴露 `debug_allowed`
   - 为 softlimit 禁用前端调试入口
   - 用同一策略守卫后端调试/预览 API
9. 新增可观测性：
   - 生成计数
   - 发送计数
   - 提供商失败计数
   - 按原因的跳过计数
   - 重复抑制计数
   - 重新入队计数
   - 短信成本估算
10. 更新测试，并在每一步落地时保持本 ExecPlan 进度最新。
11. 用通知规则列表和规则编辑器替换固定通知类型编辑器：
   - 列表列：规则名、触发事件、渠道/模板、条件摘要、启用状态和动作
   - 创建/编辑字段：规则名、受支持触发事件、已审核模板、启用状态，以及事件特定条件
   - `credit_granted` 没有运营定义阈值；`credit_expiring` 配置提醒窗口；`low_balance` 配置固定或预估天数阈值
   - 保留独立的全局投递保护区，覆盖频率、免打扰时段、黑名单、退订和预算
12. 改变运行时匹配，使每个受支持事件的每条已启用规则都能独立暂存通知。保留现有事件钩子和定时任务；只把它们的策略查找从固定类型条目改为匹配规则。
13. 新增遗留策略兼容：只包含现有 `types` 结构的配置必须继续按今天的行为工作，并在第一次托管规则保存时呈现为对应的初始规则。不要静默丢弃现有启用状态、模板、过期窗口或低余额阈值。

## 校验与验收

- `credit_granted`、`credit_expiring` 和 `low_balance` 记录用稳定去重键创建，且重试时不重复。
- 通知投递记录 `sent`、`skipped_no_mobile`、`skipped_opt_out`、`suppressed_duplicate` 或 `failed_provider`，且不改变钱包、桶或账本事实。
- 提供商失败可以从运营表面重新入队；终态跳过状态除非策略变更明确允许，否则不重新入队。
- 运营用户可以列出/筛选记录、检查失败详情、更新策略配置、运行 dry-run，以及重新入队 `failed_provider`。
- 面向老师的表面接收 `normal`、`softlimit`、`hardlimit` 和 `debug_allowed`。
- softlimit 在前端和后端禁用老师调试；hardlimit 仍由计费准入/运行时行为强制。
- 面向用户的前端字符串存在 `src/i18n/` 中，而不是硬编码在组件里。
- 仅文档变更的最低校验：`python scripts/check_repo_harness.py`。
- 实现校验必须包含定向后端测试、任务测试、运营页面前端测试，以及共享契约变化时的架构边界检查。

## 幂等与恢复

- `dedupe_key` 唯一性是所有通知类型的主要重复防御。
- `credit_granted` 使用账本 BID，因此复用现有账本的重复发放请求不会再发一次通知。
- `credit_expiring` 使用桶 BID 加窗口，因此同一桶/窗口的重复扫描是安全的。
- `low_balance` 使用创作者、阈值和日期，因此重复日扫描不会向同一创作者刷屏。
- `low_balance` 预估天数规则使用创作者、触发天数、回看天数和日期，因此固定和自动阈值保持独立幂等。
- worker 投递应锁定记录，并且只处理 `pending` 或 `failed_provider`。
- 如果提供商调用失败，把记录保留在 `failed_provider`，并带上响应/错误详情以便重试。
- 无手机号、退订、黑名单、频率、预算和重复抑制是终态，除非策略明确变更且运营选择新动作。
- 通知失败不得回滚已经成功的计费事实。

## 接口与依赖

- 新持久化表：`notification_records`。
- 新策略键：`BILL_CREDIT_NOTIFICATION_SMS_CONFIG`。
- 新通知类型：`credit_expiring`、`credit_granted`、`low_balance`。
- v1 渠道：`sms`。
- 来源 ID：
  - `wallet_bucket_bid`
  - `ledger_bid`
  - `creator_bid`
- 低余额阈值策略变体：
  - `{ "kind": "fixed", "value": "..." }`
  - `{ "kind": "estimated_days", "days": 7, "lookback_days": 7, "min_consumed_days": 2, "fallback_fixed_value": "0" }`
- 新增或更新的任务入口：
  - `billing.scan_credit_expiring_notifications`
  - `billing.scan_low_balance_notifications`
  - `billing.send_credit_notification`
  - `billing.send_low_balance_alert` 兼容路径
- 新运营端点位于 `/shifu/admin/operations/credit-notifications`。
- 需要保留的现有依赖：
  - 阿里云短信 helper `send_sms_ali`
  - 计费账本、钱包和桶记账
  - 现有鉴权短信路由和验证码模板
  - 现有 hardlimit 准入行为
- 托管规则契约（首发）：
  - 持久化在 `BILL_CREDIT_NOTIFICATION_SMS_CONFIG.rules[]` 下，带有稳定的 `rule_bid`、`name`、`trigger_event`、`channel`、`template_code`、`enabled`，以及事件特定 `conditions`
  - 受支持的 `trigger_event` 值仍是 `credit_granted`、`credit_expiring` 和 `low_balance`
  - 规则级条件仅限对应运行时能评估的字段；运营 UI 不得暴露自由形式表达式或任意事件名
  - 这次配置模型变更不需要数据库 schema 迁移；记录在现有策略快照中保留规则来源
