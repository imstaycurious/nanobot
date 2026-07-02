# 06 - Agent Loop：nanobot 的心脏

> **阶段一核心循环 - 第 1 篇**  
> 代码位置：`nanobot/agent/loop.py`

---

## 一句话总结

**AgentLoop 是 nanobot 的核心调度器**：它从 MessageBus 接收消息，管理 Session 生命周期，构建上下文（context），然后把实际的 LLM 对话循环委托给 AgentRunner 执行。

---

## 核心职责

AgentLoop 做三件事：

1. **接收消息** — 从 `MessageBus.inbound` 队列消费消息
2. **管理 Session** — 每个会话（session）都有独立的历史记录（history），需要持久化、压缩、恢复
3. **构建上下文** — 把用户消息 + 历史 + 技能（skills）+ 模板组装成 LLM 可以理解的 `messages` 列表

**它不负责调用 LLM**。调用 LLM 的工作由 `AgentRunner` 完成。

---

## 数据流：消息怎么被处理

```
┌─────────────┐
│  Channel    │  (Telegram / Discord / WebUI)
│  收到消息    │
└──────┬──────┘
       │
       ↓  publish_inbound(InboundMessage)
┌─────────────────┐
│   MessageBus    │
│  .inbound 队列  │
└─────────┬───────┘
          │
          ↓  consume_inbound()
┌──────────────────────────────────────┐
│        AgentLoop.run()               │
│  ┌────────────────────────────────┐  │
│  │  1. Session 恢复/创建          │  │
│  │     - get_or_create(key)       │  │
│  │     - 从 memory/history.jsonl  │  │
│  │       加载历史                  │  │
│  └────────────────────────────────┘  │
│                 ↓                     │
│  ┌────────────────────────────────┐  │
│  │  2. Context 构建               │  │
│  │     - ContextBuilder           │  │
│  │     - 历史 + 用户消息 + 技能   │  │
│  │     → messages 列表            │  │
│  └────────────────────────────────┘  │
│                 ↓                     │
│  ┌────────────────────────────────┐  │
│  │  3. 委托给 AgentRunner         │  │
│  │     runner.run(messages, ...)  │  │
│  │     → 返回 final_content       │  │
│  └────────────────────────────────┘  │
│                 ↓                     │
│  ┌────────────────────────────────┐  │
│  │  4. 保存结果到 Session         │  │
│  │     session.add_message(...)   │  │
│  │     sessions.save(session)     │  │
│  └────────────────────────────────┘  │
└──────────────┬───────────────────────┘
               │
               ↓  publish_outbound(OutboundMessage)
       ┌───────────────┐
       │  MessageBus   │
       │ .outbound 队列│
       └───────┬───────┘
               │
               ↓  consume_outbound()
       ┌───────────────┐
       │   Channel     │
       │  发送给用户   │
       └───────────────┘
```

---

## 关键概念

### 1. Session：会话的持久化容器

每个对话都有一个 **session_key**（例如 `telegram:123456789`），它对应一个 Session 对象：

```python
# nanobot/session/manager.py
class Session:
    key: str                      # 唯一标识
    messages: list[dict]          # 历史消息
    metadata: dict                # 会话元数据
    created_at: datetime
    updated_at: datetime
```

**Session 存在哪里？**

```
~/.nanobot/memory/
  telegram:123456789/
    history.jsonl        # 每行一个 turn（user + assistant）
  discord:987654321/
    history.jsonl
```

**Session 什么时候创建？**

```python
# loop.py:1461
ctx.session = self.sessions.get_or_create(ctx.session_key)
```

第一次收到某个 chat 的消息时，`SessionManager` 会：
1. 检查 `memory/{session_key}/history.jsonl` 是否存在
2. 不存在 → 创建新 Session
3. 存在 → 从 `.jsonl` 加载历史

---

### 2. 状态机：TurnState

AgentLoop 使用**状态机**处理每个 turn（一轮对话）：

```python
class TurnState(Enum):
    RESTORE = auto()   # 恢复 checkpoint / 提取文档
    COMPACT = auto()   # 压缩历史（如果太长）
    COMMAND = auto()   # 检查是否是命令（/new, /stop）
    BUILD   = auto()   # 构建上下文（history + skills + 用户消息）
    RUN     = auto()   # 运行 AgentRunner（调用 LLM + 工具）
    SAVE    = auto()   # 保存结果到 Session
    RESPOND = auto()   # 组装 OutboundMessage
    DONE    = auto()   # 结束
```

**为什么要状态机？**

