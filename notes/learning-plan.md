# nanobot 学习计划

## 已完成 ✅

- [x] **打包与构建系统**
  - pyproject.toml 配置解析
  - Hatch 构建流程
  - hatch_build.py 钩子机制
  - install.sh 安装脚本
  - Python 打包工具链（pip、uv、pipx）
  
- [x] **安全边界**
  - 工作区路径限制（Workspace Restriction）
  - SSRF 防护
  - Shell 沙箱（bwrap）
  
- [x] **项目规范文档研读**
  - AGENTS.md 架构概览
  - .agent/ 开发约束（design、security、gotchas）
  - CONTRIBUTING.md 贡献流程

---

## 学习路线图

```
第一阶段：核心循环 ⭐ 最重要
├── 1. Agent Loop (loop.py)
│   └── 消息接收 → Session 管理 → Context 构建 → 协调 turn
│
├── 2. Agent Runner (runner.py)
│   └── LLM 对话循环 → Tool Call → Tool 执行 → 流式响应
│
└── 3. 完整数据流
    └── InboundMessage → AgentLoop → AgentRunner → Tools → OutboundMessage

第二阶段：工具系统
├── 4. Tool 注册与发现
│   ├── pkgutil 自动扫描
│   ├── entry-points 插件机制
│   └── ToolRegistry
│
├── 5. 核心 Tools 实现
│   ├── Filesystem (read/write/edit/list)
│   ├── Shell (exec)
│   ├── Web (search/fetch)
│   └── MCP (server 连接)
│
└── 6. 安全实施
    └── 路径限制、SSRF、沙箱在代码中的实现

第三阶段：LLM 对接
├── 7. Provider 基础架构
│   ├── base.py 统一接口
│   ├── LLMResponse / ToolCallRequest 数据结构
│   └── 流式处理
│
├── 8. Provider 实现
│   ├── Anthropic
│   ├── OpenAI (含 Responses API)
│   ├── Bedrock
│   └── factory + registry
│
└── 9. 模型切换与 fallback
    └── ModelPreset / InlineFallback

第四阶段：消息与记忆
├── 10. MessageBus
│   └── 异步队列解耦 Channel 和 Agent
│
├── 11. Session 管理
│   ├── SessionManager
│   ├── 上下文压缩
│   └── TTL 自动压缩
│
├── 12. Memory 系统
│   ├── MemoryStore (history.jsonl)
│   ├── Dream 记忆整合
│   └── 原子写入
│
└── 13. Context 构建
    └── 模板、技能、历史重放

第五阶段：多渠道接入
├── 14. Channel 基础
│   ├── BaseChannel 接口
│   ├── 自动发现机制
│   └── ChannelManager
│
├── 15. 典型 Channel 实现
│   ├── WebSocket (WebUI)
│   ├── Telegram
│   └── Discord / Slack
│
└── 16. Pairing 机制
    └── DM sender 批准

第六阶段：前后端通信
├── 17. WebUI Gateway
│   ├── WebSocket 多路复用协议
│   ├── HTTP API (OpenAI-compatible)
│   └── gateway_services.py
│
├── 18. 前端架构（可选）
│   ├── React + TypeScript
│   ├── WebSocket 客户端
│   └── 组件结构
│
└── 19. SDK
    └── Python SDK (nanobot.py)
```

---

## 详细学习顺序

### 阶段一：核心循环（3 篇笔记）⭐

理解 nanobot 的心脏——消息怎么进来、怎么调用 LLM、怎么执行工具、怎么返回。

```
用户消息 → Channel → MessageBus.inbound → AgentLoop.process_message
                                              ↓
                                    Session / Context 构建
                                              ↓
                                    AgentRunner.run()
                                              ↓
                    ┌─────────────────────────┴─────────────────────────┐
                    ↓                                                     ↓
              LLM 返回 text                                      LLM 返回 tool_calls
                    ↓                                                     ↓
              流式输出给用户                                      ToolRegistry.execute()
                                                                          ↓
                                                              Tool 结果追加到 messages
                                                                          ↓
                                                              继续调用 LLM（循环）
                                                                          ↓
                                                              最终文本返回
                                                                          ↓
                                                        MessageBus.outbound → Channel → 用户
```

**笔记输出：**
- `06-agent-loop.md` — AgentLoop 职责、Session 管理、Context 构建
- `07-agent-runner.md` — AgentRunner 执行流程、Tool Call 循环、流式处理
- `08-data-flow.md` — 完整数据流图 + 各模块协作关系

---

### 阶段二：工具系统（3 篇笔记）

理解 Agent 的能力来源——工具怎么注册、怎么被 LLM 调用、安全边界怎么实施。

