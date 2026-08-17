# MCP 智能体网关（MCP Gateway）— 项目规划与开发步骤

> 本文档是项目的「完整交接文档」，目标是：换一个对话后，AI 只看这份文档 + 本仓库代码，就能无缝继续开发。

---

## 0. 给新对话的一句话开场（复制即用）

> 我正在开发一个「MCP 智能体网关」项目（企业级工具网关，统一接入并管理多个 MCP Server，上层 LLM Agent 通过一个入口调用所有工具）。完整规划见本文件 `MCP-Gateway项目规划与开发步骤.md`。请先读这份文档，然后从「当前进度」处继续。

---

## 1. 项目定位（一句话）

**统一接入并管理多个 MCP Server 的企业级工具网关**：上层 LLM Agent 只对接网关一个入口，即可调用背后任意多个 MCP Server 提供的工具，全程覆盖鉴权、限流、日志、指标。

## 2. 为什么选这个方向

| 诉求 | 怎么满足 |
|---|---|
| 靠近企业需求 | MCP 是企业落地 Agent 的事实标准，企业要的正是「多工具统一接入/鉴权/限流/审计」，而不是再一个问答机器人 |
| 项目小、6 周可完成 | MVP 只做「网关」这一层，单人可控 |
| 明显能扩到企业级 | 天然长在多租户、RBAC、K8s、模型路由、审计这些企业刚需上 |
| 复用已有能力 | 直接复用作者简历上最硬的差异化能力：MCP 二次开发 + FastAPI + 大模型部署 |

## 3. 个人背景（作者）

- 陈晓伟，2026 届本科（福州理工学院，智能科学与技术，已毕业，可立即到岗）
- 技术栈：Python（FastAPI/Flask/asyncio）、MCP 智能体开发、大模型微调（LoRA/Unsloth/Ollama/GGUF）、Dify 工作流、RAG/LangChain、MySQL、Docker/Linux、Java（Spring Cloud）
- 相关经验：Windows MCP Server 二次开发（毕设 90+）、鸿蒙智能家居 AI 控制闭环（省赛三等奖）、海科 NL2SQL（Dify + MCP + 达梦）

## 4. 技术选型（贴近企业工程化）

| 类别 | 选择 | 说明 |
|---|---|---|
| 语言/框架 | Python 3.12 + FastAPI + uvicorn + asyncio | 全异步 |
| MCP 协议 | 官方 `mcp` SDK | 支持 stdio / SSE / Streamable HTTP 三种传输 |
| 配置 | pydantic-settings + YAML | config 驱动，不写死 |
| 鉴权 | 自实现 API Key + `python-jose` JWT（预留） | 展示原理而非只会调库 |
| 限流 | 自实现令牌桶 | 展示原理 |
| 可观测 | structlog 结构化日志 + prometheus-fastapi-instrumentator | 暴露 `/metrics` |
| 存储 | MVP 用 SQLite（存 Key/配额），接口层抽象 | 可无缝换 PostgreSQL |
| 测试/部署 | pytest + pytest-asyncio + httpx；Docker + docker-compose；GitHub Actions CI | 完整工程闭环 |
| 包管理 | uv（或 poetry） | 现代 Python 工程标准 |

## 5. MVP 功能范围（6 周）

核心闭环：**注册 MCP Server → 网关聚合工具列表 → 统一 REST API 供 Agent 调用 → 鉴权/限流/日志全程覆盖**

1. MCP Server 动态配置接入（YAML 声明，无需改代码）
2. 多传输协议客户端封装 + 工具注册中心（把多个 server 的工具聚合）
3. 统一 API：`GET /tools`（列工具）、`POST /tools/{name}/call`（调用）
4. API Key 鉴权中间件 + 令牌桶限流
5. 结构化日志 + `/metrics` + `/health`
6. 一个示例 MCP Server（`demo_sql_server`：自然语言 → SQL 查询）+ 一个 LLM Agent 调用示例
7. 单测 + docker-compose 一键启动 + CI

