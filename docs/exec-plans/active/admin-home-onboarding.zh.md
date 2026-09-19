# 管理后台首页 Onboarding

本文是 [`admin-home-onboarding.md`](admin-home-onboarding.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 目的 / 全局图景

在管理后台首页流程退役后，继续维护保留下来的创作者 onboarding 契约。试用欢迎对话框是 `/admin` 上唯一的首次进入表面，同时仅限所有者的课程编辑器 onboarding 继续引导符合条件的老师。历史的 `admin_home_onboarding` 后端记录仍可读以保持兼容，但不再驱动前端 UI、回放、完成写入或分析事件。

## 进度

- [x] 2026-06-17 13:10 CST：新增后端 onboarding 持久化模型、服务、路由处理，以及定向 pytest 覆盖。
- [x] 2026-06-17 13:25 CST：新增引导课程解析，以及创作者课程列表 DTO 上的 `is_guide_course` 标记，并为新列表标记补充覆盖。
- [x] 2026-06-17 13:45 CST：把管理后台 layout 中的试用对话框用法替换为共享 onboarding overlay、步骤构建器、API 接线，以及 Umami 事件钩子。
- [x] 2026-06-17 14:05 CST：完成 PR1 运行时加固、定向前端覆盖，以及提交 / PR 前的最终校验。
- [x] 2026-06-17 14:40 CST：新增 `creator_activated_at`，让上线后才成为创作者的老用户仍有资格进入管理后台首页 onboarding。
- [x] 2026-06-17 22:20 CST：为首次符合条件的所有者编辑器入口新增仅限所有者的课程编辑器 onboarding，并加固共享 overlay，覆盖抽屉目标、圆角高亮孔、边缘内边距，以及 toast/onboarding 重叠。
- [x] 2026-06-18 10:30 CST：把管理后台首页 onboarding 更新为新的三步流程：空白课程创建、lobster AI 课程创建，以及带试用积分详情的完整计费卡片。
- [x] 2026-09-16：退役管理后台首页的创建和计费引导，使保留下来的试用欢迎对话框成为唯一的首次进入表面。后端完成记录仍可读以保持兼容。

## 意外发现

- 即使退役后的管理后台首页流程不再消费引导课程元数据，它仍可用于课程列表标注。
- 共享 onboarding hook 还需要在场景于会话中途变为禁用时关闭自己，否则路由变化会把 overlay 留在无关的管理后台页面上。
- 仅靠现有用户的 `created_at` 不足以判断资格，因为运营后续可以通过转移创作者、课程复制，或共享编辑/发布权限授予创作者能力。
- 课程编辑器设置位于 Radix Sheet 内，因此 onboarding 不能依赖静态目标坐标。抽屉步骤需要明确的打开/关闭所有权、阻止外部点击、先滚动再测量的行为，以及在渲染 overlay 前的短暂坐标稳定。
- 成功 toast 和编辑器 onboarding 都是顶层反馈表面。它们应被串行化而不是叠在一起，否则 toast 会在路由过渡期间从 onboarding overlay 中透出来。

## 决策记录

2026-09-16 退役条目之前的决策作为实现历史保留。最终退役决策覆盖它们对当前行为的约束。

- 决策：按创作者资格门控 onboarding，排除运营人员，并使用后端上线阈值配置（`ADMIN_ONBOARDING_ENABLED_FROM`），避免老用户自动进入该流程。
  - 原因：产品希望上线后只有新创作者用户看到 onboarding。
- 决策：使用 `user_users.creator_activated_at` 作为主要资格时间戳，仅在创作者激活时间缺失时回退到 `created_at`。
  - 原因：老的普通用户可以在上线后通过管理后台登录、运营转移创作者、运营课程复制，或共享编辑/发布权限成为创作者。
- 决策：PR2 编辑器 onboarding 仅限所有者，针对首次符合条件的所有者编辑器入口，无论用户来自手工创建课程、lobster 创建课程、课程列表，还是直接编辑器链接。共享权限用户在 PR2 中看不到编辑器 onboarding。
  - 原因：外部 lobster 入口可能无法返回可靠的来源参数，而编辑器步骤仍是面向所有者的设置，例如模型、听课模式、定价、预览和发布。
- 决策：把共享权限 onboarding 变体作为后续迭代，而不是强迫共享用户走所有者流程。
  - 原因：共享协作者通常需要更轻的协作路径，聚焦提示词编辑、调试和预览；面向所有者的设置和发布动作会增加噪音，或误导首次使用预期。
- 决策：从现有的中/英演示课程配置键解析引导课程，并通过创作者课程列表暴露 `is_guide_course`。
  - 原因：UI 必须高亮真实课程卡片，而不是再加一条独立推荐入口。
- 决策：把引导课程解析保留在后端/列表 DTO 中，但从管理后台首页 onboarding 中移除引导课程步骤。
  - 原因：修订后的产品流程应只介绍课程创建、lobster 辅助课程创建，以及积分/套餐管理。
- 决策：lobster 辅助创建步骤可以在 onboarding 卡片内包含一个动作链接。
  - 原因：现有首页链接可以被高亮，但卡片文案也需要一种直接方式，在新标签页打开同一个外部课程创建 URL，且不推进 overlay。
- 决策：把可复用的 onboarding overlay 和目标解析逻辑保留在共享前端模块中。
  - 原因：PR2 会复用同一套流程机制做编辑器 onboarding。
- 决策：保留创建课程成功 toast，但把跳转到编辑器的导航延迟到短 toast 时长结束之后。
  - 原因：产品希望保留成功反馈，同时编辑器 onboarding 不得与来自上一路由的过期 toast 在视觉上重叠。
- 决策：使用基于目标外阴影的共享圆角高亮实现，而不是 SVG 或矩形遮罩切片。
  - 原因：它能在管理后台首页和编辑器目标上给出一致的圆角孔，包括靠近视口边缘的目标，以及位于基于 portal 的抽屉内的目标。
- 决策：停止产出管理后台首页 onboarding UI 和分析事件，同时保留后端场景契约和课程编辑器 onboarding。
  - 原因：管理后台首页已经自解释，而且它的 overlay 会与产品选择保留的试用欢迎对话框竞争。

## 结果与回顾

- 管理后台首页 walkthrough、其目标锚点、菜单入口、本地化文案、完成写入和分析生产者均已退役。试用欢迎对话框仍是 `/admin` 上唯一的自动首次进入表面。
- 延期跟进：在所有者流程落地后，新增共享权限编辑器 onboarding 场景。第一候选范围是轻量三步路径，只覆盖提示词编辑、调试和预览，有意排除课程设置和发布。
- PR2 所有者编辑器 onboarding 现在覆盖提示词编辑、调试、添加课时、设置入口、模型、听课模式、价格、预览和发布。设置抽屉步骤在 onboarding 期间保持抽屉打开，并在流程离开设置面板后关闭它。直接编辑器进入会以 `trigger_source=editor_entry` 记录；当存在手工和 lobster 来源参数时仍会保留。
- 跟进：另开一个法语 i18n 打磨 PR，规范化 `src/i18n/fr-FR/**` 中带重音的法语。本 PR 只修正 onboarding 字符串，避免把大范围文案清理和 onboarding 行为变更混在一起。
- 管理后台首页 onboarding 已于 2026-09-16 退役。试用欢迎对话框保持启用，菜单回放入口被隐藏，底层课程编辑器回放状态以及现有后端完成行被有意保持不动。

## 上下文与定位

- 后端归属路径：
  - `src/api/flaskr/service/user/onboarding.py`
  - `src/api/flaskr/route/user.py`
  - `src/api/flaskr/service/user/models.py`
  - `src/api/flaskr/service/shifu/demo_courses.py`
  - `src/api/flaskr/service/shifu/dtos.py`
  - `src/api/flaskr/service/shifu/shifu_draft_funcs.py`
- 前端归属路径：
  - `src/web/src/app/admin/layout.tsx`
  - `src/web/src/app/admin/page.tsx`
  - `src/web/src/components/onboarding/editorOnboardingSteps.ts`
  - `src/web/src/components/onboarding/OnboardingOverlay.tsx`
  - `src/web/src/components/shifu-edit/ShifuEdit.tsx`
  - `src/web/src/hooks/useOnboarding.ts`
  - `src/web/src/lib/onboardingTargets.ts`
  - `src/web/src/store/onboardingReplayStore.ts`

## 工作计划

1. 让退役的管理后台首页场景继续缺席前端渲染、回放、完成和分析路径。
2. 保留试用欢迎对话框，以及仅限所有者的课程编辑器 onboarding。
3. 在前端消费者独立迁移时，保持历史后端场景记录和响应字段兼容。

## 具体步骤

1. 验证 `/admin` 挂载试用欢迎对话框，且不启动 onboarding overlay，也不发出管理后台首页 onboarding 事件。
2. 验证课程编辑器仍会门控、渲染、完成，并可选回放其保留下来的 onboarding 场景。
3. 把 `admin_home_onboarding` 保留在后端兼容类型和已存储记录中，但不要新增前端消费者。
4. 当该契约变化时，运行定向 Jest 覆盖、类型检查、翻译检查，以及仓库 harness。

## 校验与验收

- `/admin` 不渲染创建按钮或计费卡片 onboarding。
- 当现有发放和确认规则满足时，`/admin` 继续显示试用欢迎对话框。
- 用户菜单不暴露已退役的 onboarding 回放入口。
- 不产生管理后台首页 onboarding 完成请求或分析事件。
- 历史 `admin_home_onboarding` 记录和状态字段仍可读。
- 编辑器设置步骤保持设置抽屉打开，阻止来自 onboarding 点击的外部关闭，并只在抽屉目标坐标稳定后渲染。
- 手工创建课程仍显示短暂成功 toast，然后过渡到编辑器 onboarding，且视觉层不重叠。
- 完成或回放课程编辑器 onboarding 只更新 `course_editor_onboarding`。
- 定向前端 Jest/类型检查、翻译和仓库 harness 检查通过。

## 幂等与恢复

- 历史后端完成仍通过现有 `(user_bid, scene_key, version)` 约束保持幂等；退役场景不需要迁移或删除记录。
- 包含 `admin_home_onboarding` 的旧 local-storage 值会被忽略，同时保留下来的课程编辑器回放状态仍可读。
- 课程编辑器目标缺失的步骤必须安全跳过，不能让保留流程走进死胡同。

## 共享权限后续

- 把这项作为独立的 PR2 之后迭代，而不是扩大所有者上线范围。
- 复用同一套 overlay / 目标解析原语，但存储单独的场景键，避免所有者完成和协作者完成互相干扰。
- 候选步骤集：
  - `prompt_edit`
  - `debug`
  - `preview`
- 从共享变体中排除偏所有者的动作：
  - `course_settings`
  - `publish`
- 对每个符合条件的共享协作者，在其第一次进入任意共享课程编辑器时触发一次，而不是每门课一次。
- 后续迭代的开放产品问题：如果共享协作者也有发布权限，决定这仍属于轻量协作者流程，还是应继续仅限所有者。

## 接口与依赖

- API：
  - `GET /api/user/onboarding/status`
  - `POST /api/user/onboarding/complete`
- 配置：
  - `ADMIN_ONBOARDING_ENABLED_FROM`
  - `DEMO_SHIFU_BID`
  - `DEMO_EN_SHIFU_BID`
- 追踪：
  - `creator_onboarding_started`
  - `creator_onboarding_step_viewed`
  - `creator_onboarding_completed`
  - 这些事件仅由保留下来的课程编辑器 onboarding 产出。
