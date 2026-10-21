# 04 `@earendil-works/pi-ai`

## 1. 职责

`pi-ai` 是 Pi 的统一 LLM 接入层。它把不同厂商的 HTTP/WebSocket 协议折叠成同一套：

- 模型目录（chat / image / classifier）。
- Provider 鉴权：API key、环境变量、OAuth、云凭证。
- 流式事件协议。
- 工具调用参数解析与校验。
- token 用量与成本计算。
- 上下文序列化、跨 Provider 迁移、上下文溢出判断。
- 图片生成与结构化分类。

它不包含 Agent 循环，也不包含会话存储。

源码：[packages/ai/src](../packages/ai/src/)

## 2. 核心抽象

### 2.1 API、Provider、Models

| 概念 | 含义 | 典型位置 |
|---|---|---|
| **API** | 线协议实现，例如 `anthropic-messages`、`openai-responses`、`openai-completions`、`google-generative-ai`、`bedrock-converse-stream` | `src/api/*.ts` |
| **Provider** | 运行时单元：id/name/baseUrl、鉴权、模型列表、支持的 API、图片/分类实现 | `src/providers/*.ts` |
| **Models** | Provider 集合，负责按模型路由、解析鉴权、统一 `stream/complete/generateImages/classify` | [models.ts](../packages/ai/src/models.ts) |

Provider 内部通过 `createProvider()` 组装：

```ts
createProvider({
  id: "anthropic",
  baseUrl: "https://api.anthropic.com",
  auth: { apiKey: ..., oauth: ... },
  models: [...],
  api: anthropicMessagesApi(),
});
```

`api` 可以是单个 `ProviderStreams`，也可以是按 `model.api` 分发的 map（混合 API Provider，如 GitHub Copilot）。

### 2.2 Model 类型

`Model<TApi>`（chat）关键字段：

```ts
id, name, api, provider, baseUrl,
input,              // text/image 等输入模态
inputLimits,        // 图片大小/缩放等限制
cost,               // 输入/输出/缓存读写单价，可带阶梯价
reasoning,          // 是否支持 thinking
thinkingLevelMap,   // pi thinking level → provider 值；null 表示不支持
promptCache,        // 缓存寿命
contextWindow, maxTokens,
samplingParams, samplingParamsByThinkingLevel,
compat,             // OpenAI/Anthropic/Bedrock 兼容覆盖
```

另有 `ImageModel<ImageApi>`（`type: "image"`）与 `ClassifierModel<ClassifierApi>`（`type: "classifier"`）。

### 2.3 消息与上下文

```ts
Context = {
  systemPrompt?: string,
  messages: Message[],
  tools?: Tool[],
}

TranscriptContext = {
  messages: Message[]   // 已把 systemPrompt/tools 折叠进首条 system message
}
```

`normalizeContext()` 是唯一能把 `Context` 变成 `TranscriptContext` 的入口。system message 还承载工具增量：

- `toolsAdded` / `toolsRemoved`
- `sections`：按名字 patch 的系统提示片段

`getCurrentTools()`、`getCurrentSystemMessage()`、`getCurrentSystemPrompt()` 会重放整条 transcript，得到当前有效工具与提示。

关键文件：[utils/transcript.ts](../packages/ai/src/utils/transcript.ts)。

### 2.4 消息内容

- `SystemMessage`、`UserMessage`、`AssistantMessage`、`ToolResultMessage`。
- `TextContent`、`ThinkingContent`（可带签名）、`ImageContent`、`ToolCall`。
- `AssistantMessage` 包含 `content`、`usage`、`stopReason`、`errorMessage`、`diagnostics`、`durationMs` 等。

### 2.5 流式事件

`AssistantMessageEvent`（[types.ts](../packages/ai/src/types.ts)）：

```text
start
text_start / text_delta / text_end
thinking_start / thinking_delta / thinking_end
toolcall_start / toolcall_delta / toolcall_end
done | error
```

`AssistantMessageEventStream` 是带最终结果的 `EventStream`：

```ts
const stream = models.streamSimple(model, context);
for await (const event of stream) { ... }
const message = await stream.result();
```

`done` 或 `error` 事件会解析出最终 `AssistantMessage`，并在流创建时开始计时 `durationMs`。

## 3. 关键 API

### 3.1 Models 集合

