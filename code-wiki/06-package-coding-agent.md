# 06 `@earendil-works/pi-coding-agent`

## 1. 职责

`pi-coding-agent` 是用户可见的 Pi CLI，也是把所有库粘合起来的应用层：

- CLI 参数、鉴权、配置、包管理、MCP 子命令。
- 会话生命周期、模型/思考级别、工具集、压缩、重试、Bash 执行。
- 三种运行模式：交互 TUI、Print/JSON、RPC。
- 扩展系统与资源加载（extensions/skills/prompts/themes/packages）。
- TypeScript SDK：`createAgentSession()`。
- HTML/JSONL 会话导出、分享、Bug report。

源码：[packages/coding-agent/src](../packages/coding-agent/src/)

## 2. 代码结构

```text
src/
├─ cli.ts                    # bin 入口
├─ rpc-entry.ts              # RPC 专用入口
├─ main.ts                   # CLI 主流程
├─ config.ts                 # 路径、版本、安装方式、资源定位
├─ migrations.ts             # 配置/凭据迁移
├─ package-manager-cli.ts    # pi package/config 子命令
├─ cli/                      # 参数解析、鉴权命令、文件参数、会话选择、首次设置
├─ core/
│  ├─ agent-session.ts       # 会话门面（最大核心文件之一）
│  ├─ agent-session-runtime.ts
│  ├─ agent-session-services.ts
│  ├─ sdk.ts                 # createAgentSession
│  ├─ session-manager.ts     # JSONL 会话树
│  ├─ settings-manager.ts
│  ├─ model-runtime.ts / model-registry.ts / model-resolver.ts
│  ├─ resource-loader.ts
│  ├─ extensions/            # 扩展加载、runner、类型、wrapper
│  ├─ tools/                 # read/bash/edit/write/grep/find/ls/powershell
│  ├─ compaction/            # 上下文压缩与分支摘要
│  ├─ system-prompt.ts
│  ├─ messages.ts
│  ├─ auth-storage.ts
│  ├─ cache-warmer.ts
│  ├─ trust-manager.ts / project-trust.ts
│  ├─ package-manager.ts
│  ├─ mcp-servers.ts
│  ├─ virtual-models.ts
│  ├─ event-bus.ts
│  ├─ nested-tool-calls.ts
│  └─ export-html/
├─ modes/
│  ├─ interactive/           # TUI 模式、渲染器、主题、组件
│  ├─ rpc/                   # RPC 模式、客户端、JSONL
│  ├─ print-mode.ts
│  └─ json-event.ts
├─ extensions/               # 内置扩展：codemode / mcp / tool-search / llama
└─ utils/                    # 图片、shell、路径、文本、剪贴板等
```

## 3. 启动流程（`main.ts`）

```text
cli.ts
  ├─ setupCli()                  # 终端/信号/环境初始化
  └─ main(args)
       ├─ 认证子命令（pi auth ...）
       ├─ Windows 自更新隔离清理 / managed install 清理
       ├─ SettingsManager 引导 + HTTP proxy 配置
       ├─ pi install/remove/update/list、pi config、pi mcp 子命令
       ├─ parseArgs
       ├─ --version / --export
       ├─ 计算运行模式（interactive/print/json/rpc）
       ├─ migrations、首次设置、主题
       ├─ 选择 SessionManager（新建/继续/恢复/fork/指定 id）
       ├─ ProjectTrustStore + 项目信任解析
       ├─ createAgentSessionRuntime(...)
       ├─ 诊断输出、模型准备
       └─ runRpcMode / InteractiveMode / runPrintMode
```

模式判定 `resolveAppMode()`：

- `--mode rpc` → rpc。
- `--mode json` → json。
- `-p`、stdin 非 TTY、stdout 非 TTY → print。
- 否则 interactive。

关键文件：[main.ts](../packages/coding-agent/src/main.ts)、[cli/args.ts](../packages/coding-agent/src/cli/args.ts)、[config.ts](../packages/coding-agent/src/config.ts)。

