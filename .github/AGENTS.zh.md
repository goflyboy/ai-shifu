# GitHub 自动化规则

本文是 [`AGENTS.md`](AGENTS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本文件负责 `.github/` 下仓库自动化的工程规则，包括 workflow YAML、发布自动化、issue 模板，以及 GitHub 侧 AI 兼容指令文件。

## 范围

- 本文件适用于 `.github/`，尤其是 `.github/workflows/`、`.github/instructions/` 和 `.github/copilot-instructions.md`。

- GitHub 工程约定见 [工程基线](../docs/engineering-baseline.md)，尤其是：
  [CI/CD 与发布流程](../docs/engineering-baseline.md#cicd-and-release-workflow)、
  [开发工作流](../docs/engineering-baseline.md#development-workflow)、
  [环境配置](../docs/engineering-baseline.md#environment-configuration)，
  以及 [故障排查](../docs/engineering-baseline.md#troubleshooting)。

- 本目录拥有仓库测试自动化、release-draft 行为，以及会同时影响后端、前端和 Docker 表面的镜像构建与发布自动化。

- `.github/` 下的指令兼容文件是主手册文档的镜像，必须与它们保持对齐。

## 应当

- 在改变自动化行为前，先检查受影响的 workflow、当前触发器、路径过滤，以及相关下游 job。

- 保留 workflow 触发意图，并让 branch、tag 和 path filter 保持当前发布或校验模型所需的狭窄范围。

- 把密钥、token、registry 凭据和可配置开关放在 GitHub Actions secrets 或 vars 中，而不是硬编码进 YAML。

- 让 `prepare-release.yml`、`build-latest.yml` 和 `build-on-release.yml` 与仓库其他地方实际使用的镜像名、tag 语义和 Docker 期望保持对齐。

- 当变更 `.github/instructions/` 或 `.github/copilot-instructions.md` 时，在同一次变更中更新对应的手册文档和生成镜像。

## 避免

- 不要随意扩大 workflow 触发范围，或随意移除 path filter。

- 不要在 workflow 文件中内联密钥、token 或环境相关 registry 值。

- 不要只在一个 workflow 中改变发布版本或镜像 tag 行为，而不一起检查相关的发布和构建 workflow。

- 当最近的手册 `AGENTS.md` 和 `CLAUDE.md` 另有说明时，不要把 `.github/instructions/` 或 `copilot-instructions.md` 当作事实来源。

## 命令

- `find .github/workflows -maxdepth 1 -type f | sort` 在你改变触发范围或发布行为前列出仓库 workflow 集合。

- `git diff -- .github/workflows .github/instructions .github/copilot-instructions.md` 同时展示受影响的自动化和兼容表面。

- `lefthook run pre-commit` 是 workflow 和 GitHub 侧指令变更的聚焦卫生检查（对暂存文件运行 hooks）。

- 当 `.github/` 指令镜像或手册 AI 文档入口变化时，必须运行 `python scripts/check_repo_harness.py`。

- 当 workflow 的 path filter 或 repo-harness 覆盖变化影响到源所有权时，必须运行 `python scripts/check_architecture_boundaries.py`。

## 测试

- 每当 workflow 逻辑变化时，至少手工审查一个受影响 workflow 的触发范围、密钥使用和下游 job 假设。

- 当发布或镜像构建自动化变化时，对照手册中的 Docker 镜像名、tag 和发布期望交叉检查 workflow 行为。

- 当只有 GitHub 侧 AI 指令文件变化时，运行 `python scripts/check_repo_harness.py`，并注明运行时自动化并未执行。

- 当 workflow 变更影响后端、前端、Docker 或 scripts 路径时，验证 path filter 和变更文件假设仍匹配预期自动化表面。
- 让 `repo-harness.yml`、`runtime-harness.yml` 和 `harness-gardening.yml` 与它们实际要约束的 harness 资产保持对齐。

## 相关 Skills

- `SKILL.md` 是仓库级 skill 路由索引。

- 当 workflow 变更与后端或前端行为紧密耦合时，`src/api/SKILL.md` 和 `src/web/SKILL.md` 仍是正确入口。

- 把长期有效的 GitHub 自动化规则留在这里；只有当同一套 workflow 反复出现时，才把多步调试或发布 runbook 放进聚焦 skill。