| API | 说明 |
|---|---|
| `createModels(options?)` | 创建空 `MutableModels`；可注入 `CredentialStore`、`ModelsStore`、`AuthContext` |
| `builtinModels(options?)` | 创建并注册全部内置 Provider |
| `builtinProviders()` | 返回全部内置 Provider 的新实例 |
| `setProvider()` / `deleteProvider()` / `clearProviders()` | 管理 Provider |
| `getProviders()` / `getProvider(id)` | 读取 Provider |
| `getModels(provider?)` | 同步读取最后一次已知的 chat 模型 |
| `getModel(provider, id)` | 同步查找 |
| `getAllModels(provider?)` / `getModelsOfType()` / `getModelOfType()` | 读取全部类型 |
| `refresh(options?)` | 并发刷新动态 Provider；恢复缓存 → 若允许联网则再拉取；不因单个 Provider 失败而 reject |
| `checkAuth(providerId)` | 检查鉴权是否完整，不刷新 OAuth |
| `getAvailable(providerId?)` / `getAllAvailable()` | 只返回鉴权完整的模型 |
| `getAuth(providerId|model)` | 解析请求鉴权与模型级 headers |
| `login(providerId, type, interaction)` | 执行 Provider 自有登录并持久化凭据 |
| `logout(providerId)` | 删除凭据 |
| `stream()` / `complete()` | 按 `model.api` 走完整选项 |
| `streamSimple()` / `completeSimple()` | 统一简化选项，兼容层常用 |
| `generateImages()` / `classify()` | 图片生成与分类；不 reject，返回错误结果 |

### 3.2 Provider 构建

| API | 说明 |
|---|---|
| `createProvider(input)` | 从 `auth/models/api/images/classifiers/fetchModels` 组装 Provider |
| `hasApi(model, api)` | 运行时把动态模型收窄为具体 API 类型 |
| `calculateCost(model, usage)` | 按 token 与阶梯价计算成本 |
| `getSupportedThinkingLevels(model)` | 根据 `reasoning` 与 `thinkingLevelMap` 返回可用级别 |
| `clampThinkingLevel(model, level)` | 把请求级别吸附到最近可用级别 |
| `modelsAreEqual(a, b)` | 比较 type/id/provider |

### 3.3 工具

`Tool` 定义：

```ts
{
  name: string,
  description: string,
  parameters: TSchema,          // TypeBox
  constrainedSampling?: false | { type: "json_schema"; strict: "prefer" | "require" } | { type: "grammar"; variants }
}
```

工具调用处理工具函数：

- `validateToolArguments(tool, toolCall)`：按 schema 校验。
- `parseToolCallArguments()` / `parseStreamingJson()` / `parseJsonWithRepair()`：流式 JSON 与容错解析。
- `toToolDeclaration()`：把 `AgentTool` 转为 Provider 可发送的声明。
- `getToolStateChanges()`：比较当前工具集与 transcript 声明，产出 `toolsAdded/toolsRemoved`。

## 4. 请求路径

```text
Models.stream(model, context, options)
  ├─ normalizeContext(context) → TranscriptContext
  ├─ lazyStream：延迟到真正订阅/取结果时才执行
  ├─ requireChatProvider(model)
  ├─ applyAuth(model, options)
  │    ├─ getAuth → resolveProviderAuth
  │    │    ├─ 读取 CredentialStore
  │    │    ├─ OAuth 过期则在 modify 中刷新，避免并发双刷新
  │    │    └─ API key 解析器 / 环境变量 / 云凭证
  │    ├─ 合并 Provider headers + 模型 headers + 调用方 headers
  │    └─ auth.baseUrl 可覆盖模型 baseUrl
  └─ provider.stream(requestModel, transcript, requestOptions)
       └─ 按 model.api 分发到 API 实现
```

API 实现负责：请求体转换、流解析、错误归一化、usage、stop reason、工具调用增量、重试与中止。

## 5. 鉴权体系

| 类型 | 说明 |
|---|---|
| `ApiKeyAuth` | `login()` 采集密钥；`resolve()` 返回 `{ apiKey }`/`{ headers }`/`{ env }` |
| `OAuthAuth` | `login()` 走 OAuth 流程；`refresh()` 更新 token；`isSubscription` 标记订阅型账号 |
| `CredentialStore` | 每 Provider 一份凭据；`modify()` 是唯一写路径，串行化读改写 |
| `AuthContext` | 注入 `env()` / `fileExists()`，便于测试与浏览器 |
| `resolveProviderAuth()` | 统一解析 stored credential、环境变量、cloud profile |
| `refreshStoredOAuthCredential()` | 在并发请求间共享一次刷新 |

