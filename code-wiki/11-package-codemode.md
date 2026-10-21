# 11 `@earendil-works/pi-codemode`

## 1. 职责

`pi-codemode` 在 QuickJS（编译为 WebAssembly）沙箱中执行模型生成的 JavaScript：

- 脚本唯一能力是调用注入的工具。
- 嵌套工具调用不进入 LLM 上下文，只有脚本输出与返回值进入。
- 同一套 API 支持 Node/Bun、Cloudflare Workers、浏览器/其他 WASM 运行时。
- 无 Pi 依赖，可单独用于把任意函数/API 暴露给模型脚本。

源码：[packages/codemode/src](../packages/codemode/src/)

## 2. 三种沙箱

| 沙箱 | 导入 | VM 运行位置 | 适用 |
|---|---|---|---|
| `CodemodeSandbox` | `@earendil-works/pi-codemode` | 每次执行一个 worker thread | Node/Bun |
| `RemoteCodemodeSandbox` | `/portable` | 另一个 isolate（通常是独立 Worker 的 Durable Object） | Cloudflare |
| `InlineCodemodeSandbox` | `/portable` | 调用线程 | 其他 WASM 运行时，或可接受阻塞 |

三者共享脚本环境与 `CodemodeResult` 格式。

Cloudflare 入口 `/cloudflare` 额外提供可部署的 Durable Object 与 Wrangler 编译的 QuickJS 模块。

## 3. 快速使用

```ts
import { CodemodeSandbox } from "@earendil-works/pi-codemode";

const sandbox = new CodemodeSandbox({
  timeoutMs: 60_000,
  tools: [{
    name: "read",
    execute: async (args, { signal }) => {
      const { path } = args as { path: string };
      return await readFile(path, "utf8");
    },
  }],
});

const result = await sandbox.execute(`
  const source = await tools.read({ path: "package.json" });
  text("bytes " + source.length);
  return JSON.parse(source).name;
`);

console.log(result.output); // [{ type: "text", text: "bytes 1234" }]
if (result.ok) console.log(result.value);
else console.error(result.error.kind, result.error.message);

await sandbox.close();
```

## 4. 脚本环境

`code` 是一个 async 函数体，支持 `return` 与顶层 `await`。

| 全局 | 说明 |
|---|---|
| `tools.<name>(args)` | 调用工具，返回 Promise；参数/结果 JSON 往返；抛错转成 `Error`（保留 message）。非法标识符字符会转成 `_`，因此 `my-tool` 可用 `tools.my_tool` 或 `tools["my-tool"]` |
| `ALL_TOOLS` | `{ name, description }[]`，name 为标识符形式 |
| `text(value)` | 向 `result.output` 追加文本；非字符串 JSON.stringify |
| `console.log/info/warn/error/debug` | 追加带 `console: true` 的文本项 |
| `image(urlOrItem)` | 追加图片；接受 base64 data URL、`{ image_url }`、图片块；拒绝远端 URL |
| `exit()` | 立即成功结束，保留输出与 store 写入 |
| `globals` | 宿主注入的顶层函数；行为类似工具但不记录到 `result.calls`；可用 `models.classify` 形式挂到冻结对象 |
| `store(key, value)` / `load(key)` | 同步读写 JSON；`load` 返回副本；`undefined` 删除 key |

沙箱没有 timers、`fetch`、`process`、`require`、模块、`WebAssembly`。`eval`/`Function` 仍可在同一 VM 内产生代码。

## 5. 结果模型

`CodemodeResult`：

```ts
type CodemodeResult =
  | { ok: true; value: JsonValue; output: CodemodeOutputItem[]; calls: CodemodeCall[]; storeWrites: CodemodeStoreWrites }
  | { ok: false; error: CodemodeError; output: ...; calls: ... };
```

- `output`：`text` / `image` / `console` 项。
- `calls`：工具调用记录（名称、参数、状态、耗时等）。
- `CodemodeCallStatus`：`ok | error | cancelled`。
- `CodemodeErrorKind`：语法、超时、内存、中止、工具错误等。

## 6. 关键 API

