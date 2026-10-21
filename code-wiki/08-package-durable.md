# 08 `@earendil-works/pi-durable`

> 实验性：API 可能随版本变化。当前版本 `1.2.0`。

## 1. 职责

`pi-durable` 是一个持久化 Agent Harness：

- 会话、模型 turn、工具调用、文档状态在对外可见前先提交到存储。
- 进程中途死亡后，重新打开存储可从最后 checkpoint 继续。
- 提供任务调度器、任务图、提交/订阅、压缩、用量统计。
- 支持内存、JSONL、SQLite（Node/Cloudflare）存储。
- 通过 `ExecutionEnv` 抽象把工具运行在本地、容器或远端。

源码：[packages/durable/src](../packages/durable/src/)

## 2. 核心概念

| 概念 | 说明 |
|---|---|
| **Harness** | 打开存储并运行 Agent 的入口；所有变更走一条原子提交线 |
| **Conversation** | transcript；`root()` 首次创建根会话，可创建更多或 fork；句柄无状态，按 `id` 比较 |
| **Entry** | 不可变 transcript 记录：`pi.user`、`pi.assistant`、`pi.tool-result`、`pi.system`、`pi.reset` 或自定义 kind |
| **Commit** | 原子写：可同时追加 entry、修改文档、创建任务；全成功或全失败 |
| **Document** | 与 transcript 一起存储的类型化 JSON；内置 `pi.agent`、`pi.provider`、`pi.live`、`pi.inbox`、`pi.usage` |
| **Task** | 每一步保存 checkpoint 的持久状态机；有 owner（会话或另一个任务） |
| **Submission** | 提交给会话的输入或待写 entry，可等待结果 |
| **Turn / Run** | Turn 是一次模型响应 + 工具调用；Run 是从输入到最终回答 |
| **Extension** | 工具、系统提示 section、hook、wrap、task 的命名集合 |
| **Registry** | 当前进程安装的扩展；运行中可变 |
| **Agent** | 会话运行配置：模型、thinking、扩展、工具、指令、cwd，存于 `pi.agent` |

一次输入对应的 entry/task 流程：

```text
submit(input) → pi.user
  pi.generation → pi.system（仅当 prompt/tools 变化）, pi.assistant(tool calls)
    pi.tool × n → pi.tool-result × n（由 generation 拥有并等待）
  pi.generation → pi.assistant(answer) → submission done
```

## 3. Harness API

```ts
Harness.open(storage, options, context): Promise<Harness>
```

`options` 关键字段：

| 字段 | 说明 |
|---|---|
| `models` | `pi-ai` 的 `Models` 集合 |
| `registry` | 扩展注册表（必须包含内置 task，`createRegistry()` 会提供） |
| `settings` | 运行策略；每次使用读取，可 live |
| `env` | `ExecutionEnv` 工厂：`({ id, cwd }) => env` |
| `now` | 时钟 |
| `onReport` | 后台错误报告 |

### 3.1 会话与提交

| 方法 | 说明 |
|---|---|
| `root(context, init?)` | 获取/创建根会话 |
| `createConversation(options, context)` | 创建独立会话 |
| `conversation(id, context)` | 按 id 获取句柄 |
| `submission(id, context)` | 重新获取 submission（重启后等待） |
| `resume()` | 启动任务调度器，继续上次未完成 run |
| `inspect(context)` | 检查运行中的任务/会话 |
| `usage(context)` | 读取用量 |
| `close(context)` | 关闭 |

### 3.2 Conversation 句柄

| 方法 | 说明 |
|---|---|
| `agent(context)` | 解析当前 Agent 配置 |
| `configure(change, context)` | 在一个 commit 中修改模型/thinking/扩展/工具/指令/cwd |
| `submit(draft, context)` | 持久化接纳输入，返回 `Submission` |
| `commit(fn, context)` | 原子事务：追加 entry、改文档、创建任务 |
| `context(context, { at? })` | 读取模型上下文视图 |
| `entries(query, limit, cursor, context)` | 分页读取 entry |
| `fork(at, options, context)` | 从某个 entry fork |
| `reset(handoff?, context)` | 写入 reset entry，模型只看到 reset 之后 |
| `compact(instructions?, context)` | 触发压缩任务 |
| `abort(context, options?)` | 中止该会话的任务 |
| `waitForIdle(context)` | 等待会话空闲 |
| `viewState()` / `watch()` | 观察会话视图 |
| `Submission.wait()` / `abort()` | 等待/取消一次提交 |

