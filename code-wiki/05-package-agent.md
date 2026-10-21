# 05 `@earendil-works/pi-agent-core`

## 1. 职责

`pi-agent-core` 是 Provider 无关的有状态 Agent 运行时：

- 持有 transcript、模型、思考级别、工具与运行状态。
- 驱动“模型响应 → 工具执行 → 是否继续”的循环。
- 提供生命周期事件流，供 UI、RPC、持久化层消费。
- 管理 steering / follow-up 队列。
- 定义工具、工具结果、hook 与并发执行语义。
- 通过可注入的 `StreamFn` 调用任意模型层。

源码：[packages/agent/src](../packages/agent/src/)

## 2. 模块

| 文件 | 职责 |
|---|---|
| [agent.ts](../packages/agent/src/agent.ts) | `Agent` 类：状态、事件订阅、prompt/continue、队列、中止、生命周期 |
| [agent-loop.ts](../packages/agent/src/agent-loop.ts) | 低层循环 `runAgentLoop` / `runAgentLoopContinue`、工具执行、事件发出 |
| [types.ts](../packages/agent/src/types.ts) | 全部公开类型：`AgentTool`、`AgentEvent`、`AgentLoopConfig`、hooks |
| [stream-fn.ts](../packages/agent/src/stream-fn.ts) | 默认 `StreamFn` 的注册与读取 |
| [proxy.ts](../packages/agent/src/proxy.ts) | 通过服务端代理 LLM 请求的 `streamProxy()` |
| [index.ts](../packages/agent/src/index.ts) | 汇总导出 |

## 3. 关键类型

### 3.1 AgentMessage 与 LLM Message

```ts
AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages]
```

应用可用 declaration merging 扩展 `CustomAgentMessages`。模型只理解 `user/assistant/toolResult/system`，因此需要：

```text
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
```

- `transformContext`：裁剪旧消息、注入外部上下文。
- `convertToLlm`：过滤 UI-only 消息、把自定义消息转成标准消息。
- 两者契约都是“不抛错、不 reject”。

### 3.2 AgentTool

```ts
interface AgentTool<TParameters, TDetails> extends Tool<TParameters> {
  label: string;
  parameters: TSchema;
  outputSchema?: TSchema;
  replay?: "never" | "safe";
  executionMode?: "sequential" | "parallel";
  prepareArguments?(args: unknown): Static<TParameters>;
  execute(
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: (partial: AgentToolResult<TDetails>) => void
  ): Promise<AgentToolResult<TDetails>>;
}
```

`AgentToolResult` 通道：

- `content`：给模型看的 text/image。
- `details`：给 UI/日志。
- `structuredContent`：给程序调用方，不发给模型。
- `usage`：工具自身消耗。
- `isError`、`terminate`。

### 3.3 AgentEvent

| 事件 | 说明 |
|---|---|
| `agent_start` | 运行开始 |
| `agent_end` | 最后一次运行事件；监听器仍计入 settlement |
| `turn_start` / `turn_end` | 一次模型响应 + 工具结果 |
| `message_start` / `message_update` / `message_end` | 消息生命周期；`message_update` 只用于 assistant 流式增量 |
| `tool_execution_start` / `tool_execution_update` / `tool_execution_end` | 工具生命周期 |

## 4. Agent 类

### 4.1 构造选项

`AgentOptions` 关键字段：

| 字段 | 说明 |
|---|---|
| `initialState` | `systemPrompt/model/thinkingLevel/tools/messages` |
| `streamFn` | 必需（或使用 `setDefaultStreamFn()`）；形状同 `Models.streamSimple` |
| `convertToLlm` | 默认过滤出 system/user/assistant/toolResult |
| `transformContext` | 每次请求前的上下文变换 |
| `getApiKey` | 动态解析短期凭据 |
| `beforeToolCall` / `afterToolCall` | 工具前后 hook；可阻断、改结果 |
| `finishTurn` | turn 结束后决定 `continue` / `end` |
| `prepareRequest` | 每次 Provider 请求前替换 context/model/thinking |
| `prepareNextTurn` / `prepareNextTurnWithContext` | 下一 turn 前替换状态或追加消息 |
| `steeringMode` / `followUpMode` | `"all"` 或 `"one-at-a-time"` |
| `sessionId` | Provider 缓存/会话亲和 |
| `thinkingBudgets` / `transport` / `maxRetryDelayMs` | 传给流函数 |
| `toolExecution` | 默认 `"parallel"` |

### 4.2 状态与生命周期方法

| 方法 | 说明 |
|---|---|
| `state` | 只读 `AgentState`：systemPrompt、model、thinkingLevel、tools、messages、isStreaming 等 |
| `subscribe(listener)` | 按注册顺序 await；返回退订函数 |
| `prompt(input, images?)` | 启动新 run；同一时间只允许一个 |
| `continue()` | 从现有 transcript 继续；末尾必须是 user/toolResult；若末尾是 assistant，则尝试消费 steering 或 follow-up |
| `steer(message)` | 当前 assistant turn 后注入 |
| `followUp(message)` | 当前 run 即将结束时注入 |
| `clearSteeringQueue()` / `clearFollowUpQueue()` / `clearAllQueues()` | 清队列 |
| `hasQueuedMessages()` / `peekQueuedMessages()` | 查询队列 |
| `signal` | 当前 run 的 AbortSignal |
| `abort()` | 中止当前 run |
| `waitForIdle()` | 等待 run 与 `agent_end` 监听器结束 |
| `reset()` | 清空消息与队列，保留 system prompt/tool 基线 |

