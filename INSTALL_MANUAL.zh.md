# 从源码逐步安装

本文是 [`INSTALL_MANUAL.md`](INSTALL_MANUAL.md) 的中文译本。代理、工具链和校验仍以英文版为准；若两份文档不一致，以英文版为准。

## 前置条件

### 架构概览

AI-Shifu 由两个主要组件组成：

```bash
src/
├── api/          # 后端 API 服务（Flask/Python）
└── web/          # Cook Web 前端（Next.js）
```

- **api**：用 Flask 构建的后端 API 服务
- **web**：用 Next.js 构建的 Cook Web 前端，用于创建、管理和学习课程

### 所需工具和服务

- **Python 3.11+**：用于后端 API
- **Node.js 22.16.0**：用于前端应用
- **MySQL 8.0+**：用于数据库存储
- **Redis**：用于缓存和会话管理
- **Docker & Docker Compose**（推荐，便于部署）

### 所需 API Key

必须至少配置一个 LLM 提供方：

- **OpenAI** API Key
- **Baidu ERNIE** API 凭据
- **ByteDance Volcengine Ark** API Key
- **SiliconFlow** API Key
- **Zhipu GLM** API Key
- **DeepSeek** API Key
- **Alibaba Qwen** API Key

### 可选服务

- **Alibaba Cloud OSS**：用于文件存储
- **Alibaba Cloud SMS**：用于手机验证
- **Langfuse**：用于 LLM 追踪
- **Email SMTP**：用于邮箱验证

## 安装步骤

### 第 1 步：克隆仓库

```bash
git clone https://github.com/ai-shifu/ai-shifu.git
cd ai-shifu
```

### 第 2 步：准备环境变量

复制完整环境模板（已与 Docker 默认值对齐）：

```bash
cp docker/.env.example.full docker/.env
```

对于基于 Docker 的工作流，唯一必须修改的是至少添加一个 LLM 提供方 key（例如 `OPENAI_API_KEY`、`ERNIE_API_KEY`、`GLM_API_KEY` 等）。其余变量已经有与捆绑 MySQL/Redis 服务匹配的安全默认值。

### 第 3 步：配置环境变量

编辑 `.env` 文件并配置必要设置。

#### 必填变量（必须配置）

这些变量对应用运行是必需的：

1. **数据库连接**
   - `SQLALCHEMY_DATABASE_URI`：MySQL 连接字符串
   - 示例：`mysql://root:password@localhost:3306/ai-shifu?charset=utf8mb4`

2. **安全**
   - `SECRET_KEY`：用于鉴权的 JWT 签名密钥
   - 生成安全密钥：`python -c "import secrets; print(secrets.token_urlsafe(32))"`
   - **重要**：开发、测试和生产环境应使用不同密钥

3. **LLM 提供方**（至少需要一个）
   - 可选：OpenAI、ERNIE、ARK、SiliconFlow、GLM、DeepSeek、Qwen
   - 具体提供方配置见 `.env.example.full`

#### 配置参考

- `docker/.env.example.full`：权威模板，按分组（Database、Redis、Auth、LLM 等）列出每个环境变量及其默认值和说明。把它复制为 `.env` 后就地编辑。
- **Docker 提醒**：容器化安装唯一必须改的是至少设置一个 LLM API key（例如 OpenAI、ERNIE、GLM）。只有在不使用捆绑服务时，才需要更新数据库/Redis URL。

#### 重要说明

- 所有敏感值（API key、密码）都应妥善保管
- 永远不要把 `.env` 文件提交到版本控制
- 生产部署应使用环境特定配置
- 每个变量的详细说明见示例文件

#### 可选的 Gemini Live 语音追问

Gemini Live 默认关闭。在目标环境验证完浏览器直连流程之前，保持 `GEMINI_LIVE_ENABLED=false`。要暴露允许列表中的 Live 追问模型，请配置有效的 `GEMINI_API_KEY`，保持 Redis 可用于已鉴权的会话绑定和容量租约，然后设置：

```bash
GEMINI_LIVE_ENABLED=true
```

受支持的 Live 追问模型是 `gemini-3.8-live`（Gemini 3.8 Live）。Gemini 模型发现响应必须为该模型声明 `bidiGenerateContent`。更早的 Live 模型和 Extended Thinking 变体不受支持；已保存课程必须在追问设置中选择 `Gemini Live`。已知不受支持的 Live 选项会保持禁用，既不能调用文本/SSE 生成，也不能获取 Live 凭据。Live 会话设置会省略该模型不支持的 `thinkingConfig`。