## 4. SDK 与服务创建

### 4.1 两个阶段

`createAgentSession()`（[sdk.ts](../packages/coding-agent/src/core/sdk.ts)）既可以一步创建，也可以拆分：

1. `createAgentSessionServices()`：创建 cwd 绑定服务。
   - `ModelRuntime`（`auth.json` + `models.json`）。
   - `SettingsManager`（全局 + 项目设置合并）。
   - `DefaultResourceLoader`（扩展/技能/模板/主题/context files）。
   - 注册扩展声明的 Provider、原生 Provider、虚拟模型。
   - 返回 `diagnostics`，由调用方决定如何展示。
2. `createAgentSessionFromServices()`：在这些服务上解析模型/思考级别/工具，再创建 `AgentSession`。

### 4.2 `createAgentSession()` 内部

| 步骤 | 行为 |
|---|---|
| 模型恢复 | 从活动分支的 `model_change` 恢复；不可用则回退并给出 `modelFallbackMessage` |
| 初始模型 | `findInitialModel()`：设置默认 → Provider 默认 |
| 思考级别 | 会话恢复 → 每模型设置 → 全局默认 → `clampThinkingLevel()` |
| 工具选择 | `tools` 白名单/模式；`+name/-name` 修饰符；`excludeTools` 黑名单；`noTools` |
| `convertToLlm` | `convertToLlm` + 可选的图片屏蔽（`blockImages`） |
| 流函数 | 包装 `modelRuntime.streamSimple()`；注入重试/超时/headers/缓存预热/扩展 hook |
| Agent 创建 | `new Agent({ initialState, convertToLlm, streamFn, ... })` |
| AgentSession | 传入 sessionManager、settingsManager、resourceLoader、工具选择、扩展 runner ref |
| 初始会话 entry | 新会话写入 `model_change` 与 `thinking_level_change` |

`streamFn` 包装负责：

- `buildRequestOptions()`：超时、重试、WebSocket 连接超时、Provider attribution headers、`before_provider_headers` 扩展。
- `transformProviderPayload`：`before_provider_request` 扩展。
- `handleProviderResponse`：`after_provider_response` 扩展。
- `handleProviderStreamEvent`：`provider_stream_event` 扩展。
- `CacheWarmer`：对当前会话的请求保持 prompt cache 热度。

## 5. `AgentSession`

`AgentSession`（[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)）是所有运行模式共享的核心。

### 5.1 职责

- 代理 `Agent` 状态与方法：`prompt/steer/followUp/abort/waitForIdle`。
- 把 Agent 事件持久化到 `SessionManager`，并转换成 `AgentSessionEvent`。
- 模型与思考级别切换、循环（`cycleModel`/`cycleThinkingLevel`）。
- 手动/自动压缩、分支摘要、上下文溢出恢复。
- 自动重试与取消。
- Bash 执行（用户 `!` 命令）与结果记录。
- 工具注册表与 loadout（active/callable/exposure）。
- 扩展绑定与事件转发。
- 会话统计、HTML/JSONL 导出、Bug report 摘要、树导航。

### 5.2 重要方法

| 方法 | 说明 |
|---|---|
| `prompt(text, options?)` | 展开扩展命令/技能/模板；运行中则按 `streamingBehavior` 排队；否则启动 run |
| `sendUserMessage()` | 发送用户消息并触发 turn |
| `steer()` / `followUp()` | 排队输入；返回 `"queued"` 或 `"handled"` |
| `abort()` | 中止当前操作并等待 idle |
| `waitForIdle()` | 不中止，等待空闲 |
| `setModel()` / `cycleModel()` | 切换模型；可持久化为默认 |
| `setThinkingLevel()` / `cycleThinkingLevel()` | 切换思考级别 |
| `compact()` | 手动压缩 |
| `setAutoCompactionEnabled()` / `abortCompaction()` | 自动压缩控制 |
| `executeBash()` / `abortBash()` / `recordBashResult()` | 用户 Bash |
| `setSessionName()` / `navigateTree()` / `getUserMessagesForForking()` | 会话元数据与树导航 |
| `getSessionStats()` / `getContextUsage()` | 统计 |
| `exportToHtml()` / `exportToJsonl()` | 导出 |
| `bindExtensions()` / `reload()` | 扩展绑定与资源重载 |
| `getActiveToolNames()` / `setActiveToolsByName()` / `getAllTools()` | 工具 loadout |
| `dispose()` | 中止、解绑、移除监听 |
| `subscribe()` | 订阅 `AgentSessionEvent` |

