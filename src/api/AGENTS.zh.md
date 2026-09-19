# 后端 AI 协作规则

本文是 [`AGENTS.md`](AGENTS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本文件把后端工作路由到正确的来源文档，并把硬性后端约束放在靠近 `src/api/` 的位置。

## 范围

- 本文件适用于 `src/api/`，包括 `flaskr/`、`migrations/`、`tests/` 以及后端脚本。
- 使用 `../../ARCHITECTURE.md` 作为仓库地图，使用 `../../docs/engineering-baseline.md` 作为后端工程手册。
- 服务特定规则仍位于 `src/api/flaskr/service/<module>/AGENTS.md`。

## 应当

- 在改变后端行为前，先检查所属服务代码、DTO、helper 模块和 pytest 覆盖。
- 复用带 `code`、`message` 和 `data` 的共享响应信封、现有 provider 包装层，以及共享后端 helper 层。
- 让新增或变更的环境变量与 `src/api/flaskr/common/config.py` 保持对齐，并在配置变更时重新生成 Docker env 示例。
- 当任务复杂或跨切面时，把后端执行上下文保留在 ExecPlan 中。
- 在现有共享路径中使用 request-id 和 Langfuse helper，而不是发明并行诊断逻辑。
- 先在 SQLAlchemy 模型中定义目标 schema，再用 `FLASK_APP=app.py flask db migrate -m "message"` 生成 schema 修订，并在接受前审查候选迁移。
- 用 `flaskr/dao/uow.py` 中的 `with unit_of_work():` 拥有数据库事务：helper 只 add 和 flush，从不 commit 或 roll back；嵌套块加入调用方；外部副作用（通知、celery enqueue、缓存写入、后台线程）走 `uow.on_commit`。
- 用 `uow.app_context_scope(app)` 复用调用方的 app context；只在真正的入口（celery 任务、CLI 命令）推入 `app.app_context()`，因为嵌套 context 会切换 Flask-SQLAlchemy session，并破坏调用方的事务边界。

## 避免

- 不要编辑已应用的 Alembic 迁移。
- 不要用 SQLAlchemy 或 Flask-SQLAlchemy 的 `create_all()` 调用，或自定义 schema 自省守卫，来替代版本化 Alembic 迁移。只有在有文档记录的非事务 DDL 恢复需求时，才添加范围很窄的守卫。
- 不要为业务键关系添加硬外键约束，除非架构契约被有意改变。
- 不要绕过 LiteLLM 包装层或共享 provider helper 去做 OpenAI 兼容集成。
- 不要把后端翻译放进临时 Python 模块；使用 `src/i18n/` 下的共享 JSON namespace。
- 不要在 `flaskr/dao/` 之外添加 `db.session.commit()` 或 `db.session.rollback()`；`scripts/check_uow_commit_sites.py` 只允许 `docs/generated/uow-commit-baseline.json` 中的祖父化基线收缩。多步流程对每个持久化步骤使用一个 unit of work，并且永远不要跨越 generator `yield` 或 provider HTTP 调用。

## 命令

- `cd src/api && FLASK_APP=app.py flask run`
- `cd src/api && pytest -q`
- `cd src/api && FLASK_APP=app.py flask db migrate -m "message"`
- `cd src/api && python scripts/harness_diagnostics.py --request-id <id>`
- `python scripts/check_uow_commit_sites.py`（仓库根目录）验证没有出现新的直接 commit 点；在迁移移除站点后，用 `--update` 把基线向下棘轮。

## 测试

- 当服务行为变化时，在 `src/api/tests/service/` 中新增或更新定向 pytest 覆盖。
- 每次迁移到 `unit_of_work()` 时，都配一个中途失败测试（模式：`src/api/tests/service/order/test_uow_failure_paths.py`），证明部分写入会回滚，且 `on_commit` 副作用不会触发。
- 在接受 schema 变更前，手工审查生成的迁移文件。
- 当请求流、provider 集成或共享配置行为变化时，运行更广的后端测试。

## 相关 Skills

- `src/api/SKILL.md`
- `src/api/skills/shifu-authoring-flow/SKILL.md`
- `src/api/skills/user-auth-flows/SKILL.md`
