# 14 `@earendil-works/pi-evals`

> 私有包，不发布。用于 Pi 自身的行为评测。

## 1. 职责

基于 `vitest-evals` 的评测框架，包含两类：

1. **Host evals**：在本机运行普通 vitest-evals 套件（smoke、文档审计等）。
2. **Documentation-lift evals**：在 Docker 中分别运行 `without_docs` 与 `with_docs` 两个镜像，配对比较通过率，衡量文档对 Agent 表现的增益。

源码：[packages/evals](../packages/evals/)

## 2. 目录

```text
packages/evals/
├─ evals/                    # 评测定义
│  ├─ *.docs.eval.ts         # 文档增益评测
│  ├─ *.eval.ts              # host 评测
│  ├─ acme-server.ts
│  └─ configured-runtime.ts
├─ src/
│  ├─ cli.ts                 # 文档评测编排
│  ├─ docker.ts              # 构建镜像、发现用例、运行单臂
│  ├─ plan.ts                # 展开 (case, variant, repetition)
│  ├─ report.ts              # 读取 Vitest JSON、配对、计算 lift
│  └─ harness.ts             # vitest-evals 适配器
├─ docker/                   # 镜像构建
├─ test/                     # 评测框架自身测试
└─ package.json
```

## 3. 文件约定

- `*.docs.eval.ts`：文档增益评测；`eval:docs` 在隔离容器中运行 `without_docs` / `with_docs`。
- 其他 `*.eval.ts`：host 评测；`eval:host` 用 Vitest 在本机运行。
- 评测文件只包含场景设置、模型任务、确定性评分；变体、重复、隔离、身份、持久化、报告由外层 runner 负责。

示例：

```ts
import { describeEval, StructuredOutputJudge } from "vitest-evals";
import { createPiDocumentationEvalHarness } from "../src/harness.ts";

const harness = createPiDocumentationEvalHarness();
const judge = StructuredOutputJudge({ expected: { ok: true }, match: "strict", allowExtras: false });

describeEval("Target workflow", { harness, judges: [judge], judgeThreshold: null }, (it) => {
  it("completes the task", async ({ run }) => {
    await run("Complete the target task.");
  });
});
```

对比评分使用 `judgeThreshold: null`：低分是数据，不是基础设施失败。

## 4. 运行

需要 `PI_PROVIDER` 与 `PI_MODEL`。

```bash
# host + docs
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol npm run eval -w packages/evals

# 仅 host
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol npm run eval:host -w packages/evals

# 单个 host 套件
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol \
  npm run eval:host -w packages/evals -- evals/documentation-audit.eval.ts

# 文档对比
npm run eval:docs -w packages/evals -- --provider openai-codex --model gpt-5.6-sol

# 每个变体多次
npm run eval:docs -w packages/evals -- evals/extensions.docs.eval.ts --runs-per-variant 5
```

`PI_EVAL_RUNS_PER_VARIANT` 等价于 `--runs-per-variant`。Vitest 过滤在发现阶段应用。

## 5. 文档对比流程

1. 临时挂载仓库做 Docker 构建，按仓库的 consumer-install 机制打包当前 workspace 包。
2. 基于同一运行时分别生成 `without_docs` 与 `with_docs` 镜像。
3. 在两个镜像中发现用例，要求 cohort 完全一致。
4. 规划每个 `(case, variant, model, runNumber)` 臂。
5. 每个臂在全新容器中运行；失败/缺失记录后继续。
6. 读取 Vitest JSON；配对完全一致的臂并计算差值。
7. 任一配对阻塞则扣留 headline pass rate，进程非零退出。

重复顺序按 run number 交替，降低顺序偏差。

## 6. 文档变体

- `without_docs`：移除 coding-agent 的 `README.md`、`CHANGELOG.md`、`docs/`、`examples/`，并从默认系统提示中移除 Pi 文档路由段。
- `with_docs`：保留这些文件与完整默认提示。

两个变体安装相同的本地 workspace tarball；内部依赖的文档/源码对称移除，避免充当额外指令。启动时校验镜像 allowlist，并确认 coding-agent 从 `dist/` 解析。

文档评测默认只允许 `read`、`write`、`edit`、`grep`、`find`、`ls`，不暴露 shell 或 web 搜索。Provider 流量仍需要网络，因此 Docker 不能证明 Agent 写的任意代码永不使用网络。

## 7. 产物

每次运行生成 `.eval/<timestamp>_<id>/`：

| 文件 | 内容 |
|---|---|
| `protocol.json` | 模型、镜像 ID、用例、任务、协议摘要 |
| `expected-runs.json` | 完整计划 cohort |
| `observations.jsonl` | 归一化结果与遥测 |
| `tasks/*/vitest.json` | 每个隔离臂的原生 JSON |
| `<variant>/sessions/*/session.jsonl` | 原生 Pi 会话 |
| `report.json` / `report.txt` | 配对比较 |

配对有效的条件：两个臂都恰好产生一个分数。缺失、重复、跳过、pending、未评分、报错都会阻塞配对。遥测缺失保持 unavailable，不当作 0。

报告会标记：无增益、负差值、饱和的 control/treatment、观察到的 flakiness。一次重复无法证明稳定性。

## 8. 依赖

`@vitest-evals/core`、`vitest-evals`、`autoevals`、`vitest`、`@earendil-works/pi-ai`、`@earendil-works/pi-coding-agent`。

## 9. 测试

```bash
cd packages/evals
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run --config vitest.test.config.ts
```
