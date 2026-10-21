# 09 `@earendil-works/chord`

## 1. 职责

Chord 是一个“对象 + 引用 + 契约”的分布式运行时：

- 应用由多个进程（server、worker、浏览器 tab）中的对象组成。
- 对象按地址创建，在有人引用时存活。
- 调用方拿到的是引用（`Ref`），本地与跨进程语义一致，且能跨热重载存活。
- 每个对象知道自己的 principal（owner 与 guest 看到不同对象/方法）。
- 状态是可观察 JSON：远端订阅者先收快照，再收增量。
- 热重载在运行时替换代码：进行中的工作用旧代码完成，状态保留，不兼容变更在安全点迁移。

源码：[packages/chord/src](../packages/chord/src/)

## 2. 核心概念

| 概念 | 说明 |
|---|---|
| **Contract** | 运行时值：名字、版本、输入 schema、成员（状态/方法/opaque）。TypeScript 类型由它推导 |
| **Remote type** | 可跨宿主访问的类型，必须声明 `host` placement |
| **Local type** | 仅进程内，成员可以是方法、opaque 原生对象、状态 schema |
| **Variant** | 同一类型的不同 principal 视图（如 viewer/participant） |
| **Host** | 运行对象、serve 类型、管理引用的进程 |
| **Facet** | 插件在某个宿主上的代码；一个模块 URL 对应一个 facet |
| **Object** | 地址上的状态 + 代码；按需创建，引用归零后回收 |
| **Ref** | 对对象的引用；跨进程、可跨重载 |
| **Principal** | 调用者身份（JSON），决定对象变体与权限 |
| **Registry** | 对象注册的键值表，按 code version 隔离 |
| **Link** | 双向 FIFO JSON 帧通道（memory/IPC/WebSocket） |
| **ReplicatedState** | 可观察的不可变 JSON，支持 `change/subscribe`，或附着到外部 revision source |

## 3. 契约 API

```ts
import { Type } from "typebox";
import { hostFamily, hostNamed, local, method, opaque, remote } from "@earendil-works/chord";

const Counter = remote("demo.counter", {
  version: 1,
  input: Type.Object({ name: Type.String() }),
  host: hostNamed("server"),
  variants: {
    default: {
      state: { count: Type.Number() },
      methods: { increment: method(Type.Object({ by: Type.Number() })) },
    },
  },
});
```

| API | 说明 |
|---|---|
| `remote(name, def)` | 远程契约 |
| `local(name, def)` | 本地契约 |
| `method()` / `method(args)` / `method(args, result)` | 方法签名 |
| `opaque(name@version)` | 原生进程内对象 |
| `hostNamed(name)` | 对象固定在某个宿主 |
| `hostFamily(prefix, key)` | 对象放在 `prefix + input[key]` 命名的宿主 |
| `resolveHost(typeName, placement, input)` | 计算对象宿主 |
| `placedOn(placement, hostName)` | 宿主是否可服务该类型 |

契约附带指纹；跨进程时以 description 传输并校验。

## 4. Host

```ts
const host = createHost({ name: "server", self: { kind: "server" } });

host.serve(Counter, ({ name }, principal, self) => {
  const count = self.state("count", Type.Number(), 0);
  return {
    variant: "default",
    count,
    increment: async ({ by }, ctx) => count.change(ctx, (n) => n + by),
  };
});

await host.root(async (ctx) => {
  const ref = await ctx.get(Counter, { name: "clicks" });
  ref.count.subscribe((n) => console.log(n));
  await ref.increment({ by: 2 }, ctx);
});
```

### 4.1 Host API

