# deepEssay 项目架构文档

## 1. 项目概览

deepEssay 是一款基于 Tauri 2 的桌面应用，定位为 **AI 作文批改助手**，专为中文写作教学场景设计。应用支持上传作文题目和学生作文数据，通过 LLM 进行智能批改、对比分析，并提供写作建议。

### 1.1 技术栈

| 层级 | 技术 |
|------|------|
| 桌面框架 | Tauri 2 |
| 后端语言 | Rust (edition 2024) |
| 前端框架 | Vue 3 (Composition API) + TypeScript |
| 构建工具 | Vite 8 |
| 路由 | Vue Router 4 (Hash 模式) |
| LLM 集成 | rig-core 0.30 (OpenAI-compatible) |
| 数据库 | libSQL (嵌入式 SQLite-compatible) |
| 异步运行时 | Tokio |
| Markdown 渲染 | marked + DOMPurify |

### 1.2 项目结构

```
deepEssay/
├── backend/                    # Rust 后端 (Tauri)
│   ├── src/
│   │   ├── main.rs             # 入口：Tauri 启动 + Agent 初始化
│   │   ├── lib.rs              # 模块声明
│   │   ├── app.rs              # AppBuilder：组装所有组件
│   │   ├── bootstrap.rs        # 基础路径 (~/.deepEssay/)
│   │   ├── error.rs            # 统一错误类型
│   │   ├── tenant.rs           # 租户上下文 (TenantScope / TenantCtx)
│   │   ├── timezone.rs         # 时区处理
│   │   ├── agent/              # Agent 核心：消息处理 + 会话管理
│   │   ├── channels/           # 通信通道抽象层
│   │   ├── cli/                # CLI REPL 通道
│   │   ├── common/             # 通用工具
│   │   ├── config/             # 配置系统
│   │   ├── context/            # 任务上下文 (JobContext)
│   │   ├── db/                 # 数据库抽象 + libSQL 实现
│   │   ├── history/            # 历史数据类型
│   │   ├── hooks/              # 生命周期钩子系统
│   │   ├── llm/                # LLM 抽象层
│   │   ├── skills/             # 技能系统
│   │   ├── testing/            # 测试工具
│   │   └── tools/              # 工具注册与执行
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── .deepEssay/             # 本地数据 (config.toml + deepEssay.db)
├── frontend/                   # Vue 3 前端
│   └── src/
│       ├── main.ts             # Vue 应用入口
│       ├── App.vue             # 根组件 (侧边栏 + 路由视图)
│       ├── api/                # Tauri IPC 封装
│       ├── components/         # 全局组件 (AppSidebar)
│       ├── router/             # 路由配置
│       ├── tenant/             # 当前用户
│       ├── types/              # TypeScript 类型定义
│       └── views/              # 页面视图
│           ├── chat/           # 聊天页 (核心)
│           ├── projects/       # 项目管理页
│           └── settings/       # 设置页
└── docs/                       # 文档
```

---

## 2. 启动流程

```
main.rs: main()
  ├── 解析 --config 参数 (TOML 配置路径)
  ├── tauri::Builder::default()
  │     ├── .setup(setup_backend)     ← 核心初始化
  │     └── .invoke_handler(...)      ← 注册 IPC handler
  └── .run()
```

### 2.1 async_init 初始化步骤

```
1. Config::from_toml_path()         → 加载 TOML 配置
2. AppBuilder::new(config)
     .build_all()                   → 组装所有组件:
       ├── init_database()          → 连接 libSQL + 执行迁移
       ├── init_llm()               → 构建 LLM provider 链
       ├── init_tools()             → 注册 essay 工具
       ├── HookRegistry             → 钩子注册表
       ├── SessionManager           → 会话管理器
       └── SkillRegistry            → 技能系统 (可选)
3. ChannelManager::new()            → 通道管理器
4. GatewayChannel                   → Web 网关通道 (SSE)
5. bootstrap_hooks()                → 注册内置钩子
6. Agent::new() + agent.run()       → 启动 Agent 主循环
```

---

## 3. 核心架构：Agent 系统

