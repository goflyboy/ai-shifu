# 工程基线

本文是 [`engineering-baseline.md`](engineering-baseline.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

本文是仓库的权威工程基线。仓库范围的架构说明、API 规范、数据库约定、测试期望、工作流规则、命名规则和故障排查指引都放在这里。分层的 `AGENTS.md` 文件仍是硬规则入口，本手册承载这些规则背后展开的理由、示例和故障排查细节。

## 快速开始

### 最常见任务

| 任务 | 命令 | 位置 |
|------|---------|----------|
| 启动后端开发服务器 | `flask run` | `cd src/api` |
| 启动 Cook Web（前端和 CMS） | `npm run dev` | `cd src/web` |
| 运行后端测试 | `pytest` | `cd src/api` |
| 运行前端单元测试 | `npm run test:ci` | `cd src/web` |
| 生成数据库迁移 | `FLASK_APP=app.py flask db migrate -m "message"` | `cd src/api` |
| 应用数据库迁移 | `FLASK_APP=app.py flask db upgrade` | `cd src/api` |
| 检查代码质量 | `lefthook run pre-commit --all-files` | 仓库根目录 |
| 启动全部服务（Docker） | `docker compose -f docker-compose.latest.yml up -d` | `cd docker` |
| 启动 Docker 开发栈（构建本地 latest） | `./dev_in_docker.sh` | `cd docker` |
| 构建 Cook Web 开发镜像 | `docker build ../src/web -t ai-shifu-cook-web-dev -f ../src/web/Dockerfile_DEV` | `cd docker` |

### 关键环境变量

```bash
# Backend (src/api/.env)
FLASK_APP=app.py

# Cook Web (src/web/.env.local)
NEXT_PUBLIC_API_URL=http://localhost:5000
```

### 本地工具安装

代码质量检查通过 **lefthook** 运行（一个会调用你机器上已安装工具的 Go 二进制）。只有在 `lefthook install` 把 hook 接入 `.git/hooks` 之后，git hook 才会触发；每个 hook 都会调用必须已经在 `PATH` 上的工具。一次性安装：

按你的平台安装 lefthook。

macOS（Homebrew）：

```bash
brew install lefthook
```

Linux 或 Windows（npm）：

```bash
npm install -g @evilmartians/lefthook
```

然后安装其余开发工具：

```bash
pip install ruff==0.16.5 commitizen==4.16.2 pre-commit-hooks==6.0.0
(cd src/web && npm ci)   # provides prettier + eslint
lefthook install
```

> **重要：** 如果你跳过 `lefthook install`（或从未安装 lefthook），pre-commit 检查会在提交时被**静默跳过**——不会有任何警告，缺口只会在之后的 CI 中暴露。每个 clone 都要运行一次 `lefthook install`。

随时（以及提交前）用 doctor 校验环境，它会准确报告缺了什么以及如何安装：

```bash
python scripts/check_dev_tools.py            # core gaps fail; frontend gaps warn
python scripts/check_dev_tools.py --strict   # also fail on Cook Web tooling gaps
```

## 关键要求

### 每次提交前必须做

1. 确认工具链已安装：`python scripts/check_dev_tools.py`
2. 运行 lefthook 检查：`lefthook run pre-commit`
3. 为数据库变更生成迁移：`flask db migrate -m "description"`
4. 测试相关变更表面
5. 面向代码的文本使用英文
6. 人工撰写和编码代理撰写的提交，遵循 [`AGENTS.md`](../AGENTS.md#git-commit-message-requirements) 中的 git 提交信息要求

### 常见陷阱

- 永远不要编辑已应用的迁移。始终创建新迁移。
- 不要硬编码面向用户的字符串。使用 i18n key。
- 不要为业务键关系创建数据库外键约束。
- 不要跳过 lefthook 检查。
- 不要提交密钥。
- 不要在代码或面向代码的文档中使用中文。

## 仓库概览

AI-Shifu 是一个由 AI 主导的聊天平台，在教育、叙事、产品指南和问卷中提供交互式个性化对话。与传统由人主导的聊天机器人不同，AI-Shifu 采用 AI 主导的对话流：用户可以提问和交互，但 AI 保持对叙事推进的控制。

## 架构

项目采用微服务架构，包含两个主要组件：

- 后端 API（`src/api/`）：基于 Flask 的 Python API，使用 SQLAlchemy ORM
- Cook Web（`src/web/`）：基于 Next.js 的统一前端和内容管理界面

### 后端架构说明

- 使用 Flask、SQLAlchemy 和 MySQL 构建
- 插件式架构，热重载支持位于 `flaskr/framework/plugin/`
- 服务层按领域组织，例如 `shifu`、`learn`、`user`、`order`、`profile`、`lesson` 和 `llm`
- 数据库迁移用 Alembic 管理，位于 `migrations/`
- 共享本地化数据位于 `src/i18n/`

#### LLM 集成

- 所有服务端 LLM 调用都通过 `src/api/flaskr/api/llm/__init__.py` 中的 LiteLLM 路由
- 供应商凭据继续通过现有 API-key 变量放在 `.env` 中
- 优先使用 OpenAI 兼容供应商，以便共享 LiteLLM wrapper 拥有集成

### 前端架构说明

- Cook Web 使用 Next.js、TypeScript 和 Tailwind CSS
- 前端同时提供学员路由和编写/管理工具
- 共享请求处理位于 `src/web/src/lib/request.ts` 和 `src/web/src/lib/api.ts`
- 学员和老师流程共用 `api`、`assets`、`components`、`constants`、`hooks`、`lib`、`store` 和 `types`；课程编排位于 `lib/shifu`

#### 统一请求系统

Cook Web 前端在 `/main` 和 `/c` 这类路由上使用同一套请求系统。

请求流程：

1. 业务层调用 API 函数
2. API 层构建请求，并委托给 request client
3. Request client 注入鉴权头并执行 HTTP 请求
4. 业务码处理检查 `response.code`
5. 业务层接收 `response.data`

把请求传输、业务码处理和鉴权错误处理留在这套共享栈中，而不是在功能代码里重新实现。

## 数据库模型约定

使用一致的 SQLAlchemy 模型顺序和字段语义。

### 完整模型示例

```python
from sqlalchemy import Column, BIGINT, String, SmallInteger, DateTime, func
from flaskr import db


class Order(db.Model):
    __tablename__ = "order_orders"
    __table_args__ = {"comment": "Order entities"}

    id = Column(BIGINT, primary_key=True, autoincrement=True)

    order_bid = Column(
        String(32),
        nullable=False,
        default="",
        index=True,
        comment="Order business identifier",
    )

    user_bid = Column(
        String(32),
        nullable=False,
        default="",
        index=True,
        comment="User business identifier",
    )

    amount = Column(
        BIGINT,
        nullable=False,
        default=0,
        comment="Order amount in cents",
    )

    status = Column(
        SmallInteger,
        nullable=False,
        default=0,
        comment="Status: 0=pending, 1=paid, 2=cancelled",
    )

    deleted = Column(
        SmallInteger,
        nullable=False,
        default=0,
        index=True,
        comment="Deletion flag: 0=active, 1=deleted",
    )

    created_at = Column(
        DateTime,
        nullable=False,
        default=func.now(),
        server_default=func.now(),
        comment="Creation timestamp",
    )

    created_user_bid = Column(
        String(32),
        nullable=False,
        index=True,
        default="",
        comment="Creator user business identifier",
    )

    updated_at = Column(
        DateTime,
        nullable=False,
        default=func.now(),
        server_default=func.now(),
        onupdate=func.now(),
        comment="Last update timestamp",
    )

    updated_user_bid = Column(
        String(32),
        nullable=False,
        index=True,
        default="",
        comment="Last updater user business identifier",
    )
```

### 数据库变更清单

- [ ] 在 `src/api/flaskr/service/[module]/models.py` 中完成模型变更
- [ ] 用 `FLASK_APP=app.py flask db migrate -m "description"` 生成迁移
- [ ] 在 `src/api/migrations/versions/` 中审查迁移
- [ ] 把迁移文件提交到版本控制
- [ ] 为新的模型行为更新或补充测试
- [ ] 需要时更新文档

### 迁移故障排查

| 问题 | 解决方案 |
|---------|----------|
| `flask: command not found` | `export FLASK_APP=app.py` 或 `python -m flask db migrate` |
| `Could not locate a Flask application` | `export FLASK_APP=app.py` |
| `Target database is not up to date` | 运行 `flask db current`，然后 `flask db upgrade` |
| 数据库连接错误 | 检查 `DATABASE_URL` 或本地数据库凭据 |
| 迁移没有检测到变更 | 确认模型已在模块 init 路径中导入 |

全新 MySQL 回放冒烟测试：

```bash
cd src/api
RUN_MYSQL_MIGRATION_SMOKE=1 \
TEST_SQLALCHEMY_DATABASE_URI='mysql+pymysql://root:pass@127.0.0.1:33067/mysql?charset=utf8mb4' \
pytest -q tests/migrations/test_fresh_mysql_upgrade.py
```

## API 契约基线

### 标准响应格式

```json
{
  "code": 0,
  "message": "Success",
  "data": {}
}
```

### 常见错误码期望

| 代码 | 含义 | 典型动作 |
|------|---------|----------------|
| 0 | 成功 | 处理 `data` |
| 1001 | 未授权 | 重定向到登录 |
| 1004 | Token 过期 | 刷新 token 或强制重新鉴权 |
| 1005 | 无效 token | 清除 token 并重定向 |
| 9002 | 无权限 | 展示权限错误 |
| 5001+ | 业务错误 | 展示返回的 message |

### 鉴权头

```javascript
{
  "Authorization": "Bearer {token}",
  "Token": "{token}",
  "X-Request-ID": "{uuid}"
}
```

## 测试期望

### 测试文件结构

```text
src/api/tests/
├── conftest.py
├── service/
│   ├── shifu/
│   │   ├── test_models.py
│   │   ├── test_service.py
│   │   └── test_api.py
│   └── ...
└── common/
    └── fixtures/
        └── test_data.py
```

### 测试模式

- 测试文件命名：`test_[module].py`
- 测试函数命名：`test_[function]_[scenario]`
- 当能提高可读性时，把相关测试分组到 class 中
- 同时覆盖 happy path 和最高风险失败路径

### 覆盖率要求

- 目标代码覆盖率大于 80%
- 关键路径应争取 100% 覆盖
- 覆盖率命令：`pytest --cov=flaskr --cov-report=html`

### Ruff 发现与规则采纳

把 Ruff 发现视为代码或契约信号，而不是让配置变安静的请求。对于新增或变更代码中的发现：

1. 阅读 `ruff rule <CODE>`、最近的 `AGENTS.md`、实现、调用点，以及最近的测试。
2. 优先使用现有项目抽象或直接修代码。只要这次改写可能改变行为、错误路径、序列化、持久化、时序或公共/内部契约，就补上针对性回归覆盖。
3. 如果框架或协议要求被标记的写法，对该写法使用行内 `# noqa: CODE`，并在附近用英文说明原因。不要使用一揽子 `# noqa`。
4. 只有当文件或文件类别的目的本身与规则冲突时，才使用 `per-file-ignores`，例如 lint fixture 或不可变迁移历史。单个普通调用点应行内处理，而不是写进 `ruff.toml`。
5. 只有当已记录的仓库范围契约与规则根本冲突时，才使用全局 ignore。不要为了让无关 PR 通过而削弱 `select` 或 `ignore`。

对于 `RUF001`，在中文正文、通知格式、TTS 边界模式和回归 fixture 中保留有意使用的全角中文标点（`，`、`：`、`！`、`？` 和 `；`）。对每条已审核源码行使用 `# noqa: RUF001` 和英文原因抑制；不要把字符加入 `allowed-confusables`，因为该选项也会削弱独立的 RUF002 和 RUF003 检查。把其他所有易混字符视为可疑，并用目标码点替换，除非精确外部协议或回归 fixture 需要它。Python 测试模块和测试配置文件豁免 RUF001，因为它们必须原样保留用户文本、供应商 payload、协议样本和畸形输入。不要把该豁免复制到生产路径；把新测试放到 `tests/` 目录，或使用常规的 `test_*.py`、`*_test.py` 或 `conftest.py` 文件名。

对于 `N803`，即使对应序列化字段使用 camelCase，也保持 Python 函数和方法参数为 snake_case。在 DTO 内部把 Python 参数翻译成面向传输的属性，并更新内部关键字调用者；不要把 JSON 拼写复制进 Python 签名，也不要添加 ignore-name 模式。只有当调用方真正拥有关键字契约时，才保留外部 callback 或 override 签名，并在该参数处说明例外。

对于 `S101`，只在断言报告就是接口的地方使用 `assert`，例如 pytest 测试和可执行自测 fixture。生产校验、授权和状态守卫必须抛出显式异常，因为 Python 在 `-O` 下会移除断言。当路径消失，或现有测试模式已经拥有它时，删除过时的精确文件例外。

对于 `ARG002`，只有当仓库拥有该签名和全部调用者时，才删除方法参数。保持外部拥有的框架、协议、fixture 和 test-double 签名完整；在边界附近显式消费兼容值，使关键字契约保持可见且 lint 干净。

对 `ARG001` 应用同样的所有权测试：只有当全部调用者都由仓库拥有时，才删除函数参数；显式消费框架 callback、fixture、迁移和兼容门面所需的值。

对于 `ARG005`，当调用方不传任何参数时，优先使用零参数 lambda。当 callback 或 test double 必须接受位置或关键字参数时，使用命名 helper 或以 underscore 为前缀的参数，保留实际调用契约；不要重命名外部提供的关键字参数。

对于 `ANN002`，标注 `*args` 接受的元素类型，而不是 tuple 类型。当转发契约同质时使用窄的共享类型；只有真正异质的兼容转发才使用 `object`。

对于 `ANN003`，标注 `**kwargs` 接受的值类型，而不是字典类型。对同质选项优先使用共享窄值类型；只有当 adapter 真正转发异质关键字值时才使用 `object`。

对于 `D102`，记录每个公共生产方法和工具方法的可观察职责或契约。以行为为中心的 pytest 函数保持豁免；它们的名称和断言就是可执行规格。不要添加只是复述方法名或签名的泛化 docstring。

对于 `D103`，记录每个公共生产函数或贡献者工具函数所拥有的结果、副作用、数据边界、路由组或命令行工作流。优先使用 `Return`、`Yield`、`Serialize`、`Register` 和 `Run` 这类契约动词；只有当精确函数名还需要目的地、来源或协议边界才能 unambiguous 时，才展开它。不要添加只说“执行了某操作”的不透明填充。以行为命名的测试和有意极简的架构 fixture 保持豁免，带有常规 `upgrade` 和 `downgrade` hook 的不可变 Alembic 历史也是。因为函数 docstring 是运行时数据，批量改动前先搜索 `__doc__` 或 inspection 消费者；在只移除新增函数 docstring 后证明可执行 AST 相等，并对受影响表面运行针对性 Swagger 或 CLI 回归。

对于 `N815`，即使公共 JSON 契约使用 camelCase，也保持 Python DTO 字段名为 snake_case。Pydantic DTO 应用 `Field(alias="...")` 声明公共名，通过 `populate_by_name` 接受内部构造，并用 `by_alias=True` 序列化；测试必须同时锁住 Python 属性和精确 wire key。如果非 Pydantic DTO 的注解名直接驱动 JSON 和生成的 Swagger，可为该字段使用一条带说明的行内抑制。不要豁免整个文件，也不要重命名外部契约。

对于 `G004`，保持日志消息构造惰性：把常量消息及其值作为位置 logging 参数传入，保留消息文本、参数顺序和日志级别。普通或 `!s` 插值使用 `%s`，匹配转换使用 `%r`/`%a`，字面百分号转义为 `%%`。当 f-string 字段有 logging 插值无法精确表达的格式说明时，只用 `format(value, "spec")` 预格式化该字段，并让其余消息保持参数化。不要只为了让规则安静，就把 f-string 藏进临时变量。

对于 Ruff `EM` 族（`EM101`-`EM103`），在抛出前立刻把直接字符串字面量、上下文 f-string 或 `.format()` 表达式赋给无冲突局部变量，然后把该局部变量作为同一个构造参数传入。优先使用 `message`；当整个函数已经使用该名称时，使用 `error_message` 或 `exception_message`。保留异常类型、精确消息表达式、其余位置和关键字参数，以及任何显式 `raise ... from` 原因。不要通过拼接字符串、改变格式，或从错误中丢掉有用值来规避规则。引入局部变量前扫描整个函数，包括嵌套作用域，避免新绑定遮蔽已有局部变量、自由变量或全局查找。批量改动时，证明改写在结构上可逆，并运行覆盖受影响错误路径的测试。`pytest.raises` 体必须保持一条简单语句：在 context-manager 块前绑定固定消息，块内只保留 `raise`。

对于 `D205`，把完整摘要放在 docstring 的第一物理行，然后在细节、小节或嵌入协议内容前恰好留一行空行。不要把摘要折到多行源码；行长度由 formatter 负责。在 Flasgger 路由 docstring 中，把 `---` YAML 分隔符放在该空行之后，并运行 Swagger-docstring parser 回归测试，而不是抑制规则或重新缩进规格。

对于 `D107`，描述构造实例确立了什么：它的 payload、拥有的状态、绑定的依赖，或不明显的 setup。当那就是全部契约时，把构造函数 docstring 保持为一行；只有真实副作用或不变量才补充细节。不要复制签名，也不要只为满足规则写一句泛化的 “Initialize the object”。Test double 应点出它们所代替的状态或协作者。

对于 `D105`，记录可观察的协议契约，而不是复述魔术方法名。`__json__` docstring 应说明它返回标量、JSON 兼容数据还是 JSON 字符串；mapping 方法指出它们代理的 key 或 payload；表示和比较方法描述可见结果。足够时保持一行契约，并让 test-double operator 点出它们构建的假表达式。不要写 “Implement `__json__`” 这类填充。

对于 `N806`，把 class 声明或模块级 class import 与函数内部绑定的值区分开。真实 class 名保持 CapWords，但把本地加载的 model 或 constructor 绑定到描述性 snake_case 名，例如 `draft_shifu_model` 或 `session_factory`，然后在函数中一致使用该名称。保留有意的惰性 import，而不是仅为了 lint 把它移到模块作用域；也不要为普通局部绑定添加按文件例外。运行覆盖本地 loader 或 factory 的测试，避免重命名悄然把查询或对象创建指向另一个 class。

对于 `D100`，在其所有权边界描述模块为何存在。生产模块点出它拥有的服务职责、协议或数据契约；测试模块点出它保护的行为组；可执行脚本点出它执行的操作。不要机械复述文件名，也不要写 “module helpers” 这类泛化填充。删除未被引用的空占位，而不是给它发明一个目的。把 docstring 保持为第一条 Python 语句，同时保留 shebang、编码注释和文件级工具指令。因为它会改变 `module.__doc__`，批量采纳前先搜索运行时 introspection，并在移除新增模块 docstring 后验证可执行 AST 相等。

对于 `D101`，记录 class 的角色或可观察契约，而不是它的拼写。DTO 点出它们携带的 payload，持久化模型点出它们存储的状态，协议点出它们要求的操作，异常点出它们发出的失败，test double 点出它们模拟的协作者或失败。测试 class 点出它验证的行为组。足够时把契约保持为一行，不要写 “Class for X” 这类填充，也不要只把 CapWords 名拆成一句话。因为 class docstring 会通过 `Class.__doc__` 可见，并可能喂给 schema 或 inspection 框架，添加前先搜索运行时 introspection，并在只移除新增 class docstring 后验证可执行 AST 相等。当被触及的 class 注册到 Swagger 或由 Pydantic 创建时，运行针对性 schema 测试。
对于 `TC002` 和 `TC003`，只有在确认每次使用都是 Python 运行时不需要解析的注解后，才把第三方或标准库 import 移进 `if TYPE_CHECKING:` 块。推迟求值注解是常规证明；带引号的注解，或从未求值的局部变量注解也是安全的。import 的来源并不能证明它可以延迟：保留会提供注册副作用的 import，以及框架、装饰器、没有推迟求值的函数签名，或显式运行时反射在导入模块时会解析的 import。当一条 import 语句混合运行时值和仅注解类型时，拆开语句，只移动仅类型名称。

Pydantic `BaseModel` 字段在 `ruff.toml` 中被声明为运行时求值，因此它们的标准库和第三方字段类型都保持正常导入；不要用散落的 `noqa` 注释替换该契约。当引入另一个共享的运行时求值基类或装饰器时，在 Ruff 中集中建模，并添加 import 或 schema 冒烟测试。只对 Ruff 无法建模的一次性运行时消费者使用带说明的窄抑制。移动 import 前，在测试中搜索 `monkeypatch`、`getattr` 和模块属性断言：保留真实注入缝，但删除从不影响被测代码的过时 patch，而不是保留假运行时依赖。

一次只采纳或移除一个规则单元。规则单元通常是一个 Ruff code；只有当它们报告同一写法、并且有相同修复和例外边界时，才合并 code。每个规则 PR 基于前一个规则分支，并把无关清理排除在 diff 之外。在 `docs/exec-plans/active/ruff-rule-minimization.md` 中跟踪该栈和规则普查。

对每个规则单元，先运行针对性检查，再运行仓库门禁：

```bash
ruff check . --select CODE
ruff check .
ruff format --check .
python scripts/check_repo_harness.py
```

对每个被触及的运行时表面运行最近的行为测试。Lint 通过不能替代测试覆盖。提交前还要运行 `python scripts/check_dev_tools.py` 和 `lefthook run pre-commit --all-files`。

## 开发工作流

### 分支命名

- 功能：`feat/description-of-feature`
- 缺陷修复：`fix/description-of-fix`
- 重构：`refactor/description`
- 文档：`docs/description`

### Pull Request 清单

- [ ] 代码遵循项目约定
- [ ] Pre-commit hook 通过
- [ ] 测试已添加或更新并且通过
- [ ] 需要时已创建数据库迁移
- [ ] 需要时已更新文档
- [ ] PR 标题遵循 Conventional Commits
- [ ] 面向用户的表面没有硬编码字符串
- [ ] 代码中没有密钥

### 部署流程

1. 合并到 `main`
2. CI/CD 运行测试并构建
3. 部署到 staging
4. 运行冒烟测试
5. 部署到生产

## CI/CD 与发布流程

### 工作流清单

- `backend-tests.yml`：对 `src/api/**` 变更运行后端测试，并在直接推送到 `main` 时运行。
- `frontend-tests.yml`：对前端和共享 i18n 变更运行 Cook Web Jest 测试，并对无关 PR 报告成功的 no-op check。
- `prettier-check.yml`：对前端变更检查 Cook Web 格式。
- `repo-harness.yml`：`Static Checks` job 在进入 `main` 的 PR 上校验架构边界、生成的 AI 和知识产物、翻译对等与 locale 元数据，以及 MarkdownFlow release 钉版本。
- `runtime-harness.yml`：对影响运行时的后端、前端、Docker 和脚本变更，运行基于 Docker 的 Playwright 冒烟 harness。
- `prepare-release.yml`：根据请求的 `vX.Y.Z` 版本手动准备 release draft，并更新已版本化的项目文件。
- `build-latest.yml`：从 `main` 构建最新已发布 Docker 镜像，也可以手动触发。
- `build-on-release.yml`：在 GitHub release 发布时构建并推送带 release tag 的 Docker 镜像。

### 发布路径

1. 从 `prepare-release.yml` 开始，并提供以 `v` 开头的版本，例如 `v1.5.0`。
2. 在发布 GitHub release 前，核对生成的版本更新、release draft 内容，以及 tag 期望。
3. Release draft 包含自上一 `vX.Y.Z` tag 以来的仓库提交，以及当钉住的 `markdown-flow` 或 `markdown-flow-ui` 版本变化时的 MarkdownFlow 依赖更新；当对应库仓库存在 tag 时，依赖说明由该 tag 范围生成。当匹配 tag 不存在时，使用 registry 发布时间限制对该依赖仓库的 GitHub 提交查找。
4. 发布 release 会触发 `build-on-release.yml`，它会校验 tag，跳过 draft 或 prerelease，并构建带 release tag 的镜像。
5. `main` 继续驱动 `build-latest.yml`，因此 `:latest` 镜像和带 release tag 的镜像必须保持语义对齐。
6. 镜像发布后，在把这次发布视为就绪前，冒烟检查 pinned 或 latest Docker Compose 启动路径、后端启动，以及主要前端入口路径。

### 发布与自动化规则

- 让 GitHub Actions secrets 和 vars 负责 registry 凭据、push 开关，以及发布相关配置。
- 除非自动化表面本身被有意改变，否则保持 workflow 路径过滤和触发意图。
- 当改变镜像名、tag 或发布语义时，在同一任务中一起审查 GitHub workflow 和 `docker-compose*.yml` 文件。

## 性能指引

### 数据库优化

- 始终为 `_bid` 和其他业务键关系列建立索引
- 大批量写入优先使用批处理
- 对大结果集使用分页
- 避免 N+1 查询
- 在适当时缓存频繁访问的热点数据

### API 性能

- 常见读取目标低于 200ms，常见写入低于 500ms
- 默认分页：20 条，最多 100 条
- 只有真正适合 I/O 工作时才使用异步模式
- 对容易被滥用的端点应用限流
- 对外部依赖使用请求超时

### 前端性能

- 对重路由和组件做懒加载
- 使用合适的图片格式和尺寸
- 控制共享 bundle
- 通过共享数据层缓存 API 响应
- 对搜索和类似流程的用户输入做 debounce

## 环境配置

### 配置文件

- Docker：`docker/.env`
- 本地开发：组件级 `.env` 文件
- Docker 示例文件：`docker/.env.example.full`
- 重要分组：LLM API key、数据库、Redis、鉴权、存储、应用配置

### 管理环境变量

新增或修改环境变量时：

1. 更新 `src/api/flaskr/common/config.py` 中的配置定义
2. 用 `cd src/api && python scripts/generate_env_examples.py` 重新生成示例
3. 需要时更新 fixture 和测试

## 国际化规则

- 所有面向用户的字符串都必须使用 i18n
- 共享翻译位于 `src/i18n/<locale>`
- 不要在 `public/locales` 下添加主翻译
- 后端应通过共享 helper 引用翻译 key
- 前端面向用户的 locale 必须与 `src/i18n/locales.json` 保持对齐

新增 namespace 时：

- 更新每个受支持 locale
- 运行 `python scripts/generate_languages.py`
- 运行 `python scripts/check_translations.py`
- 运行 `python scripts/check_translation_usage.py --fail-on-unused`

## 文件和目录命名约定

### 目录命名

- 目录使用 kebab-case
- 保留 Next.js 特殊文件夹约定，例如 `(group)`、`[dynamic]` 和 `[[...catchAll]]`
- 在按职责组织的共享模块中保留学员和老师兼容性；不要为同一行为重建平行源码树

### 文件命名

- 组件文件：PascalCase，例如 `UserProfile.tsx`
- 普通 TypeScript 或 JavaScript 文件：kebab-case
- CSS 和 SCSS 文件：kebab-case
- CSS module：匹配组件名
- 测试文件：匹配被测文件，并使用 `.test.ts` 或 `.spec.ts`
- 类型定义文件：kebab-case，带 `.d.ts`
- 配置文件：小写，带点号

### 特殊情况（Next.js）

- API 路由：`route.ts`
- 页面：`page.tsx`
- 布局：`layout.tsx`
- 加载状态：`loading.tsx`
- 错误边界：`error.tsx`

## 故障排查

### 常见问题与解决方案

| 问题 | 解决方案 |
|-------|----------|
| Flask 应用无法启动 | 检查 `FLASK_APP=app.py` |
| 数据库连接失败 | 检查 MySQL 和凭据 |
| 迁移没有检测到变更 | 确认模型已导入 |
| 前端无法连接 API | 检查 CORS 和 API URL 配置 |
| Lefthook 检查失败 | 运行 `lefthook install` |
| Hook 从不运行，或工具报告 “command not found” | 运行 `python scripts/check_dev_tools.py` 并安装它列出的内容 |
| 测试因 import 错误失败 | 检查 `PYTHONPATH` 和本地环境 |
| Docker 构建失败 | 确认所需 `.env` 文件存在 |
| Cook Web 中的 TypeScript 错误 | 运行 `npm run type-check` |
| Redis 连接可选 | 在许多流程中，应用没有 Redis 仍可运行 |

### 调试命令

```bash
# Check Python environment
which python
pip list

# Check Node environment
node --version
npm --version

# Check database connection
mysql -u root -p -e "SHOW DATABASES;"

# Check Flask configuration
flask routes

# Check Docker status
docker ps
docker compose logs [service]

# Check port usage
lsof -i :5000
lsof -i :3000
```

## 额外资源

- Flask Documentation: <https://flask.palletsprojects.com/>
- SQLAlchemy Documentation: <https://www.sqlalchemy.org/>
- React Documentation: <https://react.dev/>
- Next.js Documentation: <https://nextjs.org/>
- Conventional Commits: <https://www.conventionalcommits.org/>