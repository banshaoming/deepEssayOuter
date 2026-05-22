# deepEssay 架构简图

## 整体数据流

```
┌──────────────────────────────────────────────────────────────────────┐
│                            Channels                                 │
│  ┌──────────┐              ┌─────────────────┐                      │
│  │   REPL   │              │  Web Gateway    │                      │
│  │(Terminal)│              │ (Tauri IPC+SSE) │                      │
│  └────┬─────┘              └────────┬────────┘                      │
│       └────────────┬───────────────┘                                │
│                    │                                                │
│          ┌─────────▼─────────┐                                      │
│          │    Agent Loop     │  SubmissionParser                    │
│          │  (handle_message) │  Intent routing                     │
│          └────────┬──────────┘                                      │
│                   │                                                 │
│          ┌────────▼────────┐                                       │
│          │Session Manager  │                                       │
│          │ Sessions/Threads│                                       │
│          └────────┬────────┘                                       │
│                   │                                                 │
│          ┌────────▼──────────────────────────┐                     │
│          │         Agentic Loop              │                     │
│          │  ┌─────────────────────────────┐  │                     │
│          │  │ Reasoning ─── LLM Provider  │  │  ← Skills 注入      │
│          │  │   (respond_with_tools)      │  │    上下文           │
│          │  └─────────────┬───────────────┘  │                     │
│          │                │                  │                     │
│          │  ┌─────────────▼───────────────┐  │                     │
│          │  │     Tool Execution          │  │  ← Hooks 拦截       │
│          │  │  run_one_tool / JoinSet     │  │    (BeforeToolCall) │
│          │  └─────────────────────────────┘  │                     │
│          └────────┬──────────────────────────┘                     │
│                   │                                                 │
│  ┌────────────────┼──────────────────┐                             │
│  │                │                  │                             │
│  │     ┌──────────▼────┐  ┌─────────▼─────────┐                    │
│  │     │Skills Registry│  │  Hooks Registry   │                    │
│  │     │  (SKILL.md)   │  │(6 lifecycle pts)  │                    │
│  │     └───────────────┘  └───────────────────┘                    │
│  └─────────────────────────────────────────────────────────────────┘│
│                   │                                                 │
│          ┌────────▼────────┐                                       │
│          │  Tool Registry  │                                       │
│          │  (Built-in:     │                                       │
│          │  Essay Topic    │                                       │
│          │  + Student)     │                                       │
│          └────────┬────────┘                                       │
│                   │                                                 │
│          ┌────────▼────────┐                                       │
│          │    Database     │                                       │
│          │    (libSQL)     │                                       │
│          └─────────────────┘                                       │
└──────────────────────────────────────────────────────────────────────┘
```

## 各层说明

### Channels（通道层）

消息入口，负责接收用户输入并转发给 Agent Loop。Channel trait 定义了统一的 `start / respond / send_status / broadcast` 接口。

| 通道          | 说明                              |
|-------------|---------------------------------|
| REPL        | 终端命令行交互 (crossterm + rustyline) |
| Web Gateway | Tauri IPC + SSE，前后端双向通信         |

### Agent Loop（消息分发）

`Agent::run()` 主循环，通过 `ChannelManager::start_all()` 合并所有通道的消息流，用 `tokio::select!` 统一处理。每条消息经
`SubmissionParser` 解析意图后分发：

- `UserInput` → `process_user_input()` → 进入 Agentic Loop
- `ApprovalResponse` → 审批确认/拒绝
- `SystemCommand` → `/help` 等系统命令
- `Interrupt` → 中断当前处理

> 对比参考图：参考图在 Agent Loop 后分为 Scheduler（并行任务调度）和 Routines Engine（定时/事件触发），deepEssay 目前只有 Chat
> 交互路径，这两个分支尚未实现。

### Session Manager（会话管理）

管理 `Session → Thread → Turn` 三层结构：

- **Session**: 按 `user_id` 隔离，一个用户一个 Session
- **Thread**: 按 `user_id + agent_id + channel + external_thread_id` 映射，支持历史线程 DB 水化 (Hydration) 和 UUID
  采用 (Adoption)
- **Turn**: 每次对话轮次，记录 user_input → tool_calls → response

### Agentic Loop（核心循环）

`run_agentic_loop()` 是 LLM ↔ Tool 的多轮交互引擎，通过 `LoopDelegate` trait 解耦具体场景：

```
for iteration in 1..=max_iterations:
  check_signals()
  → before_llm_call()          // 发送 Thinking 状态
  → call_llm()                 // Reasoning::respond_with_tools()
  → match RespondResult:
      Text       → 返回最终回复
      ToolCalls  → execute_tool_calls() → 执行后继续下一轮
```

- **单工具**顺序执行，**多工具**通过 `JoinSet` 并行执行
- 截断保护：`finish_reason=Length` 时丢弃不完整的 tool_calls，累计 3 次后强制 `force_text` 模式
- 上下文溢出时触发压缩（TODO）

### Skills / Hooks（横切关注点）

- **Skills Registry**: 扫描 `~/.deepEssay/skills/` 目录，解析 SKILL.md 的 YAML frontmatter，按关键词匹配激活，将匹配的技能
  prompt 注入 LLM 系统提示的 `<skill>` 块中
- **Hooks Registry**: 6 个生命周期钩子点（BeforeInbound / BeforeToolCall / BeforeOutbound / OnSessionStart /
  OnSessionEnd / TransformResponse），按优先级链式执行，支持 FailOpen/FailClosed 策略

> 这两个模块在参考图中没有独立体现 — 参考图的钩子能力隐含在 Agent Loop 内部。

### Tool Registry（工具注册表）

管理 LLM 可用的工具，提供 `register / get / definitions` 接口。当前内置 2 个作文查询工具：

| 工具                    | 功能                |
|-----------------------|-------------------|
| `essay_topic_query`   | 按关键词模糊查询作文题目      |
| `student_essay_query` | 按题目ID/班级/学生姓名查询作文 |

> 对比参考图：参考图的 Tool Registry 支持 Built-in + MCP + WASM 三类工具源，deepEssay 目前仅有 Built-in。MCP 和 WASM
> 是未来的扩展方向。

### Database（持久化层）

libSQL 嵌入式数据库，存储 5 张表：
`agents / conversations / conversation_messages / essay_topics / student_essays`

每个 Turn 的消息按顺序持久化：`user → tool_calls → assistant`，支持崩溃恢复。