### 3.1 整体数据流

```
Frontend (Vue) ──[Tauri IPC]──▶ GatewayChannel ──▶ ChannelManager
                                                        │
                                                   消息流 (Stream)
                                                        │
                                                        ▼
                                                   Agent::run()
                                                        │
                                                   主循环 select!
                                                    ├── 处理消息
                                                    │   ├── SubmissionParser 解析
                                                    │   ├── SessionManager 分配会话/线程
                                                    │   ├── process_user_input()
                                                    │   │   ├── persist_user_message (DB)
                                                    │   │   └── run_agentic_loop()
                                                    │   │       └── Reasoning ◀──▶ LLM
                                                    │   │           └── ToolRegistry 执行工具
                                                    │   └── 发送响应 + 状态更新
                                                    └── Ctrl+C 退出
```

### 3.2 Agent 组件 (/backend/src/agent/)

| 文件 | 职责 |
|------|------|
| `agent_loop.rs` | Agent 结构体、主循环 `run()`、消息分发 `handle_message()`、技能选择 |
| `session.rs` | Session / Thread / Turn 数据结构、消息序列化 `messages()` |
| `session_manager.rs` | 会话管理：创建/查找 Session，Thread 映射与采用 (UUID adoption) |
| `dispatcher.rs` | ChatDelegate：实现 LoopDelegate trait，工具执行的 6 步流程 |
| `agentic_loop.rs` | 通用 Agentic 循环：LLM 调用 → 工具执行 → 文本响应，可被多种 delegate 复用 |
| `submission.rs` | 消息解析：UserInput / ApprovalResponse / Interrupt / SystemCommand |
| `thread_ops.rs` | 用户输入处理、响应转换、DB 持久化 (user / tool_calls / assistant) |

### 3.3 Agentic 循环流程

```
run_agentic_loop()  ← 单次消息处理
  │
  └── for iteration in 1..=max_iterations:
        ├── check_signals()          → 检查中止信号
        ├── before_llm_call()        → 发送 "Thinking" 状态
        ├── call_llm()               → Reasoning::respond_with_tools()
        ├── match RespondResult:
        │     ├── Text               → handle_text_response() → 返回
        │     └── ToolCalls          → execute_tool_calls()
        │           ├── 注入 assistant 消息
        │           ├── 发送 ToolStarted 状态
        │           ├── 并行/顺序执行工具 (JoinSet)
        │           ├── 发送 ToolCompleted 状态
        │           └── 注入 tool_result 消息 → 下一轮 LLM
        └── after_iteration()
```

### 3.4 Thread状态机

```
ThreadState:
  Idle ──▶ Processing ──▶ [Idle | AwaitingApproval | Completed | Interrupted]

TurnState:
  Processing ──▶ [Completed | Failed | Interrupted]
```

---

## 4. LLM 层 (/backend/src/llm/)

### 4.1 模块结构

| 文件 | 职责 |
|------|------|
| `provider.rs` | LlmProvider trait、ChatMessage、ToolCall、ToolDefinition |
| `config.rs` | LlmConfig、RegistryProviderConfig |
| `reasoning.rs` | Reasoning 引擎：系统提示构建、respond_with_tools、响应清理 |
| `rig_adapter.rs` | RigAdapter：将 rig-core 适配为 LlmProvider trait |
| `registry.rs` | ProviderProtocol 枚举 (当前仅 OpenAiCompatible) |
| `recording.rs` | RecordingLlm：LLM 调用录制 (调试用) |
| `error.rs` | LLM 错误类型 |

### 4.2 LLM 调用流程

```
Reasoning::respond_with_tools(context)
  ├── merge_system_messages()        ← 合并多段 system 提示
  ├── 构建 messages [system, ...history, user]
  ├── 判断是否有可用工具:
  │     ├── 有工具 → complete_with_tools()
  │     │   ├── 返回 ToolCalls → truncate_at_tool_tags() + clean_response()
  │     │   └── 返回 Text → truncate_at_tool_tags() + clean_response()
  │     └── 无工具 → complete()
  └── 返回 RespondOutput { RespondResult, FinishReason }
```

