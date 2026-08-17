# 部署到 Vercel — 操作步骤

> 目标：把 MCP Gateway 部署为 Vercel Function，拿到可点击的公网 demo 链接。
> 依据 Vercel 官方文档（2026-07 更新）：Python 运行时默认 3.12、支持 pyproject.toml + uv.lock 装依赖、
> 支持 FastAPI lifespan 启动事件、自动识别 `app/main.py` 里的 `app` 实例——**本仓库已满足全部约定，零代码改动**。

## 仓库侧已备好的文件

| 文件 | 作用 |
|---|---|
| `app/main.py` | Vercel 自动识别的入口（`app/` 内的 `main.py` + 顶层 `app` 变量） |
| `pyproject.toml` + `uv.lock` | 依赖声明，Vercel 直接用它安装 |
| `.python-version` | 固定 Python 3.12（Vercel 默认即 3.12） |
| `config/gateway.vercel.yaml` | Vercel 专用配置：SQLite 路径改到 `/tmp`（函数文件系统只读，/tmp 除外） |
| `vercel.json` | 把函数 maxDuration 提到 60s（冷启动要拉起 MCP 子进程） |

## 部署步骤（约 5 分钟）

1. **推送代码**：确认本仓库最新 main 已 push 到 GitHub。
2. **导入项目**：Vercel Dashboard → Add New → Project → 选 `MCP-Gateway-Demo` 仓库。
   - Framework Preset 会自动识别为 **FastAPI**，无需手选；Root Directory 保持仓库根目录。
3. **配环境变量**（Settings → Environment Variables，或导入时展开 Environment Variables）：

   | Name | Value | 说明 |
   |---|---|---|
   | `GATEWAY_CONFIG_FILE` | `config/gateway.vercel.yaml` | 指向 Vercel 专用配置（SQLite 落 /tmp） |
   | `DEMO_SQL_DB_PATH` | `/tmp/demo_sql.db` | demo server 子进程的 SQLite 路径（继承自父进程环境） |

4. **Deploy**，等构建完成，拿到 `https://<项目名>.vercel.app`。

## 部署后验收清单

```bash
# 1. 健康检查（无需 Key）
curl https://<项目名>.vercel.app/health

# 2. 无 Key 应 401
curl -i https://<项目名>.vercel.app/tools

# 3. 列工具（应见 4 个 demo_sql__* 工具）
curl -H "X-API-Key: dev-key-please-change" https://<项目名>.vercel.app/tools

# 4. NL2SQL 完整闭环（面试官要看的就是这个）
curl -X POST https://<项目名>.vercel.app/tools/demo_sql__ask/call \
  -H "X-API-Key: dev-key-please-change" -H "Content-Type: application/json" \
  -d '{"arguments": {"question": "有多少客户？"}}'
```

> 首次请求是冷启动：lifespan 要拉起 demo server 子进程 + MCP 握手，等 3–10 秒属正常。

## 已知注意事项（如实告知面试官也可）

- **冷启动**：Serverless 实例冷启动时才连 MCP Server（lifespan），首请求略慢；热实例毫秒级。
- **状态为实例级**：限流令牌桶、Prometheus 指标、/tmp 下的 SQLite 都是单实例内存/本地态，多实例不共享——MVP 演示无影响，企业生产应换外部存储（这正是 README「企业级拓展路径」里讲的）。
- **演示 Key 是公开的**：`dev-key-please-change` 就在仓库里，任何拿到链接的人都能调（60 次/分钟限流兜底）。想换 Key：改 `config/gateway.vercel.yaml` 的 key 值，但别把新 Key 提交进公开仓库。
- **构建源走的是阿里云 PyPI 镜像**（pyproject.toml 里的 `[[tool.uv.index]]`）：Vercel 海外构建机访问阿里云可能偏慢。若构建卡在装依赖，临时删掉该 index 段再 push 即可。
- **若 stdio 子进程在 Vercel 环境受限**（小概率，Lambda 系环境一般允许拉起 python 子进程）：备选方案是用仓库里已验证过的 `Dockerfile` 部署到 Render / Railway（Docker 运行时，无 Serverless 限制），步骤见下节。

## 备选：Render（Docker，最稳）

本仓库 `Dockerfile` 已经过 `docker compose up --build` 全链路验证：

1. Render → New → Web Service → 连接 GitHub 仓库。
2. Runtime 选 **Docker**（自动用根目录 `Dockerfile`），实例选 Free。
3. 环境变量不用配（默认 `config/gateway.yaml`，stdio 拉起 demo server，同容器）。
4. 部署后同样跑上面「验收清单」的 4 条 curl。

> docker-compose.yml 的双容器形态（gateway + 独立 demo_sql 容器）适合云服务器/自有 K8s；
> Render 单容器形态用默认 stdio 配置即可，无需 compose。