内部状态机：

```text
runWithLifecycle
  ├─ 创建 AbortController 与 activeRun.promise
  ├─ isStreaming = true
  ├─ executor(signal)  → runAgentLoop / runAgentLoopContinue
  ├─ 失败 → 合成 error/aborted assistant message 并发事件
  └─ finally finishRun() → isStreaming = false，resolve promise
```

`processEvents()` 在把事件交给监听器之前先更新状态：assistant 增量、消息落盘、pending tool calls、errorMessage。

## 5. 低层循环

### 5.1 入口

```ts
agentLoop(prompts, context, config, signal, streamFn): EventStream<AgentEvent, AgentMessage[]>
agentLoopContinue(context, config, signal, streamFn): EventStream<AgentEvent, AgentMessage[]>
runAgentLoop(prompts, context, config, emit, signal, streamFn): Promise<AgentMessage[]>
runAgentLoopContinue(context, config, emit, signal, streamFn): Promise<AgentMessage[]>
```

`agentLoop` 是流式包装；`runAgentLoop` 是可直接 await 的实现。

### 5.2 单 turn 流程

```text
declareToolChanges(context, pendingMessages)
emit agent_start, turn_start
emit pending messages
prepareRequest? → 替换 context/model/thinking
streamAssistantResponse
  ├─ transformContext
  ├─ convertToLlm
  ├─ normalizeContext
  ├─ getApiKey
  └─ streamFn(...) → 流式事件
若 stopReason = error/aborted → finishTurn → turn_end → agent_end
若 stopReason = length → 所有 toolCall 直接失败
否则 executeToolCalls
  ├─ sequential：逐个准备/执行/finalize
  └─ parallel：顺序准备，允许的工具并发执行
finishTurn
turn_end
  ├─ action = "end" → agent_end
  └─ 否则检查 steering / follow-up
```

### 5.3 工具执行细节

- 工具不存在、参数校验失败、`beforeToolCall` 抛出或 signal 已中止 → 立即生成错误 toolResult。
- `beforeToolCall` 返回 `{ block: true, reason }` → 不执行，生成错误结果；可带 `terminate: true`。
- `afterToolCall` 在 `tool_execution_end` 前运行，可替换 `content/details/structuredContent/isError/usage/terminate`。
- 并行模式下：preflight 顺序执行，实际 `execute()` 并发；`tool_execution_end` 按完成顺序发出；toolResult 消息按 assistant 内容源顺序写入。
- 整批结果都带 `terminate: true` 才会跳过下一轮模型请求。

### 5.4 截断保护

当 assistant `stopReason === "length"` 时，流式工具参数可能是“可解析但不完整”的。循环会拒绝执行所有工具调用，并为每个调用生成错误结果，提示模型重新发起。

## 6. 排队与中断语义

```text
prompt()
  ├─ 无 active run → 立即执行
  └─ 有 active run → 抛错，要求 steer()/followUp() 或等待

steer(msg)
  └─ 在当前 assistant turn 与工具结果之后、下一次模型请求之前注入

followUp(msg)
  └─ 在当前 run 本来会停止时注入，并开启新一轮循环
```

`AgentSession` 在此之上增加了 `streamingBehavior` 参数、扩展输入处理、持久化队列展示等。

## 7. 默认 StreamFn 与代理

`stream-fn.ts`：

```ts
setDefaultStreamFn(streamFn);
getDefaultStreamFn();
```

这样 `pi-agent-core` 不需要依赖 Provider 目录；coding-agent 在 [sdk.ts](../packages/coding-agent/src/core/sdk.ts) 启动时把 `streamSimple` 注册为默认值。

`proxy.ts` 提供 `streamProxy(model, context, options)`：请求发往自建代理服务，由服务端持有 Provider 凭据；客户端根据精简 delta 事件重建 partial message。

## 8. 事件顺序示例

无工具调用：

```text
agent_start
turn_start
message_start(user)
message_end(user)
message_start(assistant)
message_update × n
message_end(assistant)
turn_end
agent_end
```

有工具调用：

```text
... assistant message_end
tool_execution_start
tool_execution_update × n
tool_execution_end
message_start(toolResult)
message_end(toolResult)
turn_end
turn_start                 ← 下一轮
...
agent_end
```

## 9. 依赖与使用方

- 依赖：`@earendil-works/pi-ai`、`typebox`。
- 主要使用方：`@earendil-works/pi-coding-agent`。
- 最小用法：

```ts
const agent = new Agent({
  initialState: { systemPrompt: "...", model },
  streamFn: models.streamSimple.bind(models),
});
agent.subscribe((event) => { ... });
await agent.prompt("Hello");
```

## 10. 测试

`packages/agent/test/` 覆盖循环、工具并发、hook、队列、代理流、e2e。测试不需要真实 Provider，可注入 faux stream。

```bash
cd packages/agent
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/agent.test.ts
```