- 每个阶段独立、可测试
- 错误时可以精确定位卡在哪个状态
- 方便插入新逻辑（比如在 BUILD 之前做额外检查）

**状态转换表**：

```python
_TRANSITIONS = {
    (TurnState.RESTORE, "ok"): TurnState.COMPACT,
    (TurnState.COMPACT, "ok"): TurnState.COMMAND,
    (TurnState.COMMAND, "dispatch"): TurnState.BUILD,  # 不是命令，继续
    (TurnState.COMMAND, "shortcut"): TurnState.DONE,   # 是命令，直接结束
    (TurnState.BUILD, "ok"): TurnState.RUN,
    (TurnState.RUN, "ok"): TurnState.SAVE,
    (TurnState.SAVE, "ok"): TurnState.RESPOND,
    (TurnState.RESPOND, "ok"): TurnState.DONE,
}
```

每个状态处理函数返回一个事件字符串（例如 `"ok"`），状态机根据转换表决定下一个状态。

---

### 3. ContextBuilder：组装 LLM 输入

`ContextBuilder` 负责把历史消息、用户当前消息、技能（skills）、模板组装成 LLM 能理解的格式。

**输入：**

- `history: list[dict]` — Session 的历史消息
- `current_message: str` — 用户这次说的话
- `media: list[str]` — 图片/文件路径
- `session_metadata: dict` — 会话元数据
- `runtime_state: AgentLoop` — 运行时状态（用于技能渲染）

**输出：**

```python
[
    {"role": "system", "content": "You are a helpful assistant..."},
    {"role": "user", "content": "历史消息1"},
    {"role": "assistant", "content": "回复1"},
    {"role": "user", "content": "当前消息 + 技能注入 + 运行时状态"},
]
```

**关键代码：**

```python
# loop.py:1540
ctx.initial_messages = self._build_initial_messages(
    ctx.msg,
    ctx.session,
    ctx.history,
    ctx.pending_summary,
)

# loop.py:663
def _build_initial_messages(self, msg, session, history, pending_summary):
    return self.context.build_messages(
        history=history,
        current_message=msg.content,
        media=msg.media,
        channel=msg.channel,
        chat_id=msg.chat_id,
        session_summary=pending_summary,  # 压缩后的摘要
        workspace=self.workspace,
        runtime_state=self,               # 传给技能渲染
    )
```

---

## 并发控制：一个 Session 串行，多个 Session 并行

**问题**：如果同一个用户快速发 3 条消息，会发生什么？

**解决方案**：每个 Session 有一把锁（`asyncio.Lock`）：

```python
# loop.py:1033
lock = self._session_locks.setdefault(session_key, asyncio.Lock())

async with lock:
    # 处理消息
```

**效果**：

- 同一个 `session_key` 的消息**串行**处理（保证历史顺序正确）
- 不同 `session_key` 的消息**并行**处理（不同用户互不阻塞）

**全局并发限制**：

```python
# loop.py:341
_max = int(os.environ.get("NANOBOT_MAX_CONCURRENT_REQUESTS", "3"))
self._concurrency_gate = asyncio.Semaphore(_max) if _max > 0 else None

# loop.py:1034
async with lock, gate:
    # ...
```

默认最多同时处理 3 个会话（跨不同 session）。

---

## 中断注入（Mid-Turn Injection）

**问题**：用户发了一条消息，Agent 正在调用工具（比如运行 shell 命令 10 秒），用户又发了第二条消息，怎么办？

**传统做法**：第二条消息等第一条完成后再处理。

**nanobot 的做法**：把第二条消息**注入到第一条消息的循环中**。

**实现机制**：

1. 每个 session 有一个 `pending_queue`（`asyncio.Queue`）
2. 当 session 正在处理时，新消息不创建新任务，而是放入队列：

```python
# loop.py:986
if effective_key in self._pending_queues:
    try:
        self._pending_queues[effective_key].put_nowait(pending_msg)
    except asyncio.QueueFull:
        logger.warning("Pending queue full for session {}", effective_key)
```

3. `AgentRunner` 在工具执行后会调用 `injection_callback`，从队列取消息：

```python
# loop.py:780
async def _drain_pending(*, limit=3):
    if pending_queue is None:
        return []
    items = []
    while len(items) < limit:
        try:
            items.append(pending_queue.get_nowait())
        except asyncio.QueueEmpty:
            break
    return items
```

4. 如果有新消息，`AgentRunner` 会把它们追加到 `messages`，继续下一轮 LLM 调用。

**好处**：