Live 就绪状态与普通 HTTP 健康检查是分开的。启动时，每个已启用的 API worker 都会调度一个后台任务，解析生效（环境或数据库）开关，并初始化共享 Redis recovery guard，但不会铸造凭据。Redis 淘汰策略不是准入前置条件。建议使用 `noeviction` 来保留凭据风险记录：如果使用会淘汰的策略，记录可能在 Google 凭据过期前消失，从而低估未完成凭据数量或中断所有权。容量保证依赖于记录保留。缺少记账标记或 Redis run ID 发生变化时，会启动完整的共享 15 分钟安全窗口；重复的 worker 启动和探测不会重置或缩短它。不要通过删除记账记录来绕过这个窗口。
如果启动时的配置/Redis 查找失败（包括数据库查找回退到 disabled），每个 API worker 会有一个守护任务每 30 秒重试一次，最多 20 次。一旦标记初始化完成，即使仍在 warming，它也会停止；显式的环境关闭开关会完成第一次尝试且不重试。更长的中断仍需要下面的部署就绪门禁；重试不会绕过它。
Gunicorn preload 会在 master 中跳过准备；现有 `post_fork` hook 会在每个 worker 的连接池和 tracing 重置后初始化它。初始化对每个进程是幂等的，包括非 preload 的 app factory。
Celery 启动会用 `serving_http=False` 创建 Flask app，并在路由注册前记录该值。这个进程本地角色会在 prefork 父进程以及 queue/beat worker 中跳过 Live 准备；它不会改变 rollout 开关，也不会在 Celery 子进程中启动不必要的就绪线程。
初始查找和重试都不会跑在 HTTP/启动线程上。配置缓存读取使用带 1 秒连接/读取超时的隔离 Redis 客户端；现有数据库查找仍留在那一个后台任务内。卡住的数据库操作不会被 join，也不会创建替代任务或阻塞 worker。

在宣布 Live 可用之前，通过正常已鉴权 API 传输查询 `GET /api/learn/live-follow-up/readiness`，并要求 `data.status == "ready"`。其他有界状态是 `warming`、`unavailable` 和 `disabled`；`retry_after_ms` 只是建议的探测间隔，不是凭据过期时间。该探测不分配用户/会话容量，只读取显式覆盖或有界缓存配置，没有数据库回退；冷/缺失的配置缓存在后台查找或现有 config 服务填充之前都是 `unavailable`。启动后的缓存未命中会在同一个 worker 本地任务槽中调度重新填充，绝不会内联执行。活跃（包括卡住）的任务不会被替换，新任务启动在完成或启动失败后有 30 秒冷却。每个任务保留“首次尝试加 20 次重试”的预算。能力校验复用同一个已解析开关，而不是再做一次共享缓存查找。
已确认的数据库缺失会作为单独元数据缓存 24 小时，因此受支持的默认关闭状态会稳定为 `disabled`；短暂的数据库失败永远不会创建这个标记。普通 config 服务的正向缓存写入在开关被创建或更新时立即优先。探测不返回凭据或 Redis 标识。启动探测使用有界 Redis socket 等待；Live 中断不得让普通 `/health` 或文本追问失败。原始追问面板也会在启用新的 Live 输入前探测，并在不可用时刷新，但不会自动连接、请求麦克风权限，或暴露内部倒计时。铸造时准入仍会在就绪成功后以原子方式强制执行同一守卫。
探测还要求允许列表模型已发现的 Bidi 能力；禁用/缺失的 Gemini 提供方、缺失模型，或仅文本能力，即使 Redis 已就绪也会返回 `unavailable`。发现保留现有启动生命周期：在修正提供方配置或启动 ListModels 失败后，重启 API worker 并验证就绪。探测不会调用 Gemini。

API 会铸造一次性、短时的 Gemini 凭据，并约束到所选模型、语音和服务器构建的 prompt。浏览器随后直接打开 Gemini Live WebSocket，因此 AI-Shifu 入口不需要 WebSocket Upgrade 路由。生产环境仍需要 HTTPS 才能访问麦克风，并且浏览器必须能到达 `generativelanguage.googleapis.com`。
服务器也必须能到达 token 创建 API。如果设置了 `GEMINI_API_URL`，token 创建会复用该 HTTPS 基址（包括任何代理路径前缀），并追加 `/v1beta/auth_tokens`；已有的末尾 `/v1beta` 会复用而不是重复。如果未设置，token 创建使用 Google 官方 API。
只应配置受信任代理，因为它会收到 API key 和私有课程上下文。这个设置永远不会改变浏览器的 Gemini WebSocket 目标。当配置的端点失败时，token 请求不会跟随重定向，也不会回退到另一个主机。
关闭该开关即可回滚 Live，而不必改动使用文本追问模型的课程。

