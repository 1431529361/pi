# 03 运行与开发

## 1. 环境要求

- Node.js `>=22.19.0`（根 `engines` 与各包一致）。
- npm（仓库以 `package-lock.json` 为依赖事实源）。
- 可选：Bun（构建独立二进制、运行 `pi-test.sh` 之外的二进制路径）、Rust/Cargo（构建 `pi-env` daemon）、Nix（离线构建）。
- Linux 构建 canvas/photon 相关依赖时，CI 安装：
  `libcairo2-dev libpango1.0-dev libjpeg-dev libgif-dev librsvg2-dev fd-find ripgrep`。

## 2. 安装依赖

```bash
npm install --ignore-scripts     # 本地开发
npm ci --ignore-scripts          # 干净/CI 安装
```

仓库策略：直接依赖精确版本、`min-release-age=2`、不运行生命周期脚本。`packages/coding-agent/install-lock/` 是 pi.dev 安装器使用的传递依赖锁定文件。

## 3. 构建

```bash
npm run build          # 刷新模型数据后构建所有包
npm run build:offline  # 使用已有模型数据离线构建
```

构建顺序（根 `package.json`）：

```text
chord → tui → telemetry → codemode → mcp → ai → durable → env → agent → coding-agent
```

说明：

- `packages/ai` 的 `build` 会先运行 `generate-models`（联网刷新模型目录），再 `tsc`；`build:offline` 只做 `check:model-data` + `tsc` + 拷贝 `providers/data`。
- `packages/coding-agent` 的 `build` = `tsc`（`dist/`）+ esbuild 打包（`dist/bundle/cli.js`）。
- TUI 原生剪贴板模块按平台单独构建：

```bash
npm run build:native:darwin
npm run build:native:linux
npm run build:native:win32
```

- `pi-env` 的 Rust daemon：

```bash
npm run build:daemon --workspace packages/env
```

## 4. 从源码运行

```bash
./pi-test.sh                 # 任意目录可运行，保留调用者 cwd
./pi-test.sh --no-env        # 清除常见 API key 环境变量
./pi-test.sh --help
```

`pi-test.sh` 通过 `packages/coding-agent/src/experimental/source-resolver.ts` 让 Node 直接解析 workspace 源码，不依赖 `dist/`。

也可以：

```bash
node --import <source-resolver-url> packages/coding-agent/src/cli.ts
```

## 5. 运行模式

| 模式 | 触发 | 输入/输出 |
|---|---|---|
| Interactive | 默认 TTY | TUI；`/` 命令、编辑器、会话树 |
| Print | `-p` / 非 TTY stdin 或 stdout | 最终回答文本 |
| JSON | `--mode json` | 所有 Agent 事件 JSONL |
| RPC | `--mode rpc` 或 `rpc-entry` | stdin JSON 命令，stdout 响应 + 事件 JSONL |

常用参数（完整列表见 `pi --help`）：

```bash
pi                         # 交互模式
pi "解释这个仓库"            # 初始消息
pi -p "总结 README"          # 单次输出
pi --mode json "..."        # 事件 JSON 流
pi --mode rpc               # 无头 RPC
pi --continue               # 继续当前目录最近会话
pi --resume                 # 会话选择器
pi --session <id|path>      # 打开指定会话
pi --fork <id|path>         # 从会话分叉
pi --model provider/model   # 指定模型
pi --provider openai --model gpt-...
pi --models "anthropic/*"   # 可循环模型范围
pi --thinking high          # off|minimal|low|medium|high|xhigh|max
pi --tools read,bash        # 工具白名单（支持 * 与 +name/-name）
pi --no-tools               # 禁用全部工具
pi --no-builtin-tools       # 禁用内置工具，保留扩展工具
pi --session-dir <dir>      # 会话目录
pi --no-session             # 不落盘
pi --export <session> [out] # 导出 HTML
pi --list-models [pattern]  # 列出可用模型
pi --version
pi --help
```

子命令：

```bash
pi auth check ...                # 检查 Provider/模型鉴权状态
pi auth print-api-key ...        # 打印解析后的 API key
pi auth print-bearer-token ...   # 打印 bearer token
pi install <source>              # 安装 Pi package
pi remove <source>               # 移除 Pi package
pi update [...]                  # 更新 Pi / 扩展 / 模型目录
pi list                          # 列出 Pi package
pi config ...                    # 配置管理
pi mcp ...                       # MCP server 管理
```

## 6. 配置与数据目录

默认 Agent 目录：`~/.pi/agent/`，可用 `PI_CODING_AGENT_DIR` 覆盖。

| 路径 | 作用 |
|---|---|
| `<agent-dir>/settings.json` | 用户设置、资源路径、Pi packages |
| `<agent-dir>/auth.json` | API key / OAuth 凭据 |
| `<agent-dir>/models.json` | 自定义兼容端点、模型覆盖 |
| `<agent-dir>/mcp.json` | 全局 MCP server |
| `<agent-dir>/extensions/` | 用户扩展 |
| `<agent-dir>/skills/` | 用户技能 |
| `<agent-dir>/prompts/` | 提示模板 |
| `<agent-dir>/themes/` | 主题 |
| `<agent-dir>/sessions/` | 默认会话目录 |
| `.pi/`（项目） | 项目设置、MCP、扩展、技能、模板、主题、系统提示 |