| 方法 | 说明 |
|---|---|
| `serve(type, factory, options?)` | 注册类型工厂 |
| `get(type, input, ctx?, options?)` | 获取已就绪引用 |
| `hold(type, input, options?)` | 获取引用但不等待就绪 |
| `root(fn, options?)` | 启动 root 作用域；root 结束时释放其引用 |
| `registry(registry)` | 读取注册表视图 |
| `install(facet, options?)` | 安装 facet |
| `load(plugins, options?)` | 加载 facet 模块/对象 |
| `reload(plugins, options?)` | 热重载 |
| `ready()` | 等待所有已安装 facet 就绪 |
| `connect(link, options)` | 连接父宿主 |
| `accept(link, options)` | 接受入站连接 |
| `route(prefix, route)` | 把 `prefix...` 的宿主请求路由到子宿主 |
| `invalidatePrincipal(match)` | 使匹配的 principal 失效 |
| `close()` | 关闭宿主 |

### 4.2 Host 选项

`name`、`self`（principal）、`principalKey`、`dev`（schema 校验）、`onError`、`taskStopDeadline`、`readyTimeout`、`linger`、`loader`、`kind`（host/client）、`upstream`。

### 4.3 Root 与生命周期

- `root(fn)` 内获取的引用在 root 结束时自动释放。
- 对象在引用归零后可回收；`serve(..., { linger })` 可让对象在最后一个引用释放后保留一段时间。
- `lifetime: "host"` 的对象随宿主存活。
- 任务（`self.task`）在被中止或对象回收时取消。

## 5. 状态与复制

### 5.1 Cell

对象状态成员是 cell：

```ts
const count = self.state("count", Type.Number(), 0);
count.get(ctx);
count.change(ctx, (n) => n + 1);
count.subscribe((value, ctx) => { ... });
```

cell 读写经过契约 schema 校验（`dev` 模式），并产生增量。

### 5.2 `replicatedState`

```ts
const status = replicatedState({ output: "", count: 0 });
status.change(ctx, (draft) => { draft.count += 1; });
status.subscribe((value, ctx, delivery) => { ... });
```

两种模式：

- `replicatedState(initial)`：权威可变状态，`change()` 原子发布一个 revision。
- `replicatedState(source, { onError })`：只发布外部 `ReplicatedStateSource` 的已提交帧；先快照，再按 cursor 顺序发布。

订阅语义：

- 每个订阅独立串行；先 hydration，再 updates。
- 最多 100 个待处理完整值；溢出只保留最新值，因此序号可能跳跃。
- 回调异常被隔离，不影响其他订阅。
- 取消订阅立即丢弃待处理工作，但不中止正在运行的回调。

### 5.3 Delta

`@earendil-works/chord/delta` 是独立的事务 JSON tracker：

```ts
import { applyImmutable, track } from "@earendil-works/chord/delta";

const tracker = track({ output: "", count: 0 });
const change = tracker.beginChange();
change.state.output += "done\n";
change.state.count += 1;
const prepared = change.prepare();

tracker.adopt(prepared);
const replica = applyImmutable(prepared.base, prepared.ops);
```

- 所有权即不可变契约：不冻结、不防御性复制。
- `track()` / `prepareReplace()` / `replicatedState()` / `replace()` 以 O(1) 接管无别名 strict-JSON 根。
- 通过 draft 放入的值会被克隆并校验。
- 支持字符串 append/front-truncate、数组 splice/置换、set/delete；大量编辑可能折叠为整体替换。
- 提供 `applyImmutableBatches()` 共享 copy-on-write 作用域，以及可变 `apply()`（每个 replica 需要独立副本）。

## 6. 链接与协议

| API | 说明 |
|---|---|
| `memoryLinkPair()` | 同进程双向 link |
| `ipcLink(channel)` | Node IPC |
| `webSocketLink(ws, options?)` | WebSocket 客户端/对端 |
| `acceptWebSockets(server, options?)` | WebSocket 服务端接受 |
| `PACK_THRESHOLD` / `inflateMessage()` | 大帧压缩 |
| `PROTOCOL_VERSION` / `Frame` | wire 协议 |

Link 语义：有序 FIFO、单接收者、`bye` 关闭、可选帧大小用于流控。

