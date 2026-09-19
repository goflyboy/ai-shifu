# AI 协作规则

本文是 [`AGENTS.md`](AGENTS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本根文件是编码代理的仓库入口。从这里开始，再进入最近的子树 `AGENTS.md`，以及它指向的知识文档。

## 范围

- 除非更深层的 `AGENTS.md` 收窄了规则，否则本文件适用于整个仓库。
- 把 `ARCHITECTURE.md`、`PLANS.md` 和 `docs/engineering-baseline.md` 视为本入口背后的主要来源文档。
- 使用 `docs/QUALITY_SCORE.md`、`docs/RELIABILITY.md` 和 `docs/SECURITY.md` 了解仓库范围的质量缺口和 harness 约束。

## 应当

- 编辑前先阅读最近的 `AGENTS.md`；当工作跨越多个表面时，用 `ARCHITECTURE.md` 给自己定位。
- 在改变行为前，先检查当前实现、相邻调用点，以及附近的测试或文档。
- 在创建新抽象之前，优先复用现有模块、DTO、store、provider wrapper 和请求路径。
- 复杂工作使用 ExecPlan。`PLANS.md` 定义格式，进行中的计划位于 `docs/exec-plans/active/`。
- 每当为本仓库创建 Git worktree 时，在启动服务前，把源 checkout 中已有的本地 `.env` 文件复制到新 worktree 的对应路径，包括仓库根目录的 `.env`，以及存在时的 `src/web/.env`。保留文件权限，永远不要提交这些副本，也不要覆盖新 worktree 中已经定制过的环境文件。
- 提交前运行 `python scripts/check_dev_tools.py`，确认 lefthook 及其底层工具已安装；如果从未安装 lefthook，本地检查会被静默跳过。
- 把 Ruff 发现视为代码或契约信号。优先使用已有项目模式，并补上针对性回归覆盖；如果被标记的写法是有意为之，使用最窄的代码级抑制，并给出浅白英文原因。只有在由进行中的 Ruff ExecPlan 跟踪的专项规则 PR 中，才改动全局 Ruff selection 或 ignore。
- 关于 git commit 的标题、正文和分类要求，使用 [Git 提交信息要求](#git-提交信息要求)；让代理专用规则文件指向这里，而不是重复这份策略。
- 当分支已经有打开的 PR 时，保持 PR 标题和描述与最新代码变更同步，使其准确描述当前实现和验证状态。
- 让每个 pull request 聚焦一个定义清晰的问题。把解决该问题所需的全部代码、测试、文档、迁移和兼容性工作包含进去，但把无关修复、清理和后续工作移到单独的 pull request。
- 代码评审时，只评估该 pull request 是否正确、安全、完整地解决了其声明的问题，并且测试是否充分，包括该变更引入的回归或契约影响。
- 保持共享指令表面对齐。当共享规则移动时，在同一次变更中更新被触及的 `AGENTS.md`、`CLAUDE.md`、生成的 `.cursor` 规则，以及生成的 `.github` 指令。
- 把产品分析视为每一条新的面向用户的 Cook Web 能力或交互路径的完成要求。在同一次变更中，按 `docs/references/frontend-product-analytics.md` 定义或扩展一个与决策相关的 Umami 事件族，实现其生产者，并补上针对性回归覆盖；缺少该分析契约时，功能不算完成。纯视觉样式、仅文案、仅性能、仅测试，以及保持行为不变的重构，不需要新事件。任何新的用户可观察动作、状态转换或调用路径，包括为无障碍引入的路径，都需要；已被分析覆盖的行为变更必须更新其现有契约。
- 把新增或变更的 Cook Web Umami 事件视为版本化的产品分析契约。实现前，按 `docs/references/frontend-product-analytics.md` 定义决策或指标、精确触发条件、适用人群和排除项、去重范围、稳定 payload schema，以及下游消费者。
- 保持产品分析尽力而为，并独立于用户可见操作。当事件名称、含义、适用规则、去重规则或 payload 契约发生变化时，一并更新其生产者、消费者、权威产品文档、兼容性计划和回归测试。
- 面向代码的文本保持英文；面向用户的文本放在 `src/i18n/` 下的共享 i18n JSON 中。
- 所有时间戳都以 UTC 存储和计算。在后端，任何写入数据库的时间都使用 `src/api/flaskr/util/datetime.py` 中的共享 `now_utc()` helper，并把新模型时间戳列默认设为 `default=now_utc` / `onupdate=now_utc`。DB session 在 `src/api/flaskr/dao/__init__.py` 中被钉在 UTC；把它当作安全网，而不是可以写入本地时间的许可。
- 在读取侧，把时间戳序列化为带尾部 `Z` 的 UTC ISO-8601。优先让 DTO 的 datetime 字段保持原始 `datetime | None`，由 `src/api/flaskr/route/common.py` 中唯一的序列化出口 `fmt()` 发出（它把 naive 值当作 UTC，并把 aware 值转换成 UTC）。必须手工序列化时，使用 `src/api/flaskr/util/datetime.py` 中的 `to_utc_iso()`；缺失时间返回 `null`，而不是 `""` 或预先格式化的字符串。展示时区转换是纯前端职责：浏览器通过 `formatAdminUtcDateTime` 这类 helper 渲染 UTC，因此 API 不得按请求时区做本地化。
- 在中文面向用户的文本和中文文档中，不要把 `创作者` 当作通用产品词。通用的课程搭建或老师账号角色使用 `老师`；指某个具体课程所有者时，使用 `课程负责人`，或在已有课程上下文中使用 `负责人`。
- 让英文和法文翻译与该区分保持一致：通用角色使用 `teacher` / `enseignant`，而具体课程所有者仍可翻译为 `creator` / `créateur`。

## 避免

- 不要只依赖聊天上下文做本应从已版本化文件中发现的仓库决策。
- 当尚未检查本地实现和相邻测试时，不要靠猜测开始改代码。
- 不要对当前 pull request 责任边界之外的无关或既有问题提出评审意见。
- 不要硬编码面向用户的字符串、密钥或环境相关 URL。
- 不要把自由文本或用户撰写内容、提示词或模型输出、姓名、联系方式、资料内容、标题或描述、优惠码、token、原始错误，或完整 URL、查询串、referrer 加入新增或变更的 Umami 事件或身份元数据。只使用必要的稳定机器 ID、布尔值、数字、时长和低基数枚举的显式允许列表；截断或哈希并不能让敏感字段可以安全采集。新增或变更的 pageview 处理必须在跟踪前剥离查询串、fragment、凭据和敏感路径数据。
- 不要把尽力而为的 Umami 数据当作计费、权限、审计或其他正确性敏感业务决策的权威来源，也不要在分析不可用时阻塞或改变用户操作。
- 不要创建新的根级 `tasks.md` 清单。复杂执行现在属于 `docs/exec-plans/` 下的 ExecPlan。
- 不要让共享指引与生成镜像或当前仓库结构发生漂移。
- 不要引入混用时区的时间戳：避免用 `func.now()` / `CURRENT_TIMESTAMP` 默认值，以及裸的 `datetime.now()` / `datetime.utcnow()` 作为存储时间。它们依赖 DB session 或进程时区，会再次引入 UTC 与本地时间漂移；改用 `now_utc()`。
- 不要绕过读取侧 UTC 契约：避免用 naive 的 `datetime.isoformat()` / `strftime(...)`（没有 `Z`）发出 API datetime；不要按请求/浏览器的 `?timezone=` 参数在服务端本地化展示时间；也不要比较携带不同序列化契约的时间戳（naive 对比 offset-aware，或字符串对比字符串）——比较前先把两边都规范化到 UTC。当新代码落到 UTC 清扫尚未覆盖的模块时，正是这些漂移会再次出现。
- 不要为了让新代码通过，就削弱 `ruff.toml`、添加一揽子 `noqa`，或创建宽泛的按文件例外。先修代码；把全局 ignore 留给已记录的仓库范围契约冲突。

## Git 提交信息要求

所有 git 提交信息要求都放在本节。其他文档和代理专用规则文件可以指向这里获取标题、正文和分类规则，但不得重复或重新定义它们。

- 人工撰写和编码代理撰写的提交信息都必须遵循下面的策略。现有工作流生成的 bot 提交可以豁免，除非该工作流正在为这项策略更新。
- 本地 `commit-msg` hook 只是基线 Conventional Commits 语法检查。它不强制下面的 `Changed:` / `Benefit:` 正文，也不强制分类规则。
- `Changed:` / `Benefit:` 正文要求只适用于提交信息；PR 描述不受此约束。
- 标题：使用不带 scope 括号的英文 Conventional Commits，例如 `type: summary`；不要使用 `type(scope): summary`。用产品用户能理解的浅白语言写摘要。当变更影响用户时，描述用户可见结果或收益，而不是只点出内部实现细节（例如用 `fix: prevent audio overlapping`，而不是 `fix: update useExclusiveAudio state`）。
- 正文：恰好包含两个小节，`Changed:` 和 `Benefit:`。
- 分类：像本文件这样仅做仓库维护的指令或生成指引更新，使用 `chore`。
- 运行时 prompt、模板和 system-prompt 变更会影响产品行为：增加能力时使用 `feat`，修正行为时使用 `fix`；不要使用 `docs`。

示例：

```text
chore: centralize commit message requirements

Changed:
Moved repository commit message requirements into the root AGENTS.md file.

Benefit:
Contributors have one place to check the required commit title and body format.
```

## 命令

- `python scripts/generate_ai_collab_docs.py` 重新生成兼容指令表面。
- `python scripts/build_repo_knowledge_index.py` 重新生成仓库知识索引和文档清单。
- `python scripts/check_repo_harness.py` 校验 AI 文档所有权、知识元数据，以及生成的 harness 产物。
- `python scripts/check_architecture_boundaries.py` 校验已提交的前端/后端边界基线，并阻止新的漂移。
- `python scripts/check_uow_commit_sites.py` 强制后端 unit-of-work 棘轮：禁止在 `flaskr/dao/` 之外新增 `db.session.commit()`，并且 `--update` 会在迁移移除站点后收缩基线。
- `python scripts/check_dev_tools.py` 验证 lefthook 及其底层工具已安装，避免 pre-commit hook 被静默跳过。
- `lefthook run pre-commit --all-files` 是提交级变更落地前的仓库范围验证门禁。

## MarkdownFlow 组件库

ai-shifu 使用两个 MarkdownFlow 组件库。改动其中任何一个通常是为了在本项目中使用，因此整体流程是：**改库 → 发布构建 → 让 ai-shifu 指向它 → 在本地 / 测试 / 生产调试**。

| 库 | 类型 | 钉在 | 发布来源 |
| ------------------ | ---------------- | ---------------------------------------------------- | ------------------------------------------------------------------------- |
| `markdown-flow` | Python（后端） | `src/api/requirements.txt` (`markdown-flow==<ver>`) | [markdown-flow-agent-py](https://github.com/ai-shifu/markdown-flow-agent-py) (PyPI) |
| `markdown-flow-ui` | npm（前端） | `src/web/package.json` (`"markdown-flow-ui"`) | [markdown-flow-ui](https://github.com/ai-shifu/markdown-flow-ui) (npm) |

### 针对本项目试用一次库改动

1. 在库仓库中，在你的分支上运行其 **Publish** action 发布一次构建——一次性测试包使用 `dev` 构建（PyPI 上是 `X.Y.Z.devN`，npm 上是 `X.Y.Z-dev.N`）。等待运行通过；此时版本已经发布。
2. 让本项目指向该版本：
   - **本地（未提交）**：编辑 `src/api/requirements.txt`（后端）或 `src/web/package.json`（前端）中的钉版本。
   - **已经推送的功能分支**：运行对应的 bump action——**Bump markdown-flow**（`bump-markdown-flow.yml`）或 **Bump markdown-flow-ui**（`bump-markdown-flow-ui.yml`）。它们会校验版本已发布，更新钉版本，并把 bump 推回该分支。两者都拒绝在 `main` 上运行。（前端 bump 会钉精确版本，并刷新 `src/web/package-lock.json`，以便 CI 的 `npm ci` 保持通过。）

### 规则：`main` 必须钉这两个库的 RELEASE 版本

dev 构建只用于功能分支 / 跨仓库测试。`main` 必须始终钉这两个库的 **release** 版本（`X.Y.Z`）。`repo-harness.yml` 中的 **Static Checks** job 会在每个进入 `main` 的 PR 上强制这一点，并在遇到预发布/dev 钉版本时失败：

- `markdown-flow` 的 release 钉版本在 `src/api/requirements.txt` 中检查。
- `markdown-flow-ui` 的 release 钉版本在 `src/web/package.json` 中检查。

合并前先把两者都钉到 release 版本。

## 内嵌的 MarkdownFlow 2.0 引擎

`src/api/flaskr/service/learn/agent/engine/` 保存 MarkdownFlow 2.0 引擎。与上面两个组件库不同，它**不是**钉版本依赖：代码被复制进来，并在这里原地编辑。`ai-shifu/markdown-flow-agent` 仍然存在，用于 playground 和开源发布，但它不是上游——两边不会来回同步，并且预期会分叉。

因为引擎不再有自己仓库的门禁，它随附的离线测试现在就是门禁：

```bash
cd src/api && python -m pytest tests/service/learn/agent/engine/ -q
```

这些测试运行在 `pydantic-ai` 的 `FunctionModel` 上，因此不需要网络，也没有成本。**`engine/` 下的任何改动都必须让它们保持绿色。** 它们覆盖周围服务测试覆盖不到的内容：在一次交互上暂停和恢复、一轮中抛出多次交互、答案在 store 往返后仍然存活、变量到达 memory、listen-mode 分段、1.0 语法检测，以及 confirm 护栏。

## 测试

- 先运行最小相关的后端、前端或脚本检查，只有当变更跨越共享契约或多个表面时，再扩大范围。
- 当任务只触及文档或指令文件时，至少运行 `python scripts/check_repo_harness.py`。
- 当任务新增或变更前端 Umami 事件时，测试精确名称和 payload，负向断言敏感字段不存在，覆盖触发时机、适用性、去重和终态结果，并证明跟踪失败不会改变用户可见结果。
- 当由 Ruff 驱动的改写改变了运行时行为或公共/内部契约时，新增或确认一条针对性回归测试；干净的 lint 结果不是行为覆盖。
- 当任务改变共享边界或引入新的 app/service 依赖时，运行 `python scripts/check_architecture_boundaries.py`。
- 当任务触及浏览器 harness 时，运行 `cd src/web && npm run test:e2e`。

## 相关技能

- `SKILL.md` 是仓库级技能路由索引。
- `src/api/SKILL.md` 负责后端工作流技能。
- `src/web/SKILL.md` 负责前端工作流技能。