---
name: user-auth-flows
description: 在修改后端用户鉴权、验证码、token 持久化、临时用户行为或鉴权提供方集成时使用。保持提供方分发和凭据状态集中。
---

# 用户鉴权流程

本文是 [`SKILL.md`](SKILL.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 核心规则

- 把 repository、token-store 和 auth-factory 路径作为事实来源。
- 保持验证码消费语义和重试安全。
- 面向前端的鉴权 payload，要与后端持久化逻辑一起更新。

## 工作流

1. 从 `repository.py`、`token_store.py`、`auth/factory.py`，以及具体流程文件开始，例如 `email_flow.py` 或 `phone_flow.py`。
2. 确认这次变更影响的是凭据、token、临时用户，还是提供方分发。
3. 校验和持久化一起更新；避免只改路由或只改提供方代码。
4. 在 `src/api/tests/service/user/` 下新增或更新测试。

## 回归检查清单

- 登录或验证的成功路径。
- 重试或重复提交路径。
- 无效或过期验证码。
- 受影响时的 token 持久化和登出语义。