## 6. 系统架构

```
                        ┌─────────────────────────────┐
                        │        LLM Agent / 应用      │
                        │  (LangChain / OpenAI 等)     │
                        └──────────────┬──────────────┘
                                       │ 统一 REST API
                        ┌──────────────▼──────────────┐
                        │       MCP Gateway (FastAPI)  │
                        │  ┌────────────────────────┐  │
                        │  │  鉴权 (API Key)         │  │
                        │  │  限流 (令牌桶)          │  │
                        │  │  日志 / 指标 (横切层)   │  │
                        │  └───────────┬────────────┘  │
                        │  ┌───────────▼────────────┐  │
                        │  │  工具注册中心 registry  │  │
                        │  │  (聚合所有 server 工具) │  │
                        │  └───────────┬────────────┘  │
                        │  ┌───────────▼────────────┐  │
                        │  │  MCP 客户端 (多传输)    │  │
                        │  └────────────────────────┘  │
                        └───────┬───────────┬───────────┘
                        stdio ──┤           ├── Streamable HTTP / SSE
                 ┌──────────────▼──┐   ┌────▼───────────────┐
                 │ MCP Server #1   │   │ MCP Server #2 ...   │
                 │ (demo_sql_server)│   │  (数据库/内部API等) │
                 └─────────────────┘   └─────────────────────┘
```

## 7. 目录结构（企业级分层）

```
mcp-gateway/
├── app/
│   ├── main.py              # FastAPI 应用入口
│   ├── config.py            # 配置中心（pydantic-settings）
│   ├── core/                # 横切关注点
│   │   ├── security.py      # API Key / JWT 鉴权
│   │   ├── rate_limit.py    # 令牌桶限流
│   │   ├── logging.py       # 结构化日志
│   │   └── metrics.py       # Prometheus 指标
│   ├── mcp/                 # 协议层
│   │   ├── registry.py      # 工具注册中心（核心）
│   │   ├── client.py        # MCP 客户端封装
│   │   ├── transports.py    # stdio/sse/http 三种传输
│   │   └── schemas.py       # Tool / Call schema
│   ├── api/                 # 接口层
│   │   ├── deps.py          # 依赖注入
│   │   └── routes/          # tools / servers / health
│   └── schemas/             # Pydantic 模型
├── servers/
│   └── demo_sql_server/     # 示例 MCP Server（自然语言→SQL）
├── examples/                # LLM Agent 调用示例
├── tests/                   # 单元测试
├── config/                  # YAML 配置文件（server 声明、gateway 配置）
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .github/workflows/ci.yml # CI
└── README.md                # 含架构图 + 拓展路径 + 演示录屏
```

---

## 8. 分阶段开发步骤（详细到可执行）

### 阶段 0：环境准备（0.5 天）
- [x] 安装 Python 3.12、uv、Git、Docker
- [x] `git init` + 建 GitHub 空仓库并关联远程
- [x] 创建目录结构骨架

### 阶段 1：工程骨架 + CI 跑通（W1）
- [x] 初始化 `pyproject.toml`（依赖：fastapi、uvicorn、mcp、pydantic-settings、structlog、prometheus-fastapi-instrumentator、python-jose、pyyaml；dev：pytest、pytest-asyncio、httpx、ruff）
- [x] `app/main.py`：最小 FastAPI 应用，含 `/health` 和 `/metrics`
- [x] `app/config.py`：pydantic-settings，从 YAML/环境变量读配置
- [x] `app/core/logging.py`：structlog 结构化日志（JSON）
- [x] 加 `.github/workflows/ci.yml`：ruff lint + pytest
- [x] 写 `README.md` 骨架 + 架构图
- **验收**：`docker compose up` 后访问 `/health` 返回 ok，CI 全绿 ✅（2026-08-15 本地与 GitHub Actions 均通过，run #31890245357）