### 4.3 系统提示结构

```
1. deepEssay 作文批改助手 身份声明
2. 响应格式 (<think>...</think> + <suggestions>)
3. 输出与表达规范 (中文、Markdown、引用块)
4. 安全与边界
5. 工具使用纪律
6. 可用工具列表
7. 身份说明 (workspace_system_prompt)
8. 活跃技能上下文
```

---

## 5. 通道系统 (/backend/src/channels/)

### 5.1 架构

```
ChannelManager (管理多个 Channel)
  ├── GatewayChannel   → Tauri IPC / SSE (前端通信)
  ├── ReplChannel      → CLI REPL (终端交互)
  └── 可扩展: Slack、Telegram、HTTP 等

Channel trait:
  ├── name()           → 通道名称
  ├── start()          → 返回 MessageStream
  ├── respond()        → 回复消息
  ├── send_status()    → 发送状态更新
  ├── broadcast()      → 主动推送
  ├── health_check()   → 健康检查
  └── shutdown()       → 优雅关闭
```

### 5.2 消息类型

| 类型 | 说明 |
|------|------|
| `IncomingMessage` | 用户输入 (含 user_id, agent_id, thread_id, content, timezone) |
| `OutgoingResponse` | Agent 回复 (文本 + 附件) |
| `StatusUpdate` | 状态更新: Thinking, ToolStarted, ToolCompleted, ToolResult, ReasoningUpdate, Suggestions, ApprovalNeeded, StreamChunk, JobStarted, TurnCost |

### 5.3 GatewayChannel 数据流

```
Frontend (ChatView)
  └── Tauri Channel<WsServerMessage>
        └── useChatConnection composable
              └── handleChatConnection() IPC
                    └── GatewayState.msg_tx → IncomingMessage → Agent Loop
                    └── GatewayState.sse → AppEvent → Frontend Channel.onmessage
```

---

## 6. 工具系统 (/backend/src/tools/)

### 6.1 结构

```
Tool trait:
  ├── name()                → 工具名称
  ├── description()         → 工具描述 (给 LLM)
  ├── parameters_schema()   → JSON Schema 参数定义
  └── execute(params, ctx)  → 执行工具

ToolRegistry:
  ├── register(tool)        → 注册工具
  ├── get(name)             → 按名查找
  ├── definitions()         → 获取所有 ToolDefinition (给 LLM)
  └── register_essay_tools()→ 注册作文相关工具
```

### 6.2 内置工具

| 工具名 | 功能 |
|--------|------|
| `essay_topic_query` | 按关键词模糊查询作文题目 |
| `student_essay_query` | 按题目ID、班级、学生姓名查询作文 |

### 6.3 工具执行流程 (dispatcher.rs)

```
execute_tool_calls():
  1. 注入 assistant 消息 (含 tool_calls)
  2. 发送 Thinking 状态
  3. 发送 ReasoningUpdate (思考过程)
  4. 写入 Turn 占位 (TurnToolCall)
  5. 执行工具:
     ├── 单个工具 → 顺序执行
     └── 多个工具 → JoinSet 并行执行
  6. Post-flight: 更新 Turn 记录 + 注入 tool_result
```

---

## 7. 持久化层 (/backend/src/db/)

### 7.1 数据库抽象

```
Database trait (组合多个子 trait):
  ├── AgentStore        → list_agents()
  ├── ConversationStore → ensure_conversation(), list_conversations(),
  │                        list_conversation_messages(), add_conversation_message(),
  │                        update_conversation_metadata_field()
  └── EssayStore        → search_essay_topic(), search_student_essay()
  └── run_migrations()  → 数据库迁移
```

### 7.2 libSQL 实现

```
LibSqlBackend
  ├── new_local(path)  → 本地文件数据库
  ├── new_memory()     → 内存数据库 (测试用)
  └── connect()        → 创建连接 (带重试 + busy_timeout)

数据表:
  ├── agents           → Agent 信息
  ├── conversations    → 会话/线程映射
  ├── conversation_messages → 消息历史 (user/tool_calls/assistant)
  ├── essay_topics     → 作文题目
  └── student_essays   → 学生作文
```

