# 可靠性

本文是 [`RELIABILITY.md`](RELIABILITY.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 目标

- 保持仓库知识是最新的，并且可被机械校验。
- 让请求级诊断对后端和浏览器冒烟失败仍然可用。
- 优先使用小而可重复的校验环，而不是大型手工 QA 清单。

## 当前可靠性环

- 生成的指令和知识索引必须是确定性的。
- `python scripts/check_repo_harness.py` 校验文档所有权、生成产物和元数据完整性。
- `python scripts/check_architecture_boundaries.py` 冻结当前边界债务，并在出现新的架构漂移时失败。
- `cd src/web && npm run test:e2e` 校验浏览器冒烟路径。
- 即使共享本地 `.env` 包含其他鉴权覆盖，默认开发 harness 也会固定手机登录，以保证冒烟确定性。
- Playwright 冒烟失败必须产出截图、控制台/网络摘要、trace，以及最终的 `X-Request-ID`。
- 当开发栈可用时，`cd src/api && python scripts/harness_diagnostics.py --request-id <id>` 把失败收窄到请求范围的后端证据和本地可观测性查询。
- 默认 API 启动会在重新运行 Alembic 之前修复已知的遗留迁移残留，使 Docker 开发 harness 能达到绿色稳态。

## 已知限制

- 本地可观测性栈只存在于默认 Docker 开发 harness；latest/release compose 文件有意保持不变。
- Playwright 冒烟覆盖有意保持狭窄，不应被视为完整回归覆盖。
- 某些流程仍依赖种子演示数据和默认 Docker 开发环境。
- 边界基线条目代表已知债务，预期会逐步收缩，而不是在一次大型重构中全部移除。