### 阶段 2：协议层打通（W2）—— 核心
- [x] `app/mcp/transports.py`：实现 stdio / SSE / Streamable HTTP 三种传输的连接管理
- [x] `app/mcp/client.py`：封装 MCP 客户端（连接、list_tools、call_tool）
- [x] `app/mcp/schemas.py`：Tool / ToolCall 的 Pydantic 模型
- [x] `app/mcp/registry.py`：工具注册中心——读取 YAML 中声明的 server，建立连接，聚合所有工具列表（工具名 → server 的映射）
- [x] `servers/demo_sql_server/`：写一个最小 MCP Server（先只暴露 1 个 `echo` 工具，跑通 stdio）
- **验收**：网关能连上 demo server，并在内部 registry 中列出其工具 ✅（2026-08-16：`pytest` 12/12 通过；uvicorn 启动日志 `registry_ready connected=1 tools=1`；`demo_sql__echo` 调用返回正确）

### 阶段 3：统一 API + 鉴权限流（W3）
- [x] `app/core/security.py`：API Key 中间件（Header `X-API-Key` 校验，Key 存 SQLite/内存）
- [x] `app/core/rate_limit.py`：令牌桶限流（按 API Key 维度）
- [x] `app/api/routes/tools.py`：`GET /tools`、`POST /tools/{name}/call`
- [x] `app/api/deps.py`：FastAPI 依赖注入（拿当前 key、限流器、registry）
- [x] `app/api/routes/servers.py`：server 列表/状态查看
- [x] 补单元测试（鉴权 401、限流 429、工具调用成功/失败）
- **验收**：无 Key 调 `/tools` 返回 401；超限返回 429；带 Key 可正常列出和调用工具 ✅（2026-08-16：`pytest` 26/26 通过；uvicorn 真实冒烟：无 Key/错 Key 均 401、第 61 次请求 429 带 Retry-After、带 Key 列出并调用 `demo_sql__echo` 成功、`/servers` 状态正常）

### 阶段 4：可观测 + demo + 部署（W4）
- [x] 完善 `app/core/metrics.py`：工具调用次数/延迟/错误率等指标
- [x] `demo_sql_server` 升级：自然语言 → SQL 查询（复用作者 NL2SQL 经验，SQLite 落库）
- [x] 写 `Dockerfile` + `docker-compose.yml`（gateway + demo server）
- [ ] 部署到公网（阿里云轻量/vercel 等），拿到**可点击 demo 链接**
- [ ] README 补演示录屏 + 使用说明
- **验收**：面试官点链接能直接看到「Agent 调网关 → 网关调 SQL server」完整闭环（代码部分 2026-08-17 完成：`pytest` 59/59；docker compose 双容器全链路冒烟通过；剩公网部署与录屏）

### 阶段 5：Agent 示例 + 开源推广 + PR（W5–6）
- [ ] `examples/`：写一个 LangChain / OpenAI function-calling 的 Agent 示例，通过网关调用工具
- [ ] README 补「企业级拓展路径」（见第 9 节）
- [ ] 发到 知乎/掘金/v2ex/即刻，README 附博客链接
- [ ] 向 MCP 生态提 1–2 个 PR（modelcontextprotocol/servers、Dify、LangChain 均可）
- **验收**：GitHub 有 star / 讨论，PR 至少提交（合入最佳）

---

## 9. 企业级拓展路径（写进 README，不实现，只体现架构意识）

- **多租户 + RBAC**：租户隔离、细粒度工具权限
- **模型路由**：统一 LLM 入口 + 多模型负载均衡（类比 One-API）
- **审计合规**：全量调用留痕、敏感操作审批流
- **K8s + 自动扩缩容**：无状态网关横向扩展、服务发现
- **链路追踪**：OpenTelemetry 全链路
- **熔断降级 / 缓存**：工具调用异常兜底

面试金句：**「我做的是网关这层，天然的演进方向就是企业内部的 Agent 中台」**。

