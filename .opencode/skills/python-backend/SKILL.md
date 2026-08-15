---
name: python-backend
description: Use when writing or reviewing Python backend code in this project (FastAPI, asyncio, pydantic-settings, pytest, ruff). Covers project layering, async rules, config, testing, and lint conventions.
---

# Python 后端开发约定

## 分层（严格）

- `app/api` 接口层：路由只做参数解析 + 调用 + 响应组装，业务不放这里
- `app/mcp` 协议层：所有 MCP 协议细节只在这里
- `app/core` 横切：security / rate_limit / logging / metrics
- `app/schemas`：Pydantic 模型

## 异步

- 全部 `async def`；IO 用 asyncio，禁止阻塞事件循环
- 数据库/网络调用统一 async，不用 requests/sync 客户端

## 配置

- 用 pydantic-settings，禁止写死常量；来源：环境变量 + YAML
- 所有可变项（host/port/key/配额）进 `app/config.py`

## 测试

- pytest + pytest-asyncio + httpx（AsyncClient）
- 测试文件放 `tests/`，与 `app/` 结构一一对应
- 每个接口至少覆盖：成功 / 401 未授权 / 429 限流

## 代码风格

- 完整类型注解
- 提交前跑 `uv run ruff check .` 和 `uv run pytest`

## 命令

- `uv sync` / `uv run uvicorn app.main:app --reload` / `uv run pytest`
- `uv run ruff check .` / `uv run ruff format .`
