---
name: mcp-dev
description: Use when writing MCP servers, MCP clients, tool registry, or transport code. Covers MCP Python SDK usage, the three transports (stdio/sse/streamable http), and tool registration conventions.
---

# MCP 开发约定

## SDK

- 用官方 `mcp` Python SDK（`modelcontextprotocol/python-sdk`）
- 查实现细节用 reference `mcp-sdk`，别凭记忆编 API

## 三种传输

- stdio：本地子进程，靠 command + args 启动
- SSE：HTTP 长连接，用于远程 server
- Streamable HTTP：现代推荐，优先用于自建 server

### mcp 2.0.0 实测要点（踩过坑，别再凭记忆）

- 服务端：`mcp.server.mcpserver.MCPServer`（无旧 fastmcp 模块）
- 客户端 http 传输函数：`mcp.client.streamable_http.streamable_http_client`
- **该函数只 yield `(read, write)` 两个值**（1.x 旧版会多 yield get_session_id；本项目曾因按 3 元组解包导致 http 传输全挂，日常只测 stdio 时不会暴露）
- `MCPServer.run(transport="streamable-http", host=..., port=...)`：host/port 经 kwargs 透传 uvicorn，服务默认挂载路径 `/mcp`
- 每个传输分支都要有集成测试覆盖（见 tests/test_mcp/test_http_transport.py）

## 工具注册（核心在 `app/mcp/registry.py`）

- 读取 config YAML 中声明的 server，建立连接，聚合所有工具
- 每个工具：name + description + input schema（Pydantic）
- tool 名全局唯一，跨 server 用 namespace 前缀避免冲突
- registry 维护「工具名 → server」的映射，供路由调用时定位

## 约定

- MCP Server 声明走 config YAML，不写死
- 连接生命周期要管理（FastAPI startup 建连接 / shutdown 清理）
- 工具调用失败要捕获异常并转成统一错误响应
