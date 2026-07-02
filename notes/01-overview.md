# nanobot 项目概览

**nanobot** 是一个轻量级开源 AI Agent 框架，用 Python + React/TypeScript 构建，核心是一个异步 Agent 循环：接收消息 → 调用 LLM → 执行工具 → 管理会话记忆。

## 核心数据流

```
外部平台消息 → Channel → MessageBus → AgentLoop → AgentRunner → LLM Provider
                                    ↑                                    ↓
                              Session/Memory                        Tool 执行
                                    ↑                                    ↓
                              Context 构建                        OutboundMessage → Channel → 用户
```

## 关键模块

| 模块 | 路径 | 职责 |
|------|------|------|
| **AgentLoop** | `nanobot/agent/loop.py` | 核心处理引擎，管理 session、hooks、上下文构建 |
| **AgentRunner** | `nanobot/agent/runner.py` | 执行多轮 LLM 对话循环（发送消息→接收工具调用→执行工具→流式响应） |
| **MessageBus** | `nanobot/bus/queue.py` | 异步消息队列，解耦 Channel 和 Agent |
| **LLM Provider** | `nanobot/providers/` | 统一的 LLM 接口（Anthropic、OpenAI、Azure、Bedrock 等） |
| **Channel** | `nanobot/channels/` | 平台集成（Telegram、Discord、Slack、飞书、微信、QQ 等 15+） |
| **Tool** | `nanobot/agent/tools/` | Agent 能力（文件读写、Shell 执行、Web 搜索、MCP、Cron、子 Agent 等） |
| **Memory** | `nanobot/agent/memory.py` | 会话持久化 + Dream 两阶段记忆整合 |
| **Session** | `nanobot/session/` | 会话管理、上下文压缩、TTL 自动压缩 |
| **Config** | `nanobot/config/schema.py` | Pydantic 配置，从 `~/.nanobot/config.json` 加载 |
| **WebUI** | `webui/` | Vite + React SPA，通过 WebSocket 与 Gateway 通信 |
| **API Server** | `nanobot/api/server.py` | OpenAI 兼容 HTTP API |
| **SDK** | `nanobot/nanobot.py` | Python SDK 入口（`Nanobot` 类） |

## 入口点

- **CLI**: `nanobot` 命令 → `nanobot/cli/commands.py`（基于 Typer）
- **Python SDK**: `Nanobot.from_config()` → `nanobot/nanobot.py`
- **Gateway**: `nanobot gateway` → 启动 WebUI + WebSocket + HTTP API

## 技术栈

- **后端**: Python 3.11+，asyncio，Pydantic，loguru，httpx
- **前端**: React + TypeScript + Vite + Bun
- **构建**: Hatch（Python），Bun（前端）
- **测试**: pytest（asyncio_mode=auto），覆盖率要求 75%+
- **代码规范**: ruff（E/F/I/N/W），行宽 100

## 项目规模

- Python 源文件约 80+ 个
- WebUI 组件约 60+ 个 TypeScript/TSX 文件
- 测试文件 40+ 个
- 支持 15+ 聊天平台、10+ LLM Provider