### 3.3 幂等提交

同一 `requestId` 重复 `submit` 会返回已有 submission，不会重复执行：

```ts
const a = await root.submit({ type: "input", content: "Hello", requestId: "greeting-1" }, ctx);
const b = await root.submit({ type: "input", content: "Hello", requestId: "greeting-1" }, ctx);
// a.id === b.id
```

## 4. Session 内核

`SessionImpl`（[session/session.ts](../packages/durable/src/session/session.ts)）负责：

- **单一变更线**：所有 commit 回调、准备、Storage 结算、adoption、发布入队都在同一条线上串行执行；监听器稍后运行。
- **文档缓存**：按地址缓存已加载文档；没有观察者的文档在 idle 保留期后逐出；有观察者的不逐出。
- **只观察已提交状态**：提交成功前不对外发布。
- **Storage 失败是终态**：第一个非取消导致的 Storage 错误会让 Session 失败，后续调用返回 `SessionFailed`，关闭监听器立即以该错误运行。
- **事务作用域**：`Transaction` 提供 `entry()`、文档读写、任务创建等。

`createSession(storage, options?)` 可单独创建 Session 内核。

## 5. 任务系统

### 5.1 定义任务

```ts
const MyTask = defineTask({
  name: "my.task",
  input: MyInputSchema,
  output: MyOutputSchema,
  phases: [...],
  run: async (input, runtime) => { ... },
});
```

`defineTask()` 返回 `{ definition }`，注册进 registry 后 Harness 才能运行该类型任务。

### 5.2 调度器

`TaskScheduler`（[harness/scheduler.ts](../packages/durable/src/harness/scheduler.ts)）：

- 每个任务有 owner（会话或任务）与 checkpoint。
- 每一步提交 checkpoint；重启后从最后 checkpoint 继续。
- 支持等待、取消、父子任务、任务图。
- 默认 task 执行策略、重试、并发控制来自 `settings`。
- 内置任务：`pi.generation`、`pi.tool`、`pi.compaction`。

### 5.3 恢复语义

- 崩溃/关闭时仍在运行的任务保持 pending。
- `resume()`、`submit()`、`wait()` 都会启动调度器。
- `abandonOnRestart: true` 的任务在下次启动时被中止并清理子任务。
- 工具的 `replay: "safe"` 决定中断后是否可重跑；否则模型收到 `interrupted` 结果。

## 6. 扩展与工具

### 6.1 扩展

```ts
const Coding = defineExtension({
  name: "coding",
  sections: [section("preamble", () => "You are a concise coding assistant.", { tag: false })],
  hooks: [hook(ToolTask, { beforeTool: (call) => isDangerous(call) ? { block: "Needs approval" } : undefined })],
  tools: [readTool, bashTool],
  wraps: [wrapTool(bashTool, (tool) => timed(tool))],
});
```

扩展可包含：`tools`、`sections`、`hooks`、`wraps`、`tasks`。会话只存扩展名字，registry 存代码；扩展卸载后会话自动跳过，重装后恢复。

### 6.2 工具

```ts
const count = defineTool({
  name: "count",
  description: "Count from 1 to n",
  parameters: Type.Object({ n: Type.Number() }),
  execute: async (args, api) => {
    for (let i = 1; i <= args.n; i++) api.output(`${i}\n`);
    return {};
  },
});
```

- 每次调用是独立 durable task；intent 在 `execute()` 前提交。
- 结果三通道：`output`（给模型）、`structuredOutput`（给程序）、`details`（给 UI）。
- `api.output()` 流式输出；`api.executeTool()` 调用嵌套工具；`control.terminate` 可提前结束 run。
- 内置工具包 `@earendil-works/pi-durable/tools` 提供 `read/write/edit/bash` 与 `CodingTools` 扩展。