- 用户不用等——新消息立刻被处理
- 上下文连续——第二条消息能看到第一条消息的工具结果

---

## Checkpoint：任务中断时的救命稻草

**问题**：用户发了 `/stop`，或者进程崩溃，怎么保留部分进度？

**解决方案**：`_set_runtime_checkpoint`

每次工具执行完成后，AgentRunner 会调用 `checkpoint_callback`：

```python
# loop.py:775
async def _checkpoint(payload: dict[str, Any]) -> None:
    if session is None:
        return
    self._set_runtime_checkpoint(session, payload)

# loop.py:1785
def _set_runtime_checkpoint(self, session: Session, payload: dict[str, Any]):
    session.metadata[self._RUNTIME_CHECKPOINT_KEY] = payload
    self.sessions.save(session)
```

**Checkpoint 内容**：

```python
{
    "phase": "tools_completed",
    "iteration": 2,
    "model": "claude-opus-4",
    "assistant_message": {...},           # LLM 返回的消息
    "completed_tool_results": [...],      # 已完成的工具结果
    "pending_tool_calls": [],             # 未完成的工具
}
```

**恢复时机**：

```python
# loop.py:1465
if self._restore_runtime_checkpoint(ctx.session):
    self.sessions.save(ctx.session)
```

下次收到消息时，AgentLoop 检查 `session.metadata` 有没有 checkpoint：
- 有 → 把 `assistant_message` 和 `completed_tool_results` 追加到 `session.messages`
- 没有 → 正常流程

**效果**：用户发 `/stop` 后，已经执行的工具结果不会丢失，下次对话能继续。

---

## 完整流程代码追踪

以一个真实场景为例：用户在 Telegram 发送 `"帮我创建一个 hello.py"`

### Step 1: Channel 发布消息

```python
# nanobot/channels/telegram.py (简化)
await self.bus.publish_inbound(InboundMessage(
    channel="telegram",
    sender_id="123456789",
    chat_id="123456789",
    content="帮我创建一个 hello.py",
))
```

### Step 2: AgentLoop 消费消息

```python
# loop.py:940
msg = await self.bus.consume_inbound()
```

### Step 3: 创建异步任务

```python
# loop.py:1016
task = asyncio.create_task(self._dispatch(msg))
self._active_tasks.setdefault(effective_key, []).append(task)
```

### Step 4: _dispatch 获取锁，启动状态机

```python
# loop.py:1030
async def _dispatch(self, msg: InboundMessage):
    session_key = self._effective_session_key(msg)
    lock = self._session_locks.setdefault(session_key, asyncio.Lock())
    
    async with lock:
        pending = asyncio.Queue(maxsize=20)
        self._pending_queues[session_key] = pending
        
        response = await self._process_message(msg, pending_queue=pending)
        await self.bus.publish_outbound(response)
```

### Step 5: _process_message 状态机

```python
# loop.py:1356
while ctx.state is not TurnState.DONE:
    handler_name = f"_state_{ctx.state.name.lower()}"
    handler = getattr(self, handler_name)
    
    event = await handler(ctx)  # 调用状态处理函数
    
    next_state = self._TRANSITIONS.get((ctx.state, event))
    ctx.state = next_state
```

#### 状态 1: RESTORE

```python
# loop.py:1446
async def _state_restore(self, ctx: TurnContext) -> str:
    ctx.session = self.sessions.get_or_create(ctx.session_key)
    
    # 恢复 checkpoint（如果有）
    if self._restore_runtime_checkpoint(ctx.session):
        self.sessions.save(ctx.session)
    
    return "ok"  # → COMPACT
```

#### 状态 2: COMPACT

```python
# loop.py:1482
async def _state_compact(self, ctx: TurnContext) -> str:
    ctx.session, pending = self.auto_compact.prepare_session(
        ctx.session, ctx.session_key
    )
    ctx.pending_summary = pending  # 压缩后的摘要
    return "ok"  # → COMMAND
```

#### 状态 3: COMMAND

```python
# loop.py:1487
async def _state_command(self, ctx: TurnContext) -> str:
    raw = ctx.msg.content.strip()
    result = await self.commands.dispatch(cmd_ctx)
    
    if result is not None:  # 是命令（/new, /stop）
        ctx.outbound = result
        return "shortcut"  # → DONE
    
    return "dispatch"  # 不是命令 → BUILD
```

#### 状态 4: BUILD

