# 可配置的 Gemini Live 准入容量

本文是 [`gemini-live-configurable-capacity.md`](gemini-live-configurable-capacity.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 目的 / 全局图景
允许美国运营在不给运行中容器打补丁的情况下提高 Live 容量。其他环境保持当前默认值。

## 进度
- [x] 核验美国生产镜像、开关、硬编码上限，以及聚合 Redis 占用。
- [x] 新增五个正整数覆盖，带保守回退和 Redis 边界测试。
- [ ] 通过定向 PR 核验并合入 API 变更。
- [ ] 把美国取值持久化到部署配置，并滚动发布兼容的 API worker。
- [ ] 上线后核验每个美国 worker 以及 Redis 准入行为。

## 意外发现
美国生产运行两个启用轮换的 API 副本。共享上限是硬编码的；扩容 pod 无法提高它们。检查时，Redis 中有五个未过期凭据，分布在两个用户上，且没有活跃所有者。

## 决策记录
默认值保持为 96 个全局凭据、每用户 8 个、24 个活跃所有者、每用户每分钟 4 次 mint，以及全局每分钟 24 次 mint。美国覆盖分别设为 192、16、48、8 和 48。保留遗留非轮换凭据上限、15 分钟凭据寿命、记账恢复和原子 Lua 准入。不要清空 Redis 账本。

## 结果与回顾
实现进行中；在运行时核验完成前，不声称生产上限已变更。

## 上下文与定位
`live_follow_up_admission.py` 负责原子准入。环境定义位于中央配置注册表。美国部署清单维护在独立的 deploy-config 仓库的 k8s/us 下。

## 工作计划
把已校验的应用配置上限传入现有 Lua 脚本。暴露五个环境变量，并重新生成 Docker 示例。用隔离的真实 Redis 核验默认和自定义边界。只在兼容 API 镜像可用后，部署美国覆盖。

## 具体步骤
运行定向准入/配置测试、Ruff 和仓库检查；创建可合入 PR。通过单独的部署 PR 持久化配置。把已批准镜像/配置滚动发布到 ack-aishifu-us / ai-shifu-us，然后检查所有 API 副本。

## 校验与验收
现有默认值和记账生命周期测试通过。每个加倍后的边界在原边界处接受流量，并在新边界处以 retry_after_ms 拒绝。无效值保留默认值。运行时 worker 报告已配置取值；就绪状态保持健康。

## 幂等与恢复
配置可以重新应用。把覆盖回滚到默认值，且不删除凭据账本；现有预占会自然过期。混合版本滚动发布可能在旧 worker 退出前暂时保留更严格上限。

## 接口与依赖
新增正整数变量：GEMINI_LIVE_GLOBAL_CREDENTIAL_LIMIT、GEMINI_LIVE_USER_CREDENTIAL_LIMIT、GEMINI_LIVE_ACTIVE_SESSION_LIMIT、GEMINI_LIVE_USER_MINT_RATE_LIMIT、GEMINI_LIVE_GLOBAL_MINT_RATE_LIMIT。容量拒绝额外返回 capacity_scopes，这是一个有序数组，包含稳定机器枚举：global_credentials、worker_credentials、user_credentials、active_sessions、user_mint_rate、global_mint_rate 和 legacy_user_credential。每个同时阻塞的范围都会被返回，并在服务端记录，且不带用户标识符或 Redis 键。error_code 和最大 retry_after_ms 契约保持不变；其他结果省略该新字段。没有 Redis 键 schema 变更。

现有前端错误提示会用全部五种语言的共享翻译，渲染所有允许列表中的容量范围。缺失或未知范围保留通用容量错误。诊断会随现有重试生命周期清除，并被排除在 Umami payload 之外。
