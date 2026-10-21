# 02 整体架构

## 1. 分层视图

```text
┌──────────────────────────────────────────────────────────────┐
│ 接口层                                                        │
│  Interactive TUI │ Print/JSON │ RPC(JSONL) │ TypeScript SDK   │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│ 会话编排层（coding-agent）                                    │
│  AgentSession │ AgentSessionRuntime │ SessionManager          │
│  SettingsManager │ ModelRuntime │ ResourceLoader │ Extensions  │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│ Agent 层（pi-agent-core）                                     │
│  Agent │ runAgentLoop │ 工具执行 │ 事件 │ steering/follow-up  │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│ 模型层（pi-ai）                                               │
│  Models 集合 │ Provider │ API 实现 │ Auth │ 流式事件 │ 成本    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│ 基础设施                                                      │
│ TUI │ Chord │ Durable │ Codemode │ MCP │ Env │ Telemetry     │
└──────────────────────────────────────────────────────────────┘
```

## 2. 包依赖

内部依赖是单向的，构建顺序即拓扑顺序：

```text
chord ─┐
tui    ├─→ (独立)
telemetry ─→ ai ─→ agent ─→ coding-agent
codemode ─┐                 ↑
mcp ──────┴─────────────────┘
chord + ai + codemode ─→ durable ─→ env
evals ─→ coding-agent + ai（开发依赖）
```

根 `npm run build` 的实际顺序：

```text
chord → tui → telemetry → codemode → mcp → ai → durable → env → agent → coding-agent
```

`coding-agent` 的 `tsconfig.build.json` 直接引用 `../agent/dist`、`../ai/dist`、`../tui/dist` 的类型声明，因此必须先构建这些包。

## 3. 一次交互式请求的生命周期

### 3.1 启动阶段

1. `src/cli.ts` 调用 `setupCli()` 后执行 `main(process.argv.slice(2))`。
2. `main()` 解析参数、处理 `auth`/`config`/`package`/`mcp` 子命令、迁移、首次设置、项目信任与 Session 选择。
3. `createAgentSessionServices()` 创建 cwd 绑定服务：`ModelRuntime`、`SettingsManager`、`DefaultResourceLoader`，并注册扩展声明的 Provider/虚拟模型。
4. `createAgentSessionFromServices()` → `createAgentSession()` 创建并连接 `Agent` 与 `AgentSession`。
5. `createAgentSessionRuntime()` 把当前 `AgentSession` 与服务和重建工厂封装成运行时，支持后续 `/new`、`/resume`、`/fork`、`/import`。
6. 根据模式进入 `InteractiveMode`、`runPrintMode()` 或 `runRpcMode()`。

关键文件：[main.ts](../packages/coding-agent/src/main.ts)、[agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)。

### 3.2 一次 Prompt

```text
session.prompt(text, options)
  ├─ 扩展命令/技能命令/prompt template 展开
  ├─ 若正在运行：
  │    ├─ streamingBehavior=steer  → 排队，当前 assistant turn 后注入
  │    └─ streamingBehavior=followUp → 排队，当前 run 结束后注入
  ├─ _prepareRun()
  │    ├─ 必要时先压缩
  │    ├─ emit before_agent_start（扩展可改 system prompt / tools / 注入消息）
  │    ├─ 处理图片、pending next-turn 消息
  │    └─ 生成 tools loadout 更新消息
  └─ agent.prompt(messages)
       └─ runAgentLoop()
```

`AgentSession` 负责业务语义（持久化、压缩、重试、模型切换、扩展），`Agent` 只负责循环和事件。

### 3.3 低层 Agent 循环

`runAgentLoop()`（[agent-loop.ts](../packages/agent/src/agent-loop.ts)）的核心步骤：