```python
# loop.py:1512
async def _state_build(self, ctx: TurnContext) -> str:
    # 从 Session 获取历史
    ctx.history = ctx.session.get_history(
        max_messages=self._max_messages,
        max_tokens=self._replay_token_budget(),
    )
    
    # 构建 LLM 输入
    ctx.initial_messages = self._build_initial_messages(
        ctx.msg, ctx.session, ctx.history, ctx.pending_summary
    )
    
    # 早期持久化用户消息（防止崩溃丢失）
    ctx.user_persisted_early = self._persist_user_message_early(
        ctx.msg, ctx.session
    )
    
    return "ok"  # → RUN
```

#### 状态 5: RUN

```python
# loop.py:1558
async def _state_run(self, ctx: TurnContext) -> str:
    result = await self._run_agent_loop(
        ctx.initial_messages,
        session=ctx.session,
        pending_queue=ctx.pending_queue,  # 中断注入队列
    )
    
    final_content, tools_used, all_msgs, stop_reason, had_injections = result
    ctx.final_content = final_content
    ctx.all_messages = all_msgs
    
    return "ok"  # → SAVE
```

#### 状态 6: SAVE

```python
# loop.py:1594
async def _state_save(self, ctx: TurnContext) -> str:
    # 保存 turn（跳过 initial_messages 中的历史部分）
    self._save_turn(ctx.session, ctx.all_messages, ctx.save_skip)
    
    # 清理 checkpoint
    self._clear_runtime_checkpoint(ctx.session)
    self.sessions.save(ctx.session)
    
    return "ok"  # → RESPOND
```

#### 状态 7: RESPOND

```python
# loop.py:1633
async def _state_respond(self, ctx: TurnContext) -> str:
    ctx.outbound = self._assemble_outbound(
        ctx.msg, ctx.final_content, ctx.all_messages, ctx.stop_reason
    )
    return "ok"  # → DONE
```

### Step 6: 发布响应

```python
# loop.py:1087
await self.bus.publish_outbound(response)
```

### Step 7: Channel 消费响应

```python
# nanobot/channels/telegram.py (简化)
msg = await self.bus.consume_outbound()
await bot.send_message(chat_id=msg.chat_id, text=msg.content)
```

---

## 重要设计决策

### 为什么要把 AgentLoop 和 AgentRunner 分开？

**AgentLoop（loop.py）**：
- 产品层关注点：Session、历史、上下文、持久化、命令
- 管理资源：锁、队列、MCP 连接、后台任务

**AgentRunner（runner.py）**：
- 算法层关注点：LLM 对话循环、工具执行、流式响应、重试
- 无状态：不关心 Session 是什么，只负责执行

**好处**：
1. **复用性**：AgentRunner 可以被 SubagentManager 复用（子 Agent 不需要 Session）
2. **测试性**：可以单独测试 AgentRunner 的工具执行逻辑
3. **清晰度**：职责分离，代码更易维护

---

## 常见坑

### 1. Session Key 冲突

```python
# 错误：不同 channel 可能有相同的 chat_id
session_key = msg.chat_id  # ❌

# 正确：加上 channel 前缀
session_key = f"{msg.channel}:{msg.chat_id}"  # ✅
```

### 2. 忘记保存 Session

```python
session.add_message("user", "hello")
# 忘记调用 sessions.save(session) ❌
# 进程崩溃后消息丢失
```

### 3. 历史太长导致上下文溢出

```python
# 错误：加载所有历史
history = session.get_history()  # ❌

# 正确：限制消息数和 token 数
history = session.get_history(
    max_messages=50,
    max_tokens=100000,
)  # ✅
```

---

## 总结

| 问题 | AgentLoop 的答案 |
|------|------------------|
| **消息从哪来？** | MessageBus.inbound 队列 |
| **历史存在哪？** | `~/.nanobot/memory/{session_key}/history.jsonl` |
| **上下文怎么构建？** | ContextBuilder.build_messages() |
| **谁调用 LLM？** | AgentRunner（AgentLoop 只负责协调） |
| **并发怎么控制？** | 每个 Session 一把锁，全局 Semaphore 限流 |
| **中断怎么恢复？** | Checkpoint 机制，保存在 session.metadata |
| **中途消息怎么处理？** | Pending Queue + Injection Callback |

**核心思想**：AgentLoop 是一个有状态的协调器，它管理 Session 生命周期和上下文，但把"和 LLM 对话"这个核心逻辑委托给无状态的 AgentRunner。

---

## 下一篇

[`07-agent-runner.md`](07-agent-runner.md) — AgentRunner 执行流程：Tool Call 循环、流式处理、错误恢复