### 7.3 消息持久化顺序

```
每个 Turn 的持久化顺序:
  1. ensure_conversation()        ← 确保 conversation 行存在
  2. add_conversation_message()   ← 写入 user 消息
  3. add_conversation_message()   ← 写入 tool_calls 消息
  4. add_conversation_message()   ← 写入 assistant 消息
```

---

## 8. 钩子系统 (/backend/src/hooks/)

### 8.1 生命周期钩子点

| HookPoint | 触发时机 |
|-----------|---------|
| BeforeInbound | 入站消息处理前 |
| BeforeToolCall | 工具执行前 |
| BeforeOutbound | 出站响应发送前 |
| OnSessionStart | 新会话创建时 |
| OnSessionEnd | 会话结束时 |
| TransformResponse | 最终响应转换前 |

### 8.2 钩子执行

```
HookRegistry::run(event):
  ├── 筛选匹配的钩子 (按 hook_point)
  ├── 按优先级排序执行
  ├── Reject → 停止链并返回错误
  ├── Modify → 修改内容并传递到下一个钩子
  └── 超时/失败处理 (FailOpen / FailClosed)
```

---

## 9. 技能系统 (/backend/src/skills/)

### 9.1 技能生命周期

```
SKILL.md 文件
  ├── YAML frontmatter (SkillManifest)
  │     ├── name, version, description
  │     └── activation: keywords, exclude_keywords, tags, max_context_tokens
  └── Markdown body → prompt_content

SkillRegistry:
  ├── discover_all()               → 扫描 skills/ + installed_skills/ 目录
  └── prefilter_skills()           → 按关键词匹配活跃技能

激活流程:
  消息到达 → select_active_skills() → 评分排序 → 注入 <skill> 上下文块
```

### 9.2 安全限制

- 技能名验证: `^[a-zA-Z0-9\p{Script=Han}][...]{0,63}$`
- 文件大小限制: 64 KiB
- 关键词上限: 20 个
- 标签上限: 10 个
- 最小关键词/标签长度: 3 字符
- Symlink 检测: 拒绝符号链接
- Gating 检查: 必需二进制/PATH/环境变量/配置文件

---

## 10. 配置系统 (/backend/src/config/)

### 10.1 配置结构 (config.toml)

```toml
owner_id = "..."                    # 实例所有者
database_backend = "libsql"         # 数据库后端
libsql_path = "..."                 # SQLite 文件路径
llm_backend = "openai_compatible"   # LLM 后端
provider_id = "..."                 # Provider ID
openai_compatible_base_url = "..."  # API 地址
openai_compatible_base_api_key = "..." # API Key
selected_model = "..."              # 模型名称
request_timeout_secs = 120          # 请求超时
skills_enabled = true               # 技能系统开关
[channels]
cli_enabled = true                  # CLI 通道开关
```

### 10.2 默认路径

| 用途 | 路径 |
|------|------|
| 配置 | `~/.deepEssay/config.toml` |
| 数据库 | `~/.deepEssay/deepEssay.db` |
| 技能 | `~/.deepEssay/skills/` |
| 已安装技能 | `~/.deepEssay/installed_skills/` |

---

## 11. 前端架构 (/frontend/)

### 11.1 路由

| 路径 | 组件 | 说明 |
|------|------|------|
| `/` | redirect → /chat | 默认跳转 |
| `/chat` | ChatView.vue | 聊天主界面 |
| `/projects` | ProjectsView.vue | 项目管理 |
| `/settings` | SettingsView.vue | 设置页 |

### 11.2 ChatView 组件树

```
ChatView.vue
├── ChatSidebar.vue                    # 左侧边栏
│   ├── AgentItem.vue                  # Agent 列表项
│   ├── ConversationList.vue           # 会话列表
│   │   └── ConversationItem.vue       # 会话项
│   └── ChatInput.vue                  # 新消息输入框
└── ChatPanel.vue                      # 右侧对话面板
    └── MessageList.vue                # 消息列表
        └── MessageBubble.vue          # 消息气泡 (Markdown 渲染)

Composables:
├── useChatConnection.ts              # Tauri Channel 连接管理
├── chatEventBus.ts                   # 事件总线 (mitt-like)
├── useAgents.ts                      # Agent 列表数据
├── useConversations.ts               # 会话列表数据
├── useChatHistory.ts                 # 聊天历史加载
└── useChatSelection.ts               # 选中状态管理
```