1. 对 pending 消息做 `declareToolChanges()`：把可执行工具集与 transcript 已声明工具的差异写入 system message。
2. 发出 `agent_start`、`turn_start`。
3. 调用 `prepareRequest`（若配置）替换本次请求的 context/model/thinking。
4. `streamAssistantResponse()`：
   - `transformContext(messages)` → `convertToLlm(messages)` → `normalizeContext()`。
   - 解析动态 API key，调用 `streamFn(model, transcript, options)`。
   - 把 Provider 的 `text/thinking/toolcall` 事件转成 `message_start/message_update/message_end`。
5. 若响应被 `length` 截断，所有工具调用直接失败，避免执行参数不完整的调用。
6. 否则执行工具批次：
   - `sequential`：逐个准备、执行、finalize。
   - `parallel`：顺序 preflight，允许的工具并发执行，`tool_execution_end` 按完成顺序发出，toolResult 消息仍按 assistant 源顺序写入。
7. 发出 `turn_end`，调用 `finishTurn`；根据决策、工具结果、steering 队列决定是否继续。
8. 没有工具调用且没有 steering 时，检查 follow-up 队列；有则继续，否则 `agent_end`。

### 3.4 事件与持久化

`Agent.subscribe()` 的监听器按注册顺序串行 await。`AgentSession` 订阅这些事件并：

- 把消息 entry 写入 `SessionManager`（JSONL 树）。
- 转发为 `AgentSessionEvent` 给 UI/RPC。
- 触发自动压缩、自动重试、扩展事件。

`agent_end` 只表示低层循环不再发事件；`agent_settled` 才表示 Pi 不会自动继续（重试、队列、before-settle 边界都已处理完）。

## 4. 会话与上下文

### 4.1 JSONL 会话树

`SessionManager` 维护：

- 一条 `session` header（id、cwd、parentSession、version）。
- 一组 entry：`message`、`model_change`、`thinking_level_change`、`compaction`、`branch_summary`、`custom`、`custom_message`、`label`、`session_info`、`context_edit`、`usage`。
- 每个 entry 有 `id` 与 `parentId`，形成树；`leafId` 指向活动分支。

模型上下文由活动分支重建：`buildSessionContext()` 应用压缩与分支摘要，产生 `messages`、`thinkingLevel`、`model`。