### 5.3 run 流程

```text
_startRun(trigger)
  ├─ _prepareRun()
  │    ├─ 必要时压缩
  │    ├─ emit before_agent_start
  │    ├─ 归一化图片
  │    ├─ 合并 pending next-turn / 扩展消息
  │    └─ 生成 system prompt + tool loadout 更新
  ├─ agent.prompt(messages)
  └─ while (!abortRequested)
       ├─ _handlePostAgentRun()：重试/溢出恢复/排队
       ├─ _runBeforeSettleBoundary()：扩展 agent_before_settle
       └─ agent.continue()
```

结束时 `_emitAgentSettled()`。

### 5.4 工具 loadout

- 每个工具定义有 `exposure`（`direct`、`codemode`、`deferred`、`hidden` 等）、`defaultActive`、`executionMode`。
- `getActiveToolNames()` 是声明给模型的工具；`getCallableToolNames()` 还包括 codemode/tool_search 可调用的工具。
- 工具增删通过 system message 的 `toolsAdded/toolsRemoved` 告知模型。
- `prepareLoadout()` 可隐藏声明或改写描述，但工具仍可调用。

## 6. `AgentSessionRuntime`

[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)

职责：持有当前 `AgentSession` + cwd 绑定服务 + 重建工厂。

| 方法 | 说明 |
|---|---|
| `switchSession(path, options?)` | 打开另一个会话，销毁旧运行时并重建 |
| `newSession(options?)` | 新建会话，可设置 parentSession 与 setup |
| `fork(entryId, options?)` | 从 entry 前/处 fork；持久化会话会创建分支文件 |
| `importFromJsonl(path, cwdOverride?)` | 复制/导入 JSONL 并切换 |
| `dispose()` | 触发 `session_shutdown`、清理 |

每次替换都会：

1. await 当前会话 `abort()`，让中止的 turn 落盘。
2. 发出 `session_shutdown`。
3. `session.dispose()`。
4. 用同一工厂为新 cwd 重建服务与会话。
5. 重新绑定 UI/订阅。

## 7. SessionManager 与会话格式

[SessionManager](../packages/coding-agent/src/core/session-manager.ts) 是模型上下文的权威来源。

- 持久化：`~/.pi/agent/sessions/<project>/...jsonl`（可用 `--session-dir`/环境变量覆盖）。
- 每条记录一行 JSON；header 包含 `id/version/cwd/parentSession`。
- Entry 类型：`message`、`thinking_level_change`、`model_change`、`compaction`、`branch_summary`、`custom`、`custom_message`、`label`、`session_info`、`context_edit`、`usage`。
- 每个 entry 带 `id` 与 `parentId`，构造成树；`leafId` 是活动分支。
- `buildSessionContext()` 从活动分支重建 `messages/thinkingLevel/model`，并应用压缩与分支摘要。
- 静态方法：`create/open/inMemory/continueRecent/list/listAll/findById/forkFrom`。
- 实例方法：`append*`、`createBranchedSession`、`getBranch`、`getEntry`、`getSessionFile`、`getSessionDir`、`setSessionName`、`rewrite` 等。

会话文件格式详见 `packages/coding-agent/docs/session-format.md`。

## 8. 设置与配置

`SettingsManager`（[settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)）合并：

- 全局 `<agent-dir>/settings.json`。
- 项目 `.pi/settings.json`（仅在项目信任后）。
- 运行时 overrides。

特点：

