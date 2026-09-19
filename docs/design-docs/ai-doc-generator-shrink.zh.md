---
title: AI 文档生成器收缩与基线恢复
status: implemented
owner_surface: repo
last_reviewed: 2026-04-17
canonical: false
---

# AI 文档生成器收缩与基线恢复

本文是 [`ai-doc-generator-shrink.md`](ai-doc-generator-shrink.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 摘要

这次变更把仓库从完全生成的 AI 指令层，改成混合模型：

- 手工维护的核心 AI 文档，覆盖仓库、后端和前端范围
- `docs/` 下的一份权威工程基线文档
- 只生成薄包装和重复兼容表面

目标是恢复拆分 AI 文档结构引入时被压缩掉的工程约定，同时不回到单份巨大的根 `AGENTS.md`。

## 问题

- 生成器当前拥有根、后端和前端 `AGENTS.md` 文件，这把稳定工程指引推进了过度简化的模板。
- 旧根 `AGENTS.md` 中的一些高价值约定，已不再能从最近范围清晰获得。
- 当前所有权模型让人太容易在调整 AI 协作模板时丢失工程基线内容。

## 决策

### 核心文档所有权

- `AGENTS.md`、`src/api/AGENTS.md` 和 `src/web/AGENTS.md` 变成手工维护文件。
- 这些文件保持简洁，聚焦 AI 协作行为、本地执行规则，以及指向共享工程基线的链接。
- 它们继续使用标准的 `Scope`、`Do`、`Avoid`、`Commands`、`Tests` 和 `Related Skills` 标题，使继承保持可预测。

### 工程基线

- 新增 `docs/engineering-baseline.md` 作为权威工程基线。
- 恢复旧根 `AGENTS.md` 中接近完整的规范性指引，包括架构说明、数据库约定、API 响应规范、测试结构、工作流预期、性能指引、环境工作流、i18n 规则、命名规则和故障排查命令。
- 保持这份文档手工维护。它是稳定的工程参考，不是生成衍生品。

### 生成表面

- 继续生成：
  - 根和嵌套 `CLAUDE.md` 包装
  - 模块级后端服务 `AGENTS.md`
  - 模块级前端领域 `AGENTS.md`
  - `.cursor/rules/**` 下的 Cursor 规则
  - `.github/**` 下的 Copilot 指令文件
- 停止生成：
  - `AGENTS.md`
  - `src/api/AGENTS.md`
  - `src/web/AGENTS.md`
  - `.claude/rules/**`

### 校验模型

- 校验器必须区分生成文档和手工维护文档。
- 生成文档仍要求生成标记、行数检查和结构检查。
- 手工维护的核心文档必须存在，包含期望的锚点标题，并在适当时链接到 `docs/engineering-baseline.md`。
- `docs/README.md` 应继续对扁平设计文档加根 `tasks.md` 工作流有效，同时允许像 `docs/engineering-baseline.md` 这样的长期文档。

## 兼容性

- 没有运行时产品行为变化
- 没有数据库或 API 契约变化
- 不改变外部 AI 指令表面名称

代理仍会与以下内容交互：

- `AGENTS.md` 作为共享指令来源
- `CLAUDE.md` 作为 Claude 包装
- `.claude/rules/**` 用于仅 Claude 的路由
- `.cursor/rules/**` 用于 Cursor 兼容
- `.github/**` 用于 Copilot 兼容

## 校验

- `python scripts/check_ai_collab_docs.py`
- `python scripts/generate_ai_collab_docs.py`
- `python scripts/check_ai_collab_docs.py`
- `pre-commit run --files AGENTS.md src/api/AGENTS.md src/web/AGENTS.md docs/engineering-baseline.md docs/README.md scripts/generate_ai_collab_docs.py scripts/check_ai_collab_docs.py`