### 6.3 代码模式

`@earendil-works/pi-durable/codemode` 提供 `createCodemodeExtension()`：

- 模型写 JS，脚本内 `await tools.<name>(args)`。
- 每次调用是嵌套 tool task，只有脚本输出进入模型上下文。
- 脚本可恢复：虚拟时钟、种子随机、UTC；结果写 journal，重启后重放并重新附着嵌套调用。
- `store/load` 按会话保存 JSON。
- 需要 `CodemodeSandbox` 或 `RemoteCodemodeSandbox`。

## 7. 文档

- `defineDoc()` / `defineDocFamily()` 定义类型化文档。
- 文档是 JSON，参与原子提交；`tx.document()` 读写。
- 内置文档：
  - `pi.agent`：会话 Agent 配置。
  - `pi.provider`：Provider 会话身份（UUIDv7，用于 prompt cache/亲和）。
  - `pi.live`：运行中的 generation/tool 状态。
  - `pi.inbox`：排队输入。
  - `pi.usage`：token/成本。
- 文档观察者通过 `watch()`/`viewState()` 获得快照 + 增量（底层使用 Chord 的复制状态）。

## 8. 存储后端

| 后端 | 导入 | 说明 |
|---|---|---|
| Memory | `@earendil-works/pi-durable/storage/memory` | 测试/临时 |
| JSONL | `@earendil-works/pi-durable/storage/jsonl`、`/node` | 追加日志，便于检查 |
| SQLite | `@earendil-works/pi-durable/storage/sqlite`、`/node`、`/cloudflare` | 迁移、事务、索引 |
| 自定义 | 实现 `Storage` 接口 | 通过 `testing` 的 conformance 套件验证 |

`Storage` 负责：会话/entry/文档/任务/提交记录的读写、分页扫描、watch。SQLite 通过 `applySqliteMigrations()` 管理 schema 版本。

## 9. 执行环境

`ExecutionEnv` 在 [env/index.ts](../packages/durable/src/env/index.ts) 定义，组合：

- `FileSystem`：读文件、二进制 reader、目录 reader、listFiles、watch、write/mkdir/remove/rename。
- `Shell`：`exec()`（argv 或字符串）、流式输出、超时、输出窗口/跳过、spill 到大文件。

内置 `NodeExecutionEnv`（`@earendil-works/pi-durable/env/node`）。`pi-env` 提供远端实现。

## 10. 压缩

- `CompactionTask` 生成摘要并写 checkpoint。
- 压缩原因：`manual`、`threshold`、`overflow`。
- 策略（`CompactionPolicy`）：reserve tokens、保留最近 token、是否跳过 prompt、自定义指令。
- 压缩后模型只看到摘要 + 保留的最近 entry；原始 entry 保留。

## 11. 观察与事件

- `watchEvents()`：订阅 Agent 事件流（消息变化、工具事件、快照）。
- `ConversationWatch` / `ConversationView`：读取会话视图。
- `TaskGraphView`：任务图节点、状态、父子关系。
- 所有异步 API 接收 Chord `Context`；取消等待不会取消后台工作。

## 12. 错误

| 错误 | 含义 |
|---|---|
| `ConversationBusy` | 会话正在运行，不能并发提交 |
| `ReadAfterWrite` | 同一事务中读到自己尚未提交的写 |
| `SessionFailed` | Storage 首次失败后 Session 进入终态 |
| `StorageRequestError` | 单次存储请求失败 |

## 13. 依赖与使用方

依赖：`@earendil-works/chord`、`@earendil-works/pi-ai`、`@earendil-works/pi-codemode`、`@silvia-odwyer/photon`（图片）、`diff`、`typebox`。

使用方：`@earendil-works/pi-env`（远端环境）、需要服务端持久化 Agent 的应用。

## 14. 测试与基准

`packages/durable/test/` 覆盖 Harness、调度器、任务图、文档、恢复、存储一致性、工具、代码模式、并发/内存基准。

```bash
cd packages/durable
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/harness-tasks.test.ts
npm run bench:storage
```