- 文件写入带锁、串行化、错误收集。
- 字段级修改跟踪：只写回被修改的字段，保留未知字段。
- 内置迁移与 schema 校验。
- 覆盖模型默认、thinking、compaction、retry、HTTP 超时、cache warming、TUI、包、分享、遥测等。

关键配置项详见 `packages/coding-agent/docs/settings.md`。

## 9. ModelRuntime

[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) 实现 `Models` 接口，是 coding-agent 的模型/鉴权中枢：

- 从 `auth.json` 加载凭据，从 `models.json` 加载自定义 Provider/模型。
- 注册内置 Provider、扩展 Provider、原生 Provider、虚拟模型。
- 合并远程模型目录与个性化目录（OpenRouter）。
- `resolveModel()` 处理虚拟模型路由。
- `getAuth()` 合并 runtime API key、环境、Provider/模型 headers。
- `streamSimple()` 注入兼容配置、重试、超时。
- 提供 `hasConfiguredAuth()`、`getProviderAuthStatus()`、`refresh()`、`login/logout`。

`model-resolver.ts` 负责 CLI/设置中的模型模式匹配与作用域解析。

## 10. 内置工具

| 工具 | 作用 | 关键实现 |
|---|---|---|
| `read` | 读文件/行范围/图片；输出截断；图片按模型限制缩放 | [read.ts](../packages/coding-agent/src/core/tools/read.ts) |
| `bash` | 执行 shell，流式输出，超时，进程树终止，截断与完整输出文件 | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) |
| `powershell` | Windows PowerShell 对应实现 | [powershell.ts](../packages/coding-agent/src/core/tools/powershell.ts) |
| `edit` | 多处精确文本替换，保留换行/BOM，生成 diff | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) |
| `write` | 写文件，经文件变更队列串行化 | [write.ts](../packages/coding-agent/src/core/tools/write.ts) |
| `grep` | 内容搜索 | [grep.ts](../packages/coding-agent/src/core/tools/grep.ts) |
| `find` | 文件查找 | [find.ts](../packages/coding-agent/src/core/tools/find.ts) |
| `ls` | 目录列表 | [ls.ts](../packages/coding-agent/src/core/tools/ls.ts) |

公共机制：

- `ToolDefinition` 同时面向模型与 UI：参数 schema、`promptSnippet`、`promptGuidelines`、`renderCall/renderResult`、`structuredContent`。
- `withFileMutationQueue()` 按文件串行化 edit/write，避免并发覆盖。
- `truncateHead/truncateTail/truncateLine` 统一输出上限（默认 2000 行 / 50KB）。
- 工具工厂都接受 `operations` 注入，可代理到远端环境。

## 11. 扩展系统

### 11.1 组成

| 模块 | 职责 |
|---|---|
| [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) | 发现、缓存、加载扩展模块；支持内置/内联扩展 |
| [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) | 触发事件、合并结果、错误隔离、系统提示构建 |
| [types.ts](../packages/coding-agent/src/core/extensions/types.ts) | `ExtensionAPI`、`ExtensionContext`、工具/事件/渲染类型 |
| [wrapper.ts](../packages/coding-agent/src/core/extensions/wrapper.ts) | 把扩展工具包装成 `AgentTool` |

### 11.2 扩展可以做什么

- 订阅 `session_*`、`agent_*`、`turn_*`、`message_*`、`tool_*`、`provider_*`、`input`、`user_bash` 等事件。
- 修改系统提示、上下文、Provider 请求头/载荷、工具调用参数/结果。
- 注册工具、命令、快捷键、CLI flag、Provider、虚拟模型、MCP server。
- 注册消息/entry 渲染器、工具渲染器、Markdown transformer。
- 发送自定义消息、追加 entry、执行命令、切换模型/工具。
- 提供 UI：选择器、确认框、输入框、通知、状态栏、widget、自定义编辑器。

### 11.3 内置扩展