`createHostManager`、`forkDriver`、`childControl`、`processLock` 在 `@earendil-works/chord/node`，用于子宿主进程与 facet 模块加载。

## 7. Facet、插件与热重载

```ts
const ChatFacet = defineFacet((api) => {
  api.serve(Chat, factory);
}, { uses: [Counter] });
```

- `defineFacet(setup, { uses })` 返回 facet 定义；`uses` 在重载时与提供者校验。
- `FacetApi` 提供：`serve`、`name`、`root`、`registry`、`signal`（facet 被替换/移除时中止）。
- `PluginRef` 可以是 facet 对象或模块路径/URL。
- `host.load([...])` 每次导入新副本并按模块 URL 命名。
- `host.reload([...])`：
  - 准备新代码版本。
  - 进行中的工作继续用旧代码完成。
  - 状态保留。
  - 不兼容变更在安全点迁移；无法迁移则报告。
- 旧 code version 在所有引用/任务结束后退休。

## 8. Principal 与变体

- 宿主自身 principal 由 `self` 指定。
- 父宿主请求的 principal 由 `connect({ principal })` 指定。
- 入站连接 principal 由 `accept(link, { principal })` 决定。
- 路由宿主的 principal 由 `route(prefix, { principal })` 决定。
- `principalKey()` 把 principal 映射为对象键（应包含连接来源与 grant 版本）。
- 同一类型可按 principal 返回不同 variant，从而暴露不同方法/状态。

## 9. Registry

```ts
const Tools = defineRegistry<Tool>("tools", { conflict: "last" });
const view = host.registry(Tools);
view.get("read", ctx);
```

- 注册表按 code version 维护表。
- `conflict: "error"` 时重复键导致错误，`"last"` 保留后注册。
- 对象可注册声明，宿主在提交时构建新表；注册表变更可订阅。

## 10. 模块结构

| 文件 | 职责 |
|---|---|
| [contract.ts](../packages/chord/src/contract.ts) | 契约、placement、description schema |
| [host.ts](../packages/chord/src/host.ts) | 宿主、serve、root、registry、连接、重载 |
| [object.ts](../packages/chord/src/object.ts) | 对象生命周期、引用、服务、context |
| [connection.ts](../packages/chord/src/connection.ts) | 帧路由、请求/响应关联、流控 |
| [remote.ts](../packages/chord/src/remote.ts) | 远端绑定与引用 |
| [state.ts](../packages/chord/src/state.ts) | 复制状态与订阅 |
| [delta/](../packages/chord/src/delta/) | tracker、diff、apply、wire 编码 |
| [link.ts](../packages/chord/src/link.ts)、[websocket.ts](../packages/chord/src/websocket.ts)、[wire.ts](../packages/chord/src/wire.ts) | 传输与帧 |
| [cluster.ts](../packages/chord/src/cluster.ts) | 子宿主集群与重载协调 |
| [registry.ts](../packages/chord/src/registry.ts) | 注册表 |
| [reload.ts](../packages/chord/src/reload.ts) | 重载流程 |
| [loader.ts](../packages/chord/src/loader.ts)、[node.ts](../packages/chord/src/node.ts) | facet 加载、Node 入口 |
| [schema.ts](../packages/chord/src/schema.ts) | 规范化、哈希、深度比较 |

## 11. 错误

- `ChordError` 带 code；`isChordError()` 判断。
- `forbidden()`、`invalid()` 构造常见错误。
- 常见 code：`invalid`、`forbidden`、`unavailable`、`ready_timeout`、`placement`、`protocol`。

## 12. 依赖与使用方

依赖：仅 `typebox`（开发依赖 `ws`）。

使用方：`@earendil-works/pi-durable`、`@earendil-works/pi-env`。

## 13. 测试

`packages/chord/test/` 覆盖契约、绑定、边界、集群、上下文、增量、流、host、hydration、link、manager、placement、principal、重载、请求、状态、WebSocket 等。

```bash
cd packages/chord
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/host.test.ts
```
