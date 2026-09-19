# Cook Web AI 协作规则

本文是 [`AGENTS.md`](AGENTS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本文件把前端工作路由到正确的来源文档，并把硬性前端约束放在靠近 `src/web/` 的位置。

## 范围

- 本文件适用于 `src/web/`，包括 app 路由、组件、共享库、store 和前端测试。
- 使用 `../../ARCHITECTURE.md` 作为仓库地图，使用 `../../docs/engineering-baseline.md` 作为前端工程手册。
- 更具体的规则仍位于 `src/web/src/**/AGENTS.md`。

## 应当

- 在改变前端行为前，先检查路由、组件、hook、store 和共享 lib 路径。
- 把请求传输保持在 `src/web/src/lib/request.ts` 和 `src/web/src/lib/api.ts`，并保留统一的业务码处理路径。
- 把路由入口文件（`page.tsx`、`layout.tsx`、`route.ts`）视为可见的路由边界，把可复用逻辑移到组件、hook、store 或共享 lib 中。
- 让浏览器 harness 变更与 Playwright 冒烟套件以及本地 Docker 开发栈保持对齐。
- 按职责把共享的 learner 和 teacher 代码组织到 `api`、`assets`、`components`、`constants`、`hooks`、`lib`、`store` 和 `types` 下。把课程编排 helper 放在 `lib/shifu`，并在移动代码时保留现有的请求、状态和 UI 兼容契约。
- 每一个新的面向用户的 Cook Web 能力或交互路径，都必须在同一次变更中新增或扩展其 Umami 事件家族契约、producer 和定向测试。至少捕获一次有意义的功能曝光、被接受的使用，或一次有意义的结果；当指标需要合格浏览分母时，增加曝光事件；当一个信号无法准确描述异步工作流时，使用 attempt 加上终态结果事件。现有的通用 SPA pageview 不满足这一要求，除非路由进入本身就是文档化的功能采纳信号。没有这层覆盖，功能就不完整。纯视觉样式、仅文案、仅性能、仅测试，以及保持行为不变的重构，不需要新事件。任何新的用户可观察动作、状态转换或调用路径，包括为可访问性引入的路径，都需要。
- 对每一个新增或变更的 Umami 事件，遵循 `../../docs/references/frontend-product-analytics.md`。通过共享的 `useTracking` 和 `tracking` 路径发送业务事件；只有集中的 tracking 实现可以访问 `window.umami`、识别用户、排队调用或清洗提供方 payload。
- 把 SPA pageview 所有权保持在 `UmamiLoader`。从真实用户动作或带稳定输入的 post-commit effect 发出业务事件，绝不要在 render 期间发出；明确 guest 和 preview 策略，以及 per-render、per-open、per-session 或其他去重范围。
- 给新事件稳定的 `snake_case` 名称，通常是 `<actor>_<object>_<action-or-state>`。把稳定资源标识放进显式 payload 字段，而不是构造动态事件名；在没有协同迁移的情况下，不要重命名或改用一个已被消费的事件。
- 从显式允许列表的扁平标量字段构建 payload。在它们实际描述的转换点上报 attempt 以及终态 `success`、`failed` 或 `cancelled` 结果，并让 tracking 失败开放，使主用户动作永远不依赖分析投递。
- 把集中 identify 和 pageview 元数据的变更也视为隐私契约变更。只允许必要的假名身份和经过审查的 session 枚举，并从任何新增或变更的 pageview 处理中移除 query、fragment、凭据和敏感路径数据。
- 对于可点击 UI，优先使用语义元素（`button`、`a`、`summary`）或共享 Radix/shadcn 原语。如果非语义元素必须处理点击，用 `data-clickable="true"` 标记实际可点击目标，并用 `disabled`、`aria-disabled="true"` 或 `data-disabled` 保留禁用态。不要依赖页面局部 cursor 样式或宽泛的 `* { cursor: pointer; }` 规则。全屏 onboarding/backdrop 前进表面是例外：让它们的大背景或卡片命中区域保持默认 cursor，避免整页读起来像一个按钮。

## 避免

- 不要添加临时的组件 fetch 逻辑，或第二套请求抽象。
- 不要在 UI 组件中硬编码面向用户的字符串，或硬编码鉴权/请求头构造。
- 不要从功能代码调用 `window.umami` 或识别用户，不要添加第二套 pageview 路径，不要把 awaited tracking 调用当成投递确认，也不要用分析结果驱动产品状态。
- 不要把表单值、API 响应、配置对象、用户编写内容、完整 URL、query、referrer 或原始错误展开进 Umami 事件或身份 payload。体量清洗不是隐私清洗。
- 当共享模块已经拥有该行为时，不要再引入并行的 learner 和 teacher 源目录。
- 不要在 ExecPlan 之外新增复杂工作清单。

## 命令

- `cd src/web && npm run dev`
- `cd src/web && npm run type-check`
- `cd src/web && npm run lint`
- `cd src/web && npm run test:e2e`

## 测试

- 先为触及的领域运行定向 Jest 测试。
- 对每一个新增或变更的 Umami 事件，断言精确事件名和允许列表 payload、敏感字段缺失、触发时机、资格和去重、所有相关终态结果，以及 tracking 抛错或不可用时业务行为仍能继续。
- 当共享路由、hook、store 或请求行为变化时，运行 `npm run type-check` 和 `npm run lint`。
- 当浏览器 harness 代码或冒烟选择器变化时，运行 `npm run test:e2e`。

## 相关 Skills

- `src/web/SKILL.md`
- `src/web/skills/README.md`
