# api Skills

本文是 [`SKILL.md`](SKILL.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 分层

- 把长期有效的结构规则放在 `AGENTS.md` 和 `CLAUDE.md` 中。
- 只把可重复工作流、调试手册和迁移清单放进 `SKILL.md`，避免把目录级规则撑得过大。
- 后端模块的 `AGENTS.md` 文件可以指向这里，或指向 `src/api/skills/` 下的聚焦 skill。

## 后端 Skill 索引

- `skills/shifu-authoring-flow/SKILL.md`
- `skills/user-auth-flows/SKILL.md`

## 何时新增 Skill

- 当同一套 provider、鉴权、流式或持久化工作流在多个任务中重复出现时，再新增一份聚焦的后端 skill。
- 保持每个 skill 可执行：触发条件、核心规则、工作流和回归清单。