| API | 说明 |
|---|---|
| `CodemodeSandbox` | Node/Bun 沙箱，worker thread 隔离 |
| `InlineCodemodeSandbox` | 同线程沙箱 |
| `RemoteCodemodeSandbox` | 远端 isolate 沙箱 |
| `serveCodemodeRemote()` | 在远端 Worker 中提供沙箱服务 |
| `CodemodeSandboxBase` | 三种沙箱的共享基类 |
| `execute(code, options?)` | 执行一次 |
| `start(code, options)` | 流式运行句柄（输出/调用增量） |
| `registerTool()` / `unregisterTool()` | 动态管理工具 |
| `close()` | 关闭沙箱 |
| `parseCodemodeSource()` | 解析可选 `// @options:` 首行 |
| `CODEMODE_SOURCE_GRAMMAR` | 语法约束工具的 Lark grammar |
| `renderDeclarations()` | 把工具 schema 渲染成 TypeScript 声明（供系统提示使用） |
| `renderToolSignature()` / `renderToolSample()` / `renderToolOutputType()` | 声明片段 |
| `schemaToType()` | JSON Schema → TS 类型 |
| `toCodemodeIdentifier()` | 工具名 → 合法标识符 |
| `loadQuickJSWasm()` | 加载/缓存 WASM |

## 7. 资源限制

| 限制 | 说明 |
|---|---|
| `timeoutMs` | 默认硬超时；`Infinity` 关闭 |
| `memoryLimitBytes` | VM 堆上限，超出报 `InternalError: out of memory` |
| `MAX_OUTPUT_CHARS` / `MAX_OUTPUT_ITEMS` | 输出上限 |
| `MAX_STORE_VALUE_CHARS`（256 Ki） / `MAX_STORE_TOTAL_CHARS`（1 Mi） | store 上限 |
| `DEFAULT_INTERRUPT_BUDGET` | 中止检查预算 |

超时/中止/`close()` 会请求 worker 终止但不等待退出；在 Bun 上某些内置函数可能不轮询中断。

## 8. Store

```ts
const result = await sandbox.execute(`store("runs", (load("runs") ?? 0) + 1)`, { store: saved });
if (result.ok) {
  for (const key of result.storeWrites.delete) delete saved[key];
  Object.assign(saved, result.storeWrites.set);
}
```

沙箱不持久化；调用方传入当前 `store`，成功后通过 `storeWrites` 应用变更。失败执行不上报写入。

## 9. 源码格式

```js
// @options: {"max_output_tokens": 2000, "timeout_ms": 30000}
const source = await tools.read({ path: "package.json" });
text(JSON.parse(source).name);
```

- 支持 `max_output_tokens`、`timeout_ms`。
- 沙箱不解释这两个字段，由调用方决定。
- 选项行被替换为空行，保持行号。
- 非法 JSON、未知字段、非首行、空输入 → `CodemodeSourceError`。

## 10. Worker 生命周期与并发

- `onWorkerStarted(exited)`：worker 实际启动/退出回调，可用于并发槽位。
- `acquireWorker()`：在 WASM 加载后、worker 创建前同步调用；抛错拒绝执行，返回释放回调。
- `close()` 不等待 worker 退出。

## 11. 打包宿主

打包后默认 WASM 与 worker 文件可能不在磁盘，需要显式传入：

```ts
const sandbox = new CodemodeSandbox({
  tools,
  wasm: loadQuickJSWasm(pathToQuickJSWasm),
  workerUrl: new URL("./codemode-worker.js", import.meta.url),
});
```

Bun 编译二进制可把 worker 作为额外入口，并传相对源码路径字符串。

## 12. 在 Pi 中

- coding-agent 内置 `codemode` 扩展使用 `CodemodeSandbox`。
- durable 的 `/codemode` 提供可恢复的 `codemode` 工具与 `createCodemodeExtension()`。
- 工具可通过 `exposure: "codemode"` / `"deferred"` 只对脚本可见。
- 结构化结果通过 `structuredContent` 返回给脚本。

## 13. 依赖与测试

依赖：`quickjs-wasi`。

```bash
cd packages/codemode
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/sandbox.test.ts
```

测试覆盖沙箱隔离、确定性、源码解析、声明生成、worker 生命周期。
