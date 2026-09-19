# Docker 工程规则

本文是 [`AGENTS.md`](AGENTS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本文件负责 `docker/` 下 Docker 表面的工程规则，包括 compose 文件、本地入口脚本、nginx 配置，以及面向部署的环境示例。

## 范围

- 本文件适用于 `docker/`，尤其是 `docker-compose*.yml`、`dev_in_docker.sh`、nginx 配置，以及 `docker/.env.example.full`。

- Docker 工程约定见 [工程基线](../docs/engineering-baseline.md)，尤其是：
  [架构](../docs/engineering-baseline.md#architecture)、
  [CI/CD 与发布流程](../docs/engineering-baseline.md#cicd-and-release-workflow)、
  [环境配置](../docs/engineering-baseline.md#environment-configuration)，
  以及 [故障排查](../docs/engineering-baseline.md#troubleshooting)。

- 本目录定义了三种不同的容器模式：挂载源码的本地开发、最新已发布镜像，以及固定版本发布部署。

- Docker 改动常常会同时影响后端和前端的启动假设，因此当服务接线或镜像语义发生变化时，应将其视为跨表面改动。

## 应当

- 在改变 Docker 行为前，一起检查受影响的 compose 文件、入口脚本，以及相关应用启动假设。

- 保持 `docker-compose.dev.yml`、`docker-compose.latest.yml` 和 `docker-compose.yml` 语义区分：本地挂载开发、最新已发布镜像，以及固定版本发布部署。

- 除非对应的发布或应用配置模型也发生变化，否则保持镜像名称、tag 语义、env-file 期望，以及服务启动顺序不变。

- 保持 `docker/.env.example.full` 与后端配置变更，以及仓库其他位置记录的环境变量指引对齐。

- 用 `docker compose ... config` 校验 compose 变更，并把面向发布的镜像行为与 GitHub 构建和发布工作流交叉核对。

## 避免

- 不要把密钥或环境相关凭据写入 compose 文件或 Docker 辅助脚本。

- 除非部署模型本身发生变化，否则不要把 dev、latest 和 pinned compose 角色合并进同一个文件。

- 不要在未检查对应 GitHub 发布和构建工作流的情况下，更改容器镜像 tag、服务名或启动语义。

- 不要在 Docker 中移动后端或前端启动假设，除非同时完成对应应用表面的更新和验证。

## 命令

- `cd docker && docker compose -f docker-compose.dev.yml config` 在编辑后校验本地开发 compose 文件。

- `cd docker && docker compose -f docker-compose.dev.yml up -d` 应为 Phase 2 harness 工作同时拉起默认应用和可观测性栈。

- 对于并行 worktree，在启动开发栈前设置稳定的 compose 项目名，并覆盖宿主机端口。项目名使用 `ai-shifu-<worktree-slug>` 模式；CI 使用 `ai-shifu-runtime-harness`。例如：
  `cd docker && docker compose -p ai-shifu-$(basename "$(git rev-parse --show-toplevel)" | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9_-' '-') -f docker-compose.dev.yml up -d`。
  用 `AI_SHIFU_WEB_PORT`、`AI_SHIFU_API_PORT`、`AI_SHIFU_MYSQL_PORT`、`AI_SHIFU_REDIS_PORT`、`AI_SHIFU_GRAFANA_PORT`、`AI_SHIFU_LOKI_PORT`、`AI_SHIFU_TEMPO_PORT`、`AI_SHIFU_TEMPO_OTLP_HTTP_PORT`、`AI_SHIFU_OTEL_GRPC_PORT` 和 `AI_SHIFU_PROMETHEUS_PORT` 覆盖端口。

- `cd docker && docker compose -f docker-compose.latest.yml config` 在编辑后校验最新已发布镜像 compose 文件。

- `cd docker && docker compose -f docker-compose.yml config` 在编辑后校验固定版本发布 compose 文件。

- `cd src/api && python scripts/generate_env_examples.py` 在后端配置变更会改变 Docker 环境期望后，刷新 `docker/.env.example.full`。

## 测试

- 对每个改动过的 compose 文件运行 `docker compose ... config`，并检查渲染输出中的服务名、env 文件和镜像引用。

- 当镜像或发布语义发生变化时，在结束任务前把 compose 引用与 GitHub 构建和发布工作流交叉核对。

- 当启动命令、entrypoint 或挂载发生变化时，检查受影响的后端和前端启动假设，并注明哪些运行时冒烟检查没有在本地执行。
- 当可观测性服务发生变化时，确认 Grafana、Loki、Tempo、Prometheus 和 OTEL collector 在开发栈内部仍然互相可达。

- 当只改 Docker 侧文档或 AI 指令时，运行 `python scripts/check_repo_harness.py`，并注明未启动容器。

## 相关技能

- `SKILL.md` 是仓库级技能路由索引。

- 当 Docker 改动与后端或前端行为紧密耦合时，仍应分别从 `src/api/SKILL.md` 和 `src/web/SKILL.md` 进入。

- 把稳定的 Docker 规则留在这里；只有当部署或发布手册变成反复出现的工作流时，才把它们移到专项技能中。