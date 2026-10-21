# 13 `@earendil-works/pi-telemetry`

## 1. 职责

提供厂商无关的遥测契约：

- 显式、基于回调的 `TelemetryContext` / `TelemetrySpan`。
- 共享 `NOOP_TELEMETRY_CONTEXT`。
- 参考实现 `InMemoryTelemetryContext`。
- 可序列化的类型化 schema 与类型推导。
- 适配器一致性测试。
- 不含 exporter、全局 current-span 状态或后端依赖。

源码：[packages/telemetry/src](../packages/telemetry/src/)

## 2. 核心 API

```ts
interface TelemetryContext {
  startSpan<T>(options: SpanOptions, callback: (span: TelemetrySpan) => T | Promise<T>): Promise<T>;
}

interface TelemetrySpan extends TelemetryContext {
  addEvent(name: string, attributes?: SpanAttributes): void;
  setAttributes(attributes: SpanAttributes): void;
  setStatus(status: SpanStatus): void;
}
```

关键语义：

- 没有公开 `end()`；`startSpan()` 在回调 settle 时自动结束 span。
- span 同时是子 span 的 parent context，形成显式嵌套。
- `SpanStatus`：`{ status: "ok" }` 或 `{ status: "error", error?: { name, message } }`。
- 属性值支持 string/number/boolean 及其数组。

## 3. 实现

| 实现 | 导入 | 说明 |
|---|---|---|
| `NOOP_TELEMETRY_CONTEXT` | 主入口 | 不记录任何数据；用于默认参数 |
| `InMemoryTelemetryContext` | 主入口 | 记录 span/event 到内存，测试与调试用 |

应用可自行实现适配器：OpenTelemetry、Sentry、日志等。

## 4. 类型化 Schema

```ts
const schema = defineTelemetrySchema({
  version: 1,
  spans: {
    "pi.request": {
      description: "One provider request",
      parents: { kind: "root_or_external" },
      startAttributes: {
        "pi.provider": { type: "string", description: "...", required: true },
      },
      endAttributes: {
        "pi.status": { type: "string", values: ["ok", "error"], description: "..." },
      },
      events: {
        "pi.retry": { description: "...", attributes: { "pi.attempt": { type: "number", required: true } } },
      },
      status: { default: "ok", errorWhen: "..." },
    },
  },
});
```

`createTypedSpanStarter(context, [schema])` 返回按 span 名重载的 starter：

- 启动属性在编译期检查必填/可选与字面量集合。
- `addEvent()` / `setAttributes()` 同样受限。
- 支持多 schema 组合并检测重复 span 名。
- 运行时不做 schema 校验，仅类型推导。

相关类型：

- `TelemetrySchemaDefinition`
- `InferStartAttributes` / `InferOptionalAttributes` / `InferEventAttributes`
- `ExactTelemetryAttributes`（拒绝多余字段）
- `TypedSpanStarter`

## 5. 适配器契约

实现 `TelemetryContext` 时必须：

- 让 `startSpan` 的返回值/异常原样传播。
- 回调 settle 后结束 span。
- 子 span 归属正确。
- 不让遥测失败改变业务结果。
- 属性/事件值不可变且可序列化。
- 不依赖全局状态。

`@earendil-works/pi-telemetry/testing` 提供一致性测试套件。

## 6. 在 Pi 中的使用

- `pi-ai` 的 `ProviderRequestOptions` 接受 `telemetryContext`。
- 各包把 telemetry 显式传递，不使用全局 current span。
- 业务 schema 由各域自行定义。

## 7. 安全

- 属性不应包含密钥、token、完整 prompt 等敏感信息。
- schema 支持 `sensitive?: boolean` 元数据，供适配器决定脱敏/丢弃。
- `cardinality` 提示高基数属性。

## 8. 依赖与测试

无运行时依赖。

```bash
cd packages/telemetry
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/conformance.test.ts
```