```
启动时：
  ToolLoader.discover()
    ↓  pkgutil.iter_modules(nanobot.agent.tools)
    ↓  + entry-points "nanobot.tools"
    ↓
  找到所有 Tool 类
    ↓
  ToolRegistry.register()

运行时：
  LLM 返回 tool_call
    ↓
  ToolRegistry.get(tool_name)
    ↓
  Tool.execute(arguments)
    ↓  安全检查（workspace、SSRF、sandbox）
    ↓
  返回结果给 AgentRunner
```

**笔记输出：**
- `09-tool-system.md` — 注册机制、pkgutil 扫描、entry-points
- `10-core-tools.md` — filesystem、shell、web、MCP 实现
- `11-security-implementation.md` — 安全边界在工具代码中的落地

---

### 阶段三：LLM 对接（3 篇笔记）

理解多模型支持——统一接口、不同 Provider 实现、fallback 机制。

```
AgentRunner 需要调用 LLM
    ↓
ProviderFactory.create(model_name)
    ↓  从 registry 找到对应 Provider 类
    ↓
AnthropicProvider / OpenAIProvider / ...
    ↓
provider.send_messages(messages, tools)
    ↓  转换成各家 API 格式
    ↓  处理流式响应
    ↓
返回 LLMResponse
    ↓  text / tool_calls / reasoning
    ↓
AgentRunner 继续处理
```

**笔记输出：**
- `12-provider-architecture.md` — base.py、LLMResponse、ToolCallRequest
- `13-provider-implementations.md` — Anthropic、OpenAI、Bedrock 实现对比
- `14-model-switching.md` — ModelPreset、fallback 机制

---

### 阶段四：消息与记忆（4 篇笔记）

理解会话持久化、上下文压缩、Dream 记忆整合。

```
Session 生命周期：
  SessionManager.get_session(key)
    ↓  从 memory/ 目录读 history.jsonl
    ↓
  Session.messages (内存中)
    ↓
  AgentLoop 添加新 turn
    ↓
  MemoryStore.append_turn() (原子写入)
    ↓  temp file + fsync + rename
    ↓
  Session 过大？→ autocompact
    ↓
  定期 Dream 整合
```

**笔记输出：**
- `15-message-bus.md` — 异步队列、InboundMessage、OutboundMessage
- `16-session-management.md` — SessionManager、上下文压缩、TTL
- `17-memory-system.md` — MemoryStore、history.jsonl、原子写入
- `18-dream-consolidation.md` — Dream 两阶段记忆整合

---

### 阶段五：多渠道接入（3 篇笔记）

理解 15+ 平台怎么统一接入。

```
Channel 启动：
  ChannelManager.start_all()
    ↓  pkgutil 扫描 nanobot.channels
    ↓  + entry-points "nanobot.channels"
    ↓
  TelegramChannel.start()
    ↓  监听 Telegram API
    ↓  收到消息 → InboundMessage
    ↓  MessageBus.publish_inbound()
    ↓
  AgentLoop 消费并处理
    ↓
  OutboundMessage → MessageBus.outbound
    ↓
  TelegramChannel 消费并发送
```

**笔记输出：**
- `19-channel-architecture.md` — BaseChannel、自动发现、ChannelManager
- `20-typical-channels.md` — WebSocket、Telegram、Discord 实现
- `21-pairing-system.md` — DM sender 批准机制

---

### 阶段六：前后端通信（3 篇笔记，可选）

理解 WebUI 怎么和后端通信。

```
前端（浏览器）
    ↓  WebSocket
Gateway (webui/gateway_services.py)
    ↓  多路复用协议
WebSocketChannel
    ↓  MessageBus
AgentLoop
```

**笔记输出：**
- `22-webui-gateway.md` — WebSocket 协议、HTTP API、gateway 服务
- `23-frontend-architecture.md` — React 组件、WebSocket 客户端
- `24-sdk.md` — Python SDK (Nanobot 类)

---

## 每阶段的输出物

每完成一个阶段，产出：

1. **笔记** — 写入 `notes/` 目录，包含：
   - 架构图（ASCII 或文字描述）
   - 关键类/函数说明
   - 数据流图
   - 设计决策分析

2. **代码注释** — 在关键代码处添加理解性注释（可选）

3. **总结** — 回答 3 个问题：
   - 这个模块的核心职责是什么？
   - 它怎么和其他模块协作？
   - 有什么值得借鉴的设计？

---

## 当前进度

```
✅ 已完成：打包构建、安全边界、项目规范
🎯 下一步：阶段一 - Agent 核心循环（loop.py + runner.py）
```

准备好了吗？我们从 `06-agent-loop.md` 开始！
