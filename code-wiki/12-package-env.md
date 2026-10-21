# 12 `@earendil-works/pi-env`

## 1. 职责

`pi-env` 把 Durable 的 `ExecutionEnv` 放到远端机器上：

- Agent 的工具在远端执行文件操作与命令。
- Durable worker、存储、凭据仍留在本地。
- 通过 SSH 部署并启动一个 Rust daemon，使用 stdin/stdout 的 framed protocol 通信。
- 同时提供本机模式（不走 SSH）与远程 PTY。

源码：[packages/env/src](../packages/env/src/)，daemon：[packages/env/daemon](../packages/env/daemon/)

## 2. 架构

```text
Durable Harness
  └─ RemoteExecutionEnv (TypeScript)
       └─ Connection
            └─ ssh 子进程
                 └─ pi-env daemon (Rust)
                      ├─ 文件操作
                      ├─ 命令执行
                      ├─ 文件监听
                      ├─ listFiles
                      └─ PTY
```

- daemon 按 SHA-256 命名，启动前校验。
- 每次 SSH 使用 `BatchMode`、严格 host key 校验、应用自有 known_hosts、固定 alias、禁用转发/共享连接/配置命令。
- Windows 远端通过 PowerShell 检测/部署，daemon 由默认 shell 启动。

## 3. 快速使用

```ts
import { acceptHostKey, connectSsh, RemoteExecutionEnv, scanHostKey } from "@earendil-works/pi-env";

const target = {
  host: "gpu-box",
  knownHostsFile: "/data/ssh/known_hosts",
  hostKeyAlias: "pi-env-gpu",
};

const { lines, fingerprints } = await scanHostKey(target);
await acceptHostKey(target, lines);

const { connection } = await connectSsh(target);
const env = new RemoteExecutionEnv({
  connection,
  id: "pi-env:gpu",
  cwd: "/home/me/project",
});
```

懒连接：

```ts
const lazy = sshConnection(target);
const env = new RemoteExecutionEnv({
  connection: lazy.connection,
  id: "pi-env:gpu",
  cwd: "/home/me/project",
});
```

首次操作时才检测/部署/连接；失败作为该操作的错误，下次重试。

## 4. 主要 API

| API | 说明 |
|---|---|
| `connectSsh(target)` | 检测平台、按需部署 daemon、返回 `{ connection, platform, ... }` |
| `sshConnection(target)` | 懒连接版本 |
| `scanHostKey(target)` | 获取 host key 行与指纹 |
| `acceptHostKey(target, lines)` | 写入 known_hosts |
| `forgetHostKey(target)` | 删除记录 |
| `HostKeyUnknownError` / `HostKeyChangedError` / `SshError` | SSH 错误 |
| `RemoteExecutionEnv` | 远端 `ExecutionEnv` 实现 |
| `LocalExecutionEnv` | `NodeExecutionEnv` + 本机 daemon 的 `listFiles` |
| `Connection` | framed 协议连接；`info()`、`session()`、`request()`、`close()` |
| `nodePlatform(RemoteInfo)` | 远端平台/架构转 Node 名称 |
| `RemoteWatcher` | 远端文件监听 |
| `openTerminal(connection, options)` | 远端 PTY |
| `RemoteTerminal` | PTY 会话：原始输出、输入、resize、退出码 |
| `localDaemon()` / `packagedDaemon()` | 本机/包内 daemon 路径 |
| `detectPlatform()` / `deployDaemon()` | 平台检测与部署 |

## 5. ExecutionEnv 能力

`RemoteExecutionEnv` 实现 Durable 的 `ExecutionEnv`：

### FileSystem

- `readFile` / `writeFile` / `stat` / `lstat`
- `openBinaryReader()`：按字节范围并发预读
- `openDirReader()`：分页目录
- `mkdir` / `remove` / `rename` / `copy`
- `watch()`：远端 watcher
- `listFiles()`：可选，daemon 支持时返回递归列表

### Shell

- `exec()`：argv 或字符串命令；流式 stdout/stderr；超时；输出窗口/跳过；spill 文件。
- Windows 上字符串命令经 Git Bash，argv 命令直接执行。
- 结果与本地 `NodeExecutionEnv` 一致；错误消息可能不同。

### 路径规则

`remotePath(connection)` 根据远端平台返回 `win32` 或 `posix` 规则。

## 6. listFiles

```ts
const listing = await env.listFiles?.(".", { exclude: ["node_modules"], maxEntries: 50_000 }, ctx);
```

行为类似 `fd --hidden --exclude .git`：

- 不跟随符号链接。
- 工作树内跳过 git ignore。
- 返回相对 `/` 路径与 `file/directory/symlink`。
- 受 `maxEntries`、`maxBytes`、`timeoutMs` 限制，返回 `truncated`。
- 旧 daemon 不支持时返回 `not_supported`，调用方回退到自己的遍历。

## 7. 终端

```ts
const terminal = await openTerminal(connection, { command, args, cwd, env, cols, rows });
terminal.onData(...); terminal.write(...); terminal.resize(...);
```

- 默认启动用户登录 shell。
- 原始输出、输入、resize、退出码。
- 本机可用 `new Connection({ command: [localDaemon()] })` 避免 SSH。
- Windows 暂不支持 PTY。

## 8. 文件监听

`RemoteWatcher` 包装 daemon 的 watch：

- `WatchTarget`：路径、递归、忽略规则。
- `WatchChange`：`{ paths }`、`{ overflow: true }`、`{ error }`。
- 支持取消、背压、重连。

## 9. 协议

- framed 二进制协议，见 [docs/protocol.md](../packages/env/docs/protocol.md)。
- 语义一致性见 [docs/semantics.md](../packages/env/docs/semantics.md)。
- `Connection.request(op, json, options)` 是主要 RPC 入口。
- `Connection` 负责子进程、帧读取、请求超时/取消、session/kill。

## 10. 平台支持

Linux、macOS、Android（Termux）、Windows，x86-64 与 arm64。

## 11. 依赖与构建

依赖：`@earendil-works/chord`、`@earendil-works/pi-durable`。

```bash
npm run build:daemon --workspace packages/env   # Cargo 构建 daemon
```

包内 `bin/` 随发布提供各平台 daemon。

## 12. 测试

- 通过管道与 `daemon/target/debug/pi-env` 通信。
- 或使用 `PI_ENV_DAEMON` 指定二进制。
- 运行 Durable 的 env conformance 套件。
- 与 `NodeExecutionEnv` 做随机操作序列与工具差分比较。
- 设置 `PI_ENV_SSH_HOST` 时运行真实 SSH 测试。

```bash
cd packages/env
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/remote.test.ts
```
