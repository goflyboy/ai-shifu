# 仓库脚本规则

本文是 [`AGENTS.md`](AGENTS.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本文件负责 `scripts/` 下共享维护脚本与生成脚本的工程规则，包括翻译工具和 AI 文档工具。

## 范围

- 本文件适用于 `scripts/`，包括 Python 与 JavaScript 生成器、校验脚本、翻译辅助工具，以及仓库维护工具。

- 脚本工程约定见 [工程基线](../docs/engineering-baseline.md)，尤其是：
  [测试期望](../docs/engineering-baseline.md#testing-expectations)、
  [CI/CD 与发布流程](../docs/engineering-baseline.md#cicd-and-release-workflow)、
  [环境配置](../docs/engineering-baseline.md#environment-configuration)，
  以及 [国际化规则](../docs/engineering-baseline.md#internationalization-rules)。

- 本目录负责可重复的仓库维护、翻译校验，以及本地开发和 CI 都会使用的 AI 文档生成或校验逻辑。

- 脚本改动可能重写已跟踪文件，或改变自动化行为，因此必须把所有权和副作用写清楚。

## 应当

- 在改变行为前，先检查脚本当前的调用点、声明的输出，以及同级的 checker 或 generator 脚本。

- 保持脚本输入、输出和文件所有权明确、使用仓库相对路径，并且可预测，使本地运行与 CI 任务行为一致。

- 维护脚本和生成脚本可能在 CI 中或由多名贡献者重复运行，优先保证幂等。

- 一旦生成产物、校验标记或预期文件清单发生变化，保持 generator 与 checker 成对对齐。

- 脚本面向的文本、注释和文档保持仅英文；面向用户的翻译仍应放在共享 i18n 数据中，而不是写进脚本。

## 避免

- 不要在脚本声明的所有权范围之外静默重写已跟踪文件。

- 不要把机器相关路径、密钥或仅本地有效的假设硬编码进共享脚本。

- 不要让校验脚本与它们本应验证的产物或不变量发生漂移。

- 当已有 generator、checker 或迁移工具已经负责该表面时，不要把不相关职责混进同一个脚本。

## 命令

- `python scripts/check_repo_harness.py` 在文档工具变更后，校验 AI 文档所有权、知识元数据，以及生成文件模型。
- `python scripts/check_architecture_boundaries.py` 校验已提交的前端/后端边界基线与 fixture 覆盖。

- `python scripts/generate_ai_collab_docs.py` 在 AI 指令或生成器变更后，重新生成派生的 AI 文档镜像。

- `python scripts/build_repo_knowledge_index.py` 重新生成知识索引和生成文档清单。

- `python scripts/check_translations.py && python scripts/check_translation_usage.py --fail-on-unused`
  是翻译工具变更后的共享翻译校验步骤。

- `python scripts/generate_languages.py` 在翻译清单行为变化时，刷新 locale 元数据。

## 测试

- 修改任何 generator 或维护脚本后，先运行最近的 checker。
- 修改边界 checker 或其 fixture 清单后，运行 `python scripts/check_architecture_boundaries.py --run-fixture-tests`。

- 当脚本会写入已跟踪文件时，重新运行该脚本并检查产出 diff，确认输出是确定性的。

- 当翻译脚本发生变化时，在同一任务中重新运行翻译对等校验、翻译使用校验，以及 locale 元数据检查。

- 当 AI 文档生成或校验脚本发生变化时，重新生成文档，再运行 `python scripts/check_repo_harness.py`，并对改动文件执行针对性的 lefthook 检查。

## 相关技能

- `SKILL.md` 是仓库级技能路由索引。

- 当共享脚本改动与后端或前端行为紧密耦合时，仍应分别从 `src/api/SKILL.md` 和 `src/web/SKILL.md` 进入。

- 把稳定的脚本规则留在这里；只有当同一套多步骤维护流程反复出现、值得单独沉淀时，才创建专项技能。