关键文件：[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[messages.ts](../packages/coding-agent/src/core/messages.ts)。

### 4.2 压缩

`AgentSession` 在请求前检查 token 阈值；`compact()` 调用 `prepareCompaction()` / `compact()`：

- 选择 cut point，保留最近消息。
- 生成摘要 entry（`compaction`），附带文件操作与 usage。
- 原始 entry 不删除，只是不再进入后续模型请求。
- 分支切换时可生成 `branch_summary`。

关键文件：[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[branch-summarization.ts](../packages/coding-agent/src/core/compaction/branch-summarization.ts)。

### 4.3 上下文装配

一次请求的上下文由以下部分组成：

- 活动分支转换出的 user/assistant/toolResult 消息。
- 系统提示：基础指令 + 项目/用户 context files + 扩展 section + 工具声明 + 技能描述。
- 工具声明：只声明当前 loadout，工具增删以 system message 形式告知模型。
- `convertToLlm`：过滤 UI-only 消息，并可选择屏蔽图片。
- `transformContext`：扩展可改写/裁剪 `AgentMessage[]`。

## 5. 扩展与资源加载

### 5.1 加载顺序

`DefaultResourceLoader.reload()`（[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)）负责：

1. 发现扩展（用户目录、项目 `.pi`、CLI 路径、Pi packages、内置扩展）。
2. 解析项目信任；未信任时不加载项目设置与项目资源。
3. 加载扩展、技能、提示模板、主题、context files、系统提示。
4. 注册扩展声明的 Provider、虚拟模型、MCP server、工具、命令、快捷键、CLI flag。

### 5.2 扩展能力

`ExtensionAPI`（[types.ts](../packages/coding-agent/src/core/extensions/types.ts)）提供：

- 生命周期/会话/Provider/工具/消息/UI 事件订阅。
- `registerTool()`、`registerCommand()`、`registerShortcut()`、`registerFlag()`。
- 自定义消息、entry、工具渲染器。
- `sendMessage()`、`sendUserMessage()`、`appendEntry()`、`exec()`、`setActiveTools()`、`setModel()`、Provider 注册。
- `ExtensionUIContext`：select/confirm/input/notify、状态栏、widget、编辑器替换、终端输入监听。

`ExtensionRunner` 负责按扩展加载顺序触发处理器、合并结果、记录错误。

### 5.3 内置扩展

| 内置扩展 | 作用 | 位置 |
|---|---|---|
| `codemode` | 模型写 JS，脚本内调用工具，只有脚本输出进入上下文 | [extensions/codemode](../packages/coding-agent/src/extensions/codemode/) |
| `mcp` | 连接 MCP server，把工具/资源暴露给 Pi | [extensions/mcp](../packages/coding-agent/src/extensions/mcp/) |
| `tool_search` | 搜索并加载延迟声明的工具 | [extensions/tool-search](../packages/coding-agent/src/extensions/tool-search/) |
| `llama` | 本地 llama.cpp Provider/UI | [extensions/llama](../packages/coding-agent/src/extensions/llama/) |

## 6. 并发、取消与失败处理

- **单运行约束**：`Agent` 同一时间只允许一个 active run；重复 `prompt()/continue()` 会抛错，应使用 `steer()`/`followUp()` 排队。
- **取消**：每次 run 创建 `AbortController`；`agent.abort()` 触发信号，工具和 Provider 请求自行遵守。
- **工具批次终止**：每个工具结果可带 `terminate: true`；仅当整批结果都为 true 时跳过后续模型请求。
- **重试**：`AgentSession` 对可重试的 assistant 错误做指数退避重试；上下文溢出可触发 overflow recovery（压缩后重试一次）。
- **存储失败**：durable 的 Session 把 Storage 错误视为终态，后续调用返回 `SessionFailed`；coding-agent 的 JSONL 写入失败会反映为会话错误。

## 7. 安全与信任

- 项目设置与项目资源只有在项目被信任后才加载；`ProjectTrustStore` 记录决定，`--project-trust` 可覆盖。
- Context files（`AGENTS.md` 等）的发现不要求项目信任。
- 工具与扩展继承 Pi 进程的 OS 权限；Pi 不内置文件/网络/凭据权限系统。
- 容器化方案见 `packages/coding-agent/docs/containerization.md`：Gondolin 微 VM、Docker、OpenShell。

## 8. coding-agent 与 durable 的关系

两者解决不同问题，互不依赖：

| 维度 | coding-agent | durable |
|---|---|---|
| 目标 | 本地编码助手 CLI | 可持久化、可恢复的服务端 Agent 运行时 |
| 存储 | JSONL 会话文件 | Memory / JSONL / SQLite，提交即持久化 |
| 循环 | `AgentSession` + `pi-agent-core` | `Harness` + 自研任务调度器 |
| 恢复 | 从会话文件重建上下文 | 进程重启后从任务 checkpoint 继续 |
| 扩展模型 | 进程内 TS 模块 | 命名 extension + registry + task |
| 代码执行 | 内置工具 + codemode 扩展 | `ExecutionEnv` + codemode 子包 |

## 9. 构建拓扑与产物

| 包 | 产物 |
|---|---|
| tui | `dist/`，可选原生剪贴板 `.node` 预构建 |
| ai | `dist/` + `providers/data` 模型数据 |
| agent | `dist/` |
| coding-agent | `dist/`、`dist/bundle/cli.js`、Bun 独立二进制 |
| durable/chord/codemode/mcp/env/telemetry | `dist/` |
| env | 另有 Rust daemon 二进制（Cargo） |

coding-agent 的 Node 发布包通过 `dist/bundle/cli.js` 暴露 `pi` bin；Bun 二进制则嵌入主入口、图片缩放 worker 与 codemode worker。
