# ai-shifu Skills 入口

本文是 [`SKILL.md`](SKILL.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 范围

- 本文件只保留仓库级 skill 路由和边界说明。
- 后端和前端各自维护自己的 skill 入口，因此本文件不应膨胀成一份混杂的故障排查手册。

## 放置规则

- 把 `ai-shifu/SKILL.md` 留给跨项目 skill 路由、所有权边界和迁移说明。
- 把 `src/api/SKILL.md` 留给后端项目级 skill 入口和聚焦 skill 索引。
- 把 `src/web/SKILL.md` 留给长期有效的 Cook Web 约束和聚焦 skill 索引。
- 把 `src/api/skills/xxx/SKILL.md` 或 `src/web/skills/xxx/SKILL.md` 留给带触发条件、工作流和回归清单的聚焦 skill。

## 入口

- 后端：`src/api/SKILL.md`
- 前端：`src/web/SKILL.md`
- 前端 skill 索引：`src/web/skills/README.md`
- 后端 skill 索引：`src/api/skills/README.md`

## 迁移说明

- 稳定规则应放在分层的 `AGENTS.md / CLAUDE.md` 中。
- 需要逐步执行或长期复用的故障排查知识，应放进对应子项目的 `SKILL.md` 体系。