---

## 10. 简历呈现（做完全部/部分后写进简历）

> 开源项目「MCP 智能体网关」：基于 FastAPI 的统一 MCP 工具网关，支持 stdio/SSE/HTTP 三种传输，聚合多个 MCP Server 供上层 Agent 统一调用；自实现 API Key 鉴权、令牌桶限流与 Prometheus 指标；提供可在线体验 Demo（附链接），GitHub 开源（附地址）。

---

## 11. 当前进度（每次开发后更新此节）

- **状态**：阶段 4 代码部分已完成（metrics + NL2SQL + docker-compose），剩公网部署与演示录屏
- **已完成阶段**：阶段 0（环境准备）、阶段 1（工程骨架 + CI）、阶段 2（协议层打通）、阶段 3（统一 API + 鉴权限流）、阶段 4 代码部分
- **仓库**：https://github.com/NightR71/MCP-Gateway-Demo.git（main 已跟踪 origin/main；阶段 2/3 提交 d43643d、f136de6 已推送）
- **⚠️ 源码丢失与恢复事件（2026-08-17）**：上一会话的阶段 4 源码（db.py / nl2sql.py / server.py 升级版 / 3 个测试文件）神秘丢失，仅剩 `__pycache__` 中的 pyc；本会话已用 marshal 反编译 pyc 提取常量与签名，完整重建全部源码并通过测试。**教训：每个小步骤完成后立即提交 git**
- **阶段 4 产出**：
  - `app/core/metrics.py`：`mcp_gateway_tool_calls_total{tool,server,status}` Counter + `mcp_gateway_tool_call_duration_seconds{tool,server}` Histogram + `record_tool_call()` + `ToolCallTimer`（with 退出自动记录，异常自动记 exception）；`app/mcp/registry.py` 的 `call_tool` 已接入（status: ok/error/exception）
  - `servers/demo_sql_server/db.py`：迷你电商库（customers 5 / products 5 / orders 9 行种子），`init_db` 幂等、`validate_readonly`（SELECT/WITH 单语句 + 危险关键字词边界正则）、`execute_readonly`（max_rows 截断标记）、`get_schema`
  - `servers/demo_sql_server/nl2sql.py`：13 条「正则关键词 → SQL 模板」规则 + `SUPPORTED_EXAMPLES` 兜底示例 + `question_to_sql()`；规则引擎与工具层解耦，未来可换 LLM 实现
  - `servers/demo_sql_server/server.py`：4 工具（echo / ask / run_sql / list_tables），结果渲染为 Markdown 表格；`DEMO_SQL_TRANSPORT=http` 时以 Streamable HTTP 运行（DEMO_SQL_HOST/PORT/DB_PATH 环境变量，默认 0.0.0.0:9001、data/demo_sql.db）
  - Docker：`config/gateway.docker.yaml`（http 传输连 `http://demo_sql:9001/mcp`）+ compose 双服务（demo_sql 带 TCP healthcheck，gateway `depends_on: service_healthy`）
  - 测试：`tests/test_core/test_metrics.py`（差值断言，指标进程级累积）、`tests/test_servers/test_db.py`（7 例）、`tests/test_servers/test_nl2sql.py`（自洽性：示例必命中规则）、`tests/test_mcp/test_http_transport.py`（独立 http 进程集成测试，防 http 分支回归）；更新 test_tools / test_servers / test_registry 的 1 工具旧断言 → 4 工具
- **🐞 阶段 4 修掉的两个存量 bug**：
  1. `app/mcp/transports.py` http 分支按 3 元组解包 `streamable_http_client`，但 **mcp 2.0.0 只 yield (read, write) 两个值**（不再返回 get_session_id）——阶段 2 只测过 stdio，该 bug 潜伏至 docker 冒烟才暴露，已修并补 http 集成测试
  2. `.dockerignore` 阶段 1 排除了 `servers/`，阶段 2 加了 `COPY servers` 却未同步——`docker compose build` 直接失败，已修