### 11.3 通信协议

```
前端 → 后端 (IPC invoke):
  ├── list_agents_handler          → 获取 Agent 列表
  ├── list_conversations_handler   → 获取会话列表
  ├── chat_history_handler         → 获取聊天历史
  ├── chat_send_handler            → 发送消息
  ├── chat_new_thread_handler      → 创建新线程
  └── handle_chat_connection       → 建立 SSE 通道

后端 → 前端 (Tauri Channel):
  ├── WsServerEvent { event_type, data }
  │   ├── 'thinking'              → 思考中状态
  │   ├── 'tool_result'           → 工具执行结果
  │   └── 'response'              → Agent 回复
  └── WsServerError { message }   → 错误消息
```

---

## 12. 错误处理

```
Error (顶层)
  ├── Channel(ChannelError)
  │     ├── StartupFailed
  │     ├── SendFailed
  │     ├── MissingRoutingTarget
  │     └── HealthCheckFailed
  ├── Job(JobError)
  │     ├── NotFound { id: Uuid }
  │     └── ContextError { id, reason }
  └── Llm(LlmError)
        ├── AuthFailed { provider }
        ├── RequestFailed { provider, reason }
        ├── InvalidResponse { provider, reason }
        └── ContextLengthExceeded { used, limit }

其他错误类型:
  ├── ConfigError         → 配置解析/缺失
  ├── DatabaseError       → 连接池/查询/迁移
  ├── SkillRegistryError  → 技能读取/解析/门控
  ├── HookError           → 钩子执行/超时/拒绝
  └── ToolError           → 工具参数/执行
```

---

## 13. 测试架构 (/backend/src/testing/)

| 文件 | 说明 |
|------|------|
| `mod.rs` | 测试模块入口 |
| `test_rig.rs` | LLM/Rig 测试 |
| `llm_test.rs` | LLM provider 测试 |
| `agent_loop_test.rs` | Agent 循环测试 |
| `test_components.rs` | 组件集成测试 |
| `test_channel.rs` | 通道测试 |
| `db_test.rs` | 数据库测试 |

---

## 14. 当前开发状态 (2026-05-22)

### 已完成
- [x] Tauri 2 桌面框架初始化
- [x] libSQL 数据库 + 迁移 + 示例数据
- [x] LLM Provider 抽象层 (OpenAI-compatible via rig-core)
- [x] Agent 主循环 + 消息分发
- [x] Session/Thread/Turn 会话管理
- [x] Agentic Loop (LLM → 工具执行 → 响应)
- [x] 工具系统 + 作文查询工具
- [x] GatewayChannel (前端 SSE 通信)
- [x] 钩子系统 (6 个生命周期钩子点)
- [x] 技能系统 (SKILL.md 解析 + 关键词匹配)
- [x] 消息持久化 (user → tool_calls → assistant)
- [x] 历史线程 DB 水化 (Hydration)
- [x] 前端聊天界面 + Markdown 渲染

### 待完成 (TODO)
- [ ] Approval 审批流程 (NeedApproval 状态已定义但分支为 `todo!()`)
- [ ] 思考标签清洗 (clean_response 目前是 no-op)
- [ ] 上下文压缩 (ContextLengthExceeded 后 compact_messages)
- [ ] 多 Provider 支持 (Ollama 等)
- [ ] 图片消息支持
- [ ] /command 系统命令 (完整路由)
- [ ] 队列中的请求处理
- [ ] 优雅关闭 + shutdown 信号处理
- [ ] Token 用量统计
- [ ] workspace 记忆加载
- [ ] 前端 ProjectsView + SettingsView 完整实现
- [ ] 前端 ToolCall 可视化展示