常用环境变量：

| 变量 | 作用 |
|---|---|
| `PI_CODING_AGENT_DIR` | Agent 配置目录 |
| `PI_CODING_AGENT_SESSION_DIR` | 会话目录 |
| `PI_OFFLINE` | 离线模式（禁用版本检查与目录网络刷新） |
| `PI_PACKAGE_DIR` | 覆盖包资源目录（Nix/Guix 场景） |
| `PI_STARTUP_BENCHMARK` | 交互模式启动基准输出 |
| `PI_SHARE_VIEWER_URL` | 会话分享查看器地址 |
| `PI_ENV_DAEMON` | pi-env 测试使用的 daemon 路径 |
| Provider 专用变量 | `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GEMINI_API_KEY`、`AWS_*` 等，见 [env-api-keys.ts](../packages/ai/src/env-api-keys.ts) |

## 7. 检查与测试

```bash
npm run check        # Biome 写入格式化 + 依赖/导入/入口图/安装锁检查 + tsc --noEmit + 浏览器 smoke
npm run check:pinned-deps
npm run check:runtime-deps
npm run check:ts-imports
npm run check:entry-graphs
npm run check:install-lock:coding-agent
```

测试：

```bash
./test.sh            # 推荐：隔离 HOME/TMP/Git/npm 配置，未配置密钥时跳过联网用例
npm test             # 根脚本：scripts 测试 + 所有 workspace 的 test
```

单包/单文件：

```bash
# Vitest 包（ai、agent、coding-agent、durable、chord、codemode、mcp、env、telemetry）
cd packages/ai
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/specific.test.ts

# tui 使用 node:test
cd packages/tui
node --test test/specific.test.ts
```

注意：

- 不要直接运行完整 Vitest 套件（包含检测到 endpoint/auth 环境变量才启用的 e2e 用例）。
- `packages/coding-agent/test/suite/` 使用 `harness.ts` + faux provider，不使用真实 API key。
- 修复 GitHub issue 的回归测试需在测试旁注释 issue 编号。

## 8. 评测

```bash
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol npm run eval -w packages/evals
```

- `eval:host`：本机运行 smoke、文档审计等。
- `eval:docs`：Docker 中构建 `without_docs` / `with_docs` 两个镜像，配对比较文档带来的通过率增益。
- 结果写入 `.eval/<timestamp>_<id>/`（protocol、observations、report、会话）。

详见 [14-package-evals.md](./14-package-evals.md)。

## 9. 独立二进制

```bash
# coding-agent 包内构建当前平台二进制
npm run build:binary --workspace packages/coding-agent

# 构建全部平台发布包
./scripts/build-binaries.sh \
  --offline-model-data \
  --platform linux-x64 \
  --out "$PWD/out"
```

可用平台：`darwin-arm64`、`darwin-x64`、`linux-x64`、`linux-arm64`、`windows-x64`、`windows-arm64`。

二进制由 Bun `build --compile` 生成，额外嵌入 `image-resize-worker.ts` 与 `codemode/worker.ts` 两个入口。

## 10. Nix

```bash
nix run github:earendil-works/pi/stable
nix build .
nix run .
```

Nix 构建离线，模型数据由 `nix/model-catalog.json` 固定的 pi.dev 目录修订提供；运行时仍会叠加更新目录。刷新 pin：

```bash
npm run update:model-catalog-pin
```

## 11. 发布与版本

```bash
npm run release:patch|minor|major   # 计算版本、生成发布内容
npm run release:local               # 本地发布流程
npm run publish:dry                 # 构建 + 检查 + 打包演练
npm run publish                     # 构建、检查、发布 npm
npm run pack:packages -- --out .artifacts/pi-packages
```

发布相关约束：

- 发布使用验证过的 tarball，不重新打包 workspace 目录。
- npm 发布不锁定传递依赖；pi.dev 安装器使用 `packages/coding-agent/install-lock/`。
- `prepublishOnly` 会执行 `clean + build + check`。

## 12. CI

`.github/workflows/ci.yml` 的主流程：

```text
checkout → Node 22 → 安装系统依赖 → npm ci --ignore-scripts
→ npm run build → npm run check → build pi-env daemon → npm test
```

另有 MCP conformance job、npm audit、Nix 构建、二进制构建、issue 分流等独立工作流。

## 13. 常见问题

| 现象 | 处理 |
|---|---|
| `npm run build` 卡在模型数据 | 使用 `npm run build:offline`；或设置 `PI_OFFLINE=1` |
| canvas/photon 安装失败 | 安装 Cairo/Pango/JPEG/GIF/librsvg 开发包 |
| `pi-env` 测试找不到 daemon | 先 `npm run build:daemon --workspace packages/env`，或设置 `PI_ENV_DAEMON` |
| 测试访问真实密钥/网络 | 使用 `./test.sh` 或 `./pi-test.sh --no-env` |
| 扩展加载失败 | 用 `pi -ne` 禁用扩展启动；查看 `~/.pi/agent/pi-debug.log` |
| 会话在其他项目目录 | 交互模式会提示是否 fork 到当前目录；非交互模式报 `MissingSessionCwdError` |