- **codemode**：注册 `codemode` 工具，在 QuickJS 沙箱执行脚本；脚本可调用 `codemode`/`deferred` 暴露的工具；只有输出进入模型上下文。
- **mcp**：读取 `.pi/mcp.json` 与 `mcp.json`，连接 stdio/HTTP server，按 exposure 注册工具/资源。
- **tool_search**：搜索延迟工具并动态加载。
- **llama**：本地 llama.cpp Provider 与下载/选择 UI。

## 12. 运行模式

### 12.1 Interactive

[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)

- 基于 `@earendil-works/pi-tui` 构建主屏或备用屏。
- 组件：聊天视口、编辑器、footer、状态指示器、工具执行、diff、markdown、mermaid、选择器、登录/OAuth、会话选择、主题/设置等。
- 负责输入解析、斜杠命令、快捷键、鼠标、图片粘贴、剪贴板、外部编辑器、会话分享、Bug report。

### 12.2 Print / JSON

[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)

- 发送初始消息与追加消息，等待 run 结束。
- `text`：只输出最终 assistant 文本。
- `json`：把 `AgentSessionEvent` 转成 JSONL（`toJsonEvent()`）。

### 12.3 RPC

[rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts)、[rpc-types.ts](../packages/coding-agent/src/modes/rpc/rpc-types.ts)

- stdin 每行一个命令，stdout 每行一个响应/事件。
- 命令：`prompt/steer/follow_up/abort/clear_queue/new_session`、模型/thinking、压缩/重试、bash、会话树/统计/导出、`get_messages/get_commands` 等。
- 响应带 `id` 关联；事件是 `AgentSessionEvent`。
- 扩展 UI 通过 `extension_ui_request` / `extension_ui_response` 往返。
- `RpcClient`（[rpc-client.ts](../packages/coding-agent/src/modes/rpc/rpc-client.ts)）提供进程级客户端。

## 13. 资源与包

`DefaultResourceLoader` 发现顺序（概念上）：

1. 内置扩展与内置资源。
2. 用户 Agent 目录。
3. 项目 `.pi`（信任后）。
4. CLI 显式路径（`--extensions/--skills/--prompt-templates/--themes`）。
5. `settings.json` 中声明的 Pi packages（npm/git/本地）。
6. 扩展 `resources_discover` 事件追加的路径。

资源冲突与诊断通过 `ResourceDiagnostic` 返回，不直接打印。

## 14. 压缩与重试

- 阈值：模型上下文窗口 - `reserveTokens`；保留最近 `keepRecentTokens`。
- 自动压缩在请求前检查；手动 `/compact` 可带自定义指令。
- 摘要请求使用独立路由 id，不污染会话缓存。
- 溢出恢复：检测上下文溢出 → 压缩 → 重试一次。
- 自动重试：可重试的 Provider/传输错误，指数退避，最大次数与延迟可配置。

## 15. 安全与信任

- `ProjectTrustStore` 记录项目是否可信。
- 未信任时：不加载项目 settings、`.pi` 资源与项目系统提示。
- Context files 仍会加载（不要求信任）。
- `--project-trust` 覆盖；非交互模式遇到未信任项目会报错或按默认策略处理。
- 扩展在 Pi 进程内运行，拥有相同 OS 权限；容器化方案见文档。

## 16. 依赖与产物

依赖：`pi-agent-core`、`pi-ai`、`pi-tui`、`pi-codemode`、`pi-mcp`，以及 `chalk`、`diff`、`grep`/`minimatch`、`jiti`、`quickjs-wasi`、`undici`、`yaml`、`photon-node` 等。

产物：

- `dist/`：未打包的 ESM + 类型。
- `dist/bundle/cli.js`：Node 发布入口。
- `dist/bundle/rpc-entry.js`：RPC 入口。
- Bun 独立二进制。
- `docs/`、`examples/`、`schemas/` 随包发布。

## 17. 测试

`packages/coding-agent/test/` 覆盖会话、压缩、扩展、工具、模式、模型、设置、主题、RPC、路径、图片、剪贴板等。

```bash
cd packages/coding-agent
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/agent-session-compaction.test.ts
```