### 第 4 步：构建最新 Docker 镜像并启动栈

1. 确保 `docker/.env` 至少包含一个 LLM API key。
2. 从仓库根目录构建标记为 `:latest` 的后端和前端镜像：

```bash
docker build -t aishifu/ai-shifu-api:latest -f src/api/Dockerfile .
docker build -t aishifu/ai-shifu-cook-web:latest -f src/web/Dockerfile .
```

3. 用跟踪 `:latest` 标签的 compose 组合启动容器：

```bash
cd docker
docker compose -f docker-compose.latest.yml up -d
```

`docker-compose.latest.yml` 始终使用最新镜像（来自 Docker Hub 或你自己的本地构建）。如果你需要固定发布标签以获得可复现环境，请改用 `docker-compose.yml`。

### 第 5 步：手工安装（开发）

本节覆盖开发用途，或你需要对安装过程有更多控制时的手工安装。

#### 第 5.1 步：准备数据库服务

在本机启动 MySQL 和 Redis，或使用 Docker：

```bash
# 只用 Docker 跑数据库
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=ai-shifu -e MYSQL_DATABASE=ai-shifu mysql:latest
docker run -d --name redis -p 6379:6379 redis:latest
```

#### 第 5.2 步：为本地开发配置环境

为本地开发更新 `.env` 文件：

```bash
# 把数据库 URL 更新为本地服务
SQLALCHEMY_DATABASE_URI="mysql://root:ai-shifu@localhost:3306/ai-shifu"

# 更新 API 基址
REACT_APP_BASEURL="http://localhost:5800"
```

#### 第 5.3 步：启动后端 API

```bash
cd src/api
# 从 docker 目录复制环境配置
cp ../../docker/.env .env

# 安装 Python 依赖
pip install -r requirements.txt

# 初始化数据库
flask db upgrade

# 启动 API 服务器
gunicorn -w 4 -b 0.0.0.0:5800 'app:app' --timeout 300 --log-level debug
```

#### 第 5.4 步：启动 Web 前端和 CMS

```bash
cd src/web
# 安装 Node.js 依赖
npm install  # 或使用 pnpm install

# 启动开发服务器
npm run dev
```

Cook Web（现在同时承载学习体验和创作控制台）将在 `http://localhost:3000` 可用。

#### 第 5.5 步：安装代码质量钩子（贡献者）

如果你计划提交变更，请安装 lefthook git hooks，让 CI 中的同一套 pre-commit 检查也在本地运行。**不做这一步，提交时检查会被静默跳过。**

为你的平台安装 lefthook。

macOS（Homebrew）：

```bash
brew install lefthook
```

Linux 或 Windows（npm）：

```bash
npm install -g @evilmartians/lefthook
```

然后从仓库根目录安装其余开发工具：

```bash
# 从仓库根目录执行
pip install ruff==0.16.5 commitizen==4.16.2 pre-commit-hooks==6.0.0
(cd src/web && npm ci)   # 提供 prettier + eslint
lefthook install

# 校验工具链（报告缺失项以及如何安装）
python scripts/check_dev_tools.py
```

## 故障排查

### 常见问题

1. **数据库连接失败**
   - 确保 MySQL 正在运行且可访问
   - 检查 `.env` 中的数据库凭据
   - 运行 `flask db upgrade` 初始化表

2. **Redis 连接失败**
   - 确保 Redis 正在运行且可访问
   - 检查 `.env` 中的 Redis 配置

3. **LLM API 错误**
   - 验证 API key 是否正确
   - 检查 API 基址
   - 确保模型名与提供方匹配

4. **前端构建失败**
   - 确保 Node.js 版本是 22.16.0
   - 清理 node_modules 后重装：`rm -rf node_modules && npm install`
   - 检查环境变量问题

5. **Pre-commit hooks 没有运行 / “command not found”**
   - 确认你已在这个 clone 中运行过一次 `lefthook install`
   - 运行 `python scripts/check_dev_tools.py` 查看缺少哪些工具以及如何安装

### 日志文件

- API 日志：查看 gunicorn 输出或 `/var/log/ai-shifu.log`
- 前端日志：查看浏览器控制台或终端输出

## 访问应用

### 手工安装

- 用户界面：`http://localhost:3000`（或配置的 PORT）
- 脚本编辑器：`http://localhost:3001`
- API：`http://localhost:5800`

### 默认登录

- 使用任意手机号进行注册/登录
- 默认验证码：`1024`
