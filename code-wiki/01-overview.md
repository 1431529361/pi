# 01 项目总览

## 1. 项目定位

Pi（包名 `@earendil-works/pi-coding-agent`，命令行名 `pi`）是一个最小、可扩展的编码 Agent 运行框架，同时提供：

- 交互式终端编码助手（TUI）。
- 单次执行模式（`-p` 文本输出、`--mode json` 事件流）。
- 无头 RPC 模式（stdin/stdout JSONL）。
- TypeScript SDK，把 Agent 嵌入其他 Node.js/Bun 进程。
- 可分发资源体系：扩展（extensions）、技能（skills）、提示模板（prompt templates）、主题（themes）、Pi packages。

设计上刻意保持内核小：子 Agent、Plan mode 等能力不内建，而以扩展/包的形式提供。

## 2. 仓库结构

```text
pi/
├─ packages/
│  ├─ ai/               # 多提供商 LLM 统一 API
│  ├─ agent/            # Agent 运行时（工具调用、事件流、状态）
│  ├─ coding-agent/     # 编码 Agent CLI、会话、模式、扩展系统
│  ├─ tui/              # 终端 UI 库（差分渲染）
│  ├─ durable/          # 持久化会话/任务/文档运行时（实验性）
│  ├─ chord/            # 契约、对象、热重载、RPC 运行时
│  ├─ codemode/         # QuickJS 沙箱代码执行
│  ├─ mcp/              # Model Context Protocol 客户端
│  ├─ env/              # SSH 远程执行环境（Rust daemon + TS 客户端）
│  ├─ telemetry/        # 厂商无关遥测契约与类型化 schema
│  └─ evals/            # 行为评测与文档增益评测（私有包）
├─ scripts/             # 构建、发布、模型目录、安装锁、issue 分流等脚本
├─ nix/                 # Nix 打包与模型目录 pin
├─ .github/workflows/   # CI、二进制构建、发布、安全审计
├─ AGENTS.md            # 面向人类与 Agent 的开发规则
├─ CONTRIBUTING.md      # 贡献门禁与质量要求
├─ test.sh              # 隔离环境下的全量测试入口
└─ pi-test.sh           # 从源码运行 pi
```

## 3. 包总览

| 包 | 版本 | 内部依赖 | 职责 |
|---|---|---|---|
| `@earendil-works/pi-ai` | 1.2.0 | telemetry | Provider/模型目录/鉴权/流式协议/成本计算；统一 OpenAI、Anthropic、Google、Bedrock、OpenRouter 等 40+ 提供商 |
| `@earendil-works/pi-agent-core` | 1.2.0 | ai | 有状态 Agent、Agent 循环、工具执行、steering/follow-up 队列、生命周期事件 |
| `@earendil-works/pi-coding-agent` | 1.2.0 | agent, ai, tui, codemode, mcp | CLI 入口、会话管理、模型运行时、工具、扩展系统、三种运行模式、SDK |
| `@earendil-works/pi-tui` | 1.2.0 | 无 | Component 模型、主屏/备用屏渲染器、差分渲染、布局、覆盖层、颜色、键位、终端图片 |
| `@earendil-works/pi-durable` | 1.2.0 | chord, ai, codemode | 提交即持久化的会话/任务/文档内核，崩溃恢复、任务调度、存储后端 |
| `@earendil-works/chord` | 1.2.0 | 无（typebox 为依赖） | 契约驱动的对象运行时、principal、复制状态、JSON link、热重载、进程集群 |
| `@earendil-works/pi-codemode` | 1.2.0 | 无 | 在 QuickJS/WASM 沙箱中执行模型生成的 JS，只暴露注入工具 |
| `@earendil-works/pi-mcp` | 1.2.0 | 无 | 独立 MCP 客户端：stdio/Streamable HTTP、JSON-RPC、OAuth、进度/取消 |
| `@earendil-works/pi-env` | 1.2.0 | chord, durable | 通过 SSH 把 Durable 的 `ExecutionEnv` 放到远端；Rust daemon 负责文件/命令/PTY |
| `@earendil-works/pi-telemetry` | 1.2.0 | 无 | `TelemetryContext`/`TelemetrySpan` 契约、NOOP 与内存实现、类型化 schema |
| `@earendil-works/pi-evals` | 1.2.0 | ai, coding-agent（dev） | 基于 vitest-evals 的宿主评测与 Docker 文档增益评测 |

## 4. 技术栈