- **阶段 4 验证结果**（2026-08-17）：`ruff check` + `ruff format` 通过；`pytest` **59/59**（13.75s）；`docker compose up --build` 双容器全链路：`registry_ready connected=1 tools=4`、无 Key 401、列出 4 工具、`demo_sql__ask`「有多少客户？」经 HTTP 返回 5、`/metrics` 含 `mcp_gateway_tool_calls_total{server="demo_sql",status="ok",tool="demo_sql__ask"} 1.0`；本地 uvicorn stdio 模式冒烟同样通过（「总销售额是多少？」→ 20289.0，与种子数据手算一致）
- **Vercel 部署已备齐（2026-08-17，待用户操作）**：Vercel 官方文档确认 Python 运行时默认 3.12、支持 pyproject+uv.lock、支持 lifespan、自动识别 `app/main.py` 的 `app`——零代码改动。新增 `config/gateway.vercel.yaml`（SQLite 落 /tmp，函数文件系统只读）、`vercel.json`（maxDuration 60s）；**完整步骤与验收清单见 `docs/部署到Vercel.md`**（含 Render/Docker 备选）。需在 Vercel 项目配两个环境变量：`GATEWAY_CONFIG_FILE=config/gateway.vercel.yaml`、`DEMO_SQL_DB_PATH=/tmp/demo_sql.db`
- **下一步动作**：用户按 `docs/部署到Vercel.md` 部署拿 demo 链接 → README 补链接与演示录屏 → 进入阶段 5（examples/ Agent 调用示例、开源推广、向 MCP 生态提 PR）
- **最后更新时间**：2026-08-17
- **Git 身份**：NightR71 / 1553364473@qq.com（已配置）
- **认证**：PAT 已获取并完成认证（credential.helper store 已记住凭据）；`GITHUB_TOKEN` 已通过 setx 写入用户环境变量（重开终端生效）
- **环境**：Python 3.12.11（uv 管理，`.python-version` 已固定 3.12）；uv 下载 GitHub Release 资源需 `UV_NATIVE_TLS=1`（本机证书问题）；Docker Desktop 已配置国内镜像加速器（registry-mirrors，写入 `~/.docker/daemon.json`）；**MCP SDK 实际安装为 2.0.0**（服务端用 `mcp.server.mcpserver.MCPServer`，无旧 fastmcp 模块；http 传输函数名为 `streamable_http_client`）；**本机 360 安全软件会拦截 pytest 拉起子进程（WinError 5）；2026-08-16 起 360 已关闭，测试可正常拉起子进程；若复现 WinError 5 先检查 360 是否又开启**
- **阶段 1 产出**：`pyproject.toml` + `uv.lock`、`app/main.py`（`/health`、`/metrics`）、`app/config.py`（YAML+环境变量配置中心）、`app/core/logging.py`（structlog JSON）、`app/core/metrics.py`、`app/api/deps.py`、`app/api/routes/health.py`、`app/schemas/health.py`、`tests/`（5 用例）、`Dockerfile` + `docker-compose.yml`、`.github/workflows/ci.yml`、README 骨架
- **阶段 2 产出**：`app/mcp/schemas.py`（ToolInfo/ToolCallRequest/ToolCallResult/ServerStatus/UnknownToolError，命名空间分隔符用 `__` 以兼容 OpenAI function calling 命名规则）、`app/mcp/transports.py`（stdio/SSE/Streamable HTTP 统一连接工厂）、`app/mcp/client.py`（MCPClient，AsyncExitStack 管理连接生命周期）、`app/mcp/registry.py`（ToolRegistry：并发连接、单点失败不阻塞、`{server}__{tool}` 命名空间映射）、`servers/demo_sql_server/server.py`（echo 工具）、main.py lifespan 接线（registry 挂 `app.state`）、`config/gateway.yaml` 声明 demo_sql server、Settings 新增 `tool_call_timeout`、Dockerfile 补 COPY servers、`tests/test_mcp/`（registry 真实子进程集成测试 3 例 + schemas 3 例）
- **验证结果**：`ruff check` + `ruff format --check` 通过；`pytest` 12/12 通过（3.92s）；uvicorn 真实启动冒烟：lifespan 自动连上 demo_sql（`registry_ready connected=1 tools=1`），`/health` 正常（2026-08-16）
- **阶段 3 产出**：`app/schemas/auth.py`（APIKeyInfo：key/name/rate_limit_per_minute）、`app/core/security.py`（APIKeyStore Protocol + SQLiteAPIKeyStore：标准库 sqlite3 + asyncio.to_thread + Lock 串行化，启动建表 + YAML 种子 INSERT OR IGNORE 幂等）、`app/core/rate_limit.py`（TokenBucket 令牌桶 + RateLimiter 按 Key 维度管理，纯内存无锁）、`app/config.py`（AuthConfig + `get_auth_config()`，读 YAML `auth` 节）、`app/api/deps.py`（KeyStoreDep/RateLimiterDep/CurrentKeyDep/ProtectedDep 依赖链：先鉴权 401 后限流 429）、`app/api/routes/tools.py`（GET /tools、POST /tools/{name}/call，未知工具 404、下游异常 502）、`app/api/routes/servers.py`（GET /servers）、main.py lifespan 接线（key_store + rate_limiter 挂 app.state）、`config/gateway.yaml` 新增 auth 节（演示 Key `dev-key-please-change`，60 次/分钟）、`tests/fixtures/gateway_test.yaml`（:memory: SQLite + test-key/limited-key）、conftest 新增 `gateway_client`（asgi-lifespan 跑完整 lifespan）、`tests/test_core/`（security 3 例 + rate_limit 4 例）、`tests/test_api/`（tools 6 例 + servers 2 例）；新增 dev 依赖 `asgi-lifespan`
- **阶段 3 验证结果**：`ruff check` 通过；`pytest` 26/26 通过（14.69s）；uvicorn 真实冒烟（须 `uv run uvicorn` 启动使 stdio 子进程拿到 .venv 的 python）：无 Key / 错 Key 均 401，配额 60/分钟下第 61 次请求 429 且带 Retry-After，带 Key 列出 `demo_sql__echo`、调用返回 `echo: smoke`，`/servers` 显示 connected（2026-08-16）