鉴权解析结果 `AuthResult` 可包含 `source`（如 `ANTHROPIC_API_KEY`、`OAuth`、`~/.aws/credentials`），用于状态 UI。

## 6. Provider 与 API 清单

内置 Provider 覆盖（完整列表见 [providers/all.ts](../packages/ai/src/providers/all.ts)）：

OpenAI、Azure OpenAI、OpenAI Codex、Anthropic、Google、Google Vertex、Amazon Bedrock、Mistral、Groq、Cerebras、xAI、OpenRouter、Vercel AI Gateway、Cloudflare AI Gateway、Cloudflare Workers AI、DeepSeek、NVIDIA NIM、Together、Baseten、Hugging Face、Moonshot、MiniMax、ZAI、Qwen Token Plan、Xiaomi、GitHub Copilot、OpenCode Zen/Go、Kimi Coding、Meta、Fireworks、Radius、TypeSafe、Ant Ling，以及任意 OpenAI 兼容端点。

API 实现：

| API | 文件 |
|---|---|
| `anthropic-messages` | [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) |
| `openai-completions` | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts) |
| `openai-responses` / `azure-openai-responses` | [openai-responses.ts](../packages/ai/src/api/openai-responses.ts) |
| `openai-codex-responses` | [openai-codex-responses.ts](../packages/ai/src/api/openai-codex-responses.ts) |
| `google-generative-ai` / `google-vertex` | [google-generative-ai.ts](../packages/ai/src/api/google-generative-ai.ts) |
| `bedrock-converse-stream` | [bedrock-converse-stream.ts](../packages/ai/src/api/bedrock-converse-stream.ts) |
| `mistral-conversations` | [mistral-conversations.ts](../packages/ai/src/api/mistral-conversations.ts) |
| `pi-messages` | [pi-messages.ts](../packages/ai/src/api/pi-messages.ts) |
| `openrouter-images` | [openrouter-images.ts](../packages/ai/src/api/openrouter-images.ts) |
| classifier APIs | `typesafe-system-one`、`cloudflare-workers-ai-system-one`、`llama-cpp-classify`、`openai-decisions` |

每个 API 都有 `.lazy.ts` 入口，按需加载，减少冷启动与包体。

## 7. 模型目录

- `src/models.generated.ts` 是生成文件，禁止手改；通过 `packages/ai/scripts/generate-models.ts` 生成。
- 每个 Provider 的静态目录在 `src/providers/<provider>.models.ts`。
- `src/providers/data/` 保存运行时模型数据与 manifest。
- `npm run generate:models`、`hydrate:model-data`、`check:model-data`、`generate:model-catalog` 均由包脚本提供。
- 动态 Provider（如 Radius、OpenRouter 用户模型）通过 `refreshModels()` 恢复持久目录并可选联网刷新。

## 8. 兼容层与子路径

| 子路径 | 用途 |
|---|---|
| `@earendil-works/pi-ai` | 核心、无副作用；不含生成目录与 Provider 工厂 |
| `@earendil-works/pi-ai/models` | `createModels` 等集合 API |
| `@earendil-works/pi-ai/providers/*` | 单 Provider 工厂 |
| `@earendil-works/pi-ai/api/*` | API 实现 |
| `@earendil-works/pi-ai/utils/*` | 事件流、重试、解析、溢出等 |
| `@earendil-works/pi-ai/compat` | 旧版全局 API 兼容层（coding-agent 大量使用） |
| `@earendil-works/pi-ai/oauth` | OAuth 入口 |
| `@earendil-works/pi-ai/bedrock-provider` | Bedrock 专用入口 |

## 9. 依赖与扩展点

依赖：`@anthropic-ai/sdk`、`@aws-sdk/client-bedrock-runtime`、`@google/genai`、`openai`、`partial-json`、`typebox`、`@earendil-works/pi-telemetry`。

扩展方式：

1. 新增 API：实现 `ProviderStreams`，导出 lazy 入口。
2. 新增 Provider：`createProvider()` + 静态/动态模型目录 + 鉴权。
3. 新增 OAuth：实现 `OAuthAuth`，在 `auth/oauth/` 下注册。
4. 自定义兼容端点：通过 `models.json` 或 `createProvider()` 构造。

## 10. 测试

`packages/ai/test/` 覆盖 Provider 转换、流解析、鉴权、token/成本、工具 schema、thinking、图片、错误与重试。许多 e2e 用例只在对应 API key 存在时启用。

```bash
cd packages/ai
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/models-entry.test.ts
```