- **语言/模块**：TypeScript 7、ESM（`"type": "module"`）、Node.js `>=22.19.0`；`erasableSyntaxOnly`，只允许可擦除语法。
- **Schema**：TypeBox（`typebox`），工具参数、契约、设置 schema 均使用它。
- **构建**：
  - 各包 `tsc -p tsconfig.build.json` 输出 `dist/`。
  - coding-agent 额外用 esbuild 生成 `dist/bundle/cli.js`。
  - 发布二进制用 Bun `--compile`。
- **测试**：Vitest（多数包）、`node:test`（tui）、`vitest-evals`（evals）。
- **持久化**：JSONL（coding-agent 会话）、Memory/JSONL/SQLite（durable）。
- **沙箱**：QuickJS compiled to WebAssembly（codemode）、SSH + Rust daemon（env）。
- **终端**：ANSI/CSI、Kitty keyboard/graphics、iTerm2 图片协议、OSC 133/2026。
- **包管理约束**：`save-exact=true`、`min-release-age=2`，直接依赖锁定精确版本；`package-lock.json` 为依赖事实源。

## 5. 核心术语

| 术语 | 含义 | 主要位置 |
|---|---|---|
| Agent | 有状态的对话执行器，持有消息、模型、工具和运行状态 | [agent.ts](../packages/agent/src/agent.ts) |
| Agent Loop | 单次运行的低层循环：请求模型 → 执行工具 → 决定是否继续 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) |
| AgentSession | 编码 Agent 的会话门面：持久化、压缩、重试、模型/思考级别、扩展绑定 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) |
| Session | 消息树 + 元数据，持久化为 JSONL；活动分支决定模型上下文 | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) |
| Provider | 提供商运行时单元：模型目录、鉴权、流式实现 | [models.ts](../packages/ai/src/models.ts) |
| API | 线协议实现，如 `anthropic-messages`、`openai-responses`、`openai-completions` | [api/](../packages/ai/src/api/) |
| Turn / Run | Turn 是一次模型响应及其工具调用；Run 是从输入到最终回答的多个 Turn | `how-pi-works.md`、durable 文档 |
| Compaction | 用摘要替换较早上下文，原始 entry 仍保留 | [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) |
| Extension | 进程内 TypeScript 模块，注册工具/命令/快捷键/Provider/事件处理器 | [extensions/](../packages/coding-agent/src/core/extensions/) |
| Skill | 按需加载的指令与配套文件 | [skills.ts](../packages/coding-agent/src/core/skills.ts) |
| Facet / Object | Chord 中“某宿主上的一段代码”/“地址上的状态 + 代码” | [host.ts](../packages/chord/src/host.ts) |
| Harness | Durable 中打开存储并运行 Agent 的入口 | [harness.ts](../packages/durable/src/harness/harness.ts) |
| ExecutionEnv | 文件系统 + Shell 的统一执行环境接口 | [env/index.ts](../packages/durable/src/env/index.ts) |

## 6. 设计要点

1. **模型与 Agent 解耦**：`pi-agent-core` 只依赖 `StreamFn` 形状，不依赖任何 Provider 目录；默认流函数由宿主通过 `setDefaultStreamFn()` 注入。
2. **Provider 与 API 解耦**：Provider 负责“谁”（鉴权、目录、路由），API 负责“怎么发请求”。混合 API 的 Provider（如 GitHub Copilot）按 `model.api` 分发。
3. **提交即事实**：coding-agent 的会话在事件流中写入 JSONL；durable 要求所有状态先提交再对外可见。
4. **可扩展优先**：扩展可拦截上下文、Provider 请求头/载荷、工具调用、消息渲染、会话切换/压缩/树导航等。
5. **安全边界外置**：Pi 不自带权限系统，默认继承启动用户的 OS 权限；需要隔离时使用容器或沙箱方案（见 `packages/coding-agent/docs/containerization.md`）。

## 7. 入口索引

| 入口 | 路径 |
|---|---|
| CLI 可执行 | `packages/coding-agent/src/cli.ts` → `main()` |
| RPC 可执行 | `packages/coding-agent/src/rpc-entry.ts` |
| SDK 入口 | [index.ts](../packages/coding-agent/src/index.ts) 的 `createAgentSession` |
| Agent 库入口 | [index.ts](../packages/agent/src/index.ts) |
| AI 库入口 | [index.ts](../packages/ai/src/index.ts) / `models.ts` |
| TUI 库入口 | [index.ts](../packages/tui/src/index.ts) |
| Durable 入口 | [index.ts](../packages/durable/src/index.ts) |
| Chord 入口 | [index.ts](../packages/chord/src/index.ts) |