---

## 12. 交接给新对话的说明

新对话接手时，请按顺序执行：
1. 读本文件 `MCP-Gateway项目规划与开发步骤.md`
2. 读仓库 `README.md` 和 `pyproject.toml`，确认当前代码状态
3. 从第 11 节「当前进度」标明的下一步开始开发
4. 每完成一个阶段，更新第 11 节「当前进度」

---

## 13. 环境认证清单（一次性手动步骤，✅ 均已完成 2026-08-15）

环境已全部就绪（Node / Docker / uv / Python 均装好）。
**PAT 已获取并完成认证**——凭据存于本机 git credential store 与 `GITHUB_TOKEN` 用户环境变量；明文不再记录于本文档（防止推送到公网泄露）。

### 创建 PAT（浏览器操作）
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)
2. 勾选 scope：`repo`（全选）、`read:org`、`workflow`
3. 复制生成的 token（形如 `ghp_...`，只显示一次）

### 认证 git push（本机终端执行，需 PAT）
```bash
# 1) 修复失效的 credential helper（原 manager-core 指向不存在的命令）
git config --global credential.helper store
# 2) 首次 push（提示输入用户名 NightR71、密码填 PAT，之后自动记住）
cd D:\New_Project
git push -u origin main
```

### 设置 GITHUB_TOKEN（供 GitHub MCP 用）
```bash
setx GITHUB_TOKEN "ghp_你的token"
# 重开终端 / opencode 后生效
```
