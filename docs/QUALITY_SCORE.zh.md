# 质量评分

本文是 [`QUALITY_SCORE.md`](QUALITY_SCORE.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 目的

从面向代理的视角跟踪仓库质量，以便机械地排定清理和治理工作的优先级。

## 表面

### repo docs

- 当前等级：`B`
- 缺口：知识布局已经版本化并建立了索引，但长期健康仍依赖定期整理，以保持 `last_reviewed` 数据和退役术语清理是最新的。
- 下一步：保持生成的清单和 harness 健康报告为绿色，并让整理工作流持续收缩过期引用。

### api

- 当前等级：`B+`
- 缺口：第二阶段后端改造增加了黄金 SSE/JSON 回归 harness，把事务边界迁到共享 unit-of-work，并配有 commit-site 棘轮（2026-09 重新基线时，`dao/` 之外有 146 处被祖父化的 `db.session.commit()`，已从 213 处下降，并在分批计划 `docs/exec-plans/completed/uow-commit-site-migration.md` 中迁移到零）；把 /run 运行时拆成 emitter/recorder/state 协作者；删除了大约 800 行死代码，同时后端测试套件从 1,862 增长到 1,942。剩余债务是仍在事务内执行的 provider/LLM HTTP 调用、广泛存在的遗留 `Model.query` 风格，以及默认 Docker 开发栈仍依赖兼容性修复步骤，外加用于浏览器冒烟验证的在线可观测性服务。
- 下一步：把 commit-site 基线保持在零（棘轮现在会拒绝任何新的直接 commit），在重构过程中保持黄金 fixture 字节稳定，并把 `scripts/harness_diagnostics.py` 以及本地可观测性栈保留在标准冒烟失败工作流中。

### Web frontend

- 当前等级：`B`
- 缺口：路由和组件测试已经存在，但浏览器 harness 仍只覆盖最低限度的登录、后台和学习路径。共享 Umami 传输以及最高风险的 producer 家族现在已有隐私和回归覆盖。遗留的通用 visit 事件，以及已修复的发布、登录、支付、账务和课程创建家族中的错误前驱名称，是删除而不是双写；尚未处理的事件家族仍需要完整的版本化消费者契约。
- 下一步：在默认开发 harness 下保持 Playwright 冒烟套件为绿色，并且只有在当前三条路径保持稳定后再扩大覆盖。生产部署后验证新的规范系列及其消费者，盘点剩余 dashboard 消费者，并对所有新增或变更的 Cook Web Umami 事件强制执行 `docs/references/frontend-product-analytics.md`。

### runtime harness

- 当前等级：`B`
- 缺口：默认开发栈现在已包含日志、追踪、指标和浏览器冒烟管道，但持续质量标准仍取决于保持冒烟套件为绿色，并逐步偿还已提交的边界基线。
- 下一步：保持默认 Docker 开发栈健康，逐步减少基线条目，并且只有在当前路径保持稳定后再扩大冒烟覆盖。

### tests

- 当前等级：`B`
- 缺口：已有较强的定向测试，但跨表面验证仍依赖贡献者选择正确的命令。
- 下一步：让 lefthook 和仓库 harness 检查器继续作为文档与指令变更的权威入口。
