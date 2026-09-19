---
title: Agent-First Harness 迁移
status: implemented
owner_surface: repo
last_reviewed: 2026-04-17
canonical: false
---

# Agent-First Harness 迁移

本文是 [`agent-first-harness.md`](agent-first-harness.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 摘要

围绕 agent-first harness 重做仓库，使仓库知识、执行计划和校验环都明确、版本化，并被机械检查。

## 目标

- 引入作为系统事实来源的仓库知识布局。
- 用 `PLANS.md` 加上 ExecPlan 替换遗留的 `tasks.md` 工作流。
- 把根/后端/前端 AI 入口收缩为导航优先文档。
- 新增最低限度的浏览器加日志 harness，供代理用来校验修复。

## 约束

- 保持 HTTP API、数据库 schema 和共享 i18n 契约不变。
- 保持 `docs/engineering-baseline.md` 在当前路径。
- 迁移扁平文档，而不是保留重复副本。

## 交付物

- 根知识文档和生成索引
- 更新后的 AI 文档生成和校验脚本
- 带诊断产物的 Playwright 冒烟覆盖
- 后端 request-id 诊断脚本
