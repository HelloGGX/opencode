# 第 6 章：AI 集成：大模型 API 的抽象与接入

## 6.0 引言：从配置到 AI 模型

在上一章中，我们构建了一个支持多层级、具备类型安全校验的配置系统。假设现在我们要在 CLI 工具中接入大模型 API，让 Agent 能够回答用户问题、执行工具调用。

最容易想到的办法是直接调用提供商的 SDK。但这样做会遇到什么问题？我们一步步来推导。

## 6.1 V1：直接调用 SDK

### 6.1.1 最直觉的实现

假设我们需要在 CLI 工具中接入 GPT-4。最直观的做法是安装官方 SDK，然后直接调用：

```bash
bun add openai
```

```typescript
// packages/opencode/src/ai/v1.ts
import OpenAI from "openai"

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
})

export async function chat(prompt: string): Promise<string> {
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
  })
  return response.choices[0].message.content ?? ""
}
```

调用方式很简单：

```typescript
const answer = await chat("什么是 SIMD 指令？")
console.log(answer)
```

功能上没有问题。但运行一段时间后，我们会发现几个问题。

### 6.1.2 问题一：多模型扩展困难

假设我们需要同时支持 Anthropic 的 Claude：

```typescript
// packages/opencode/src/ai/v1-anthropic.ts
import Anthropic from "@anthropic-ai/sdk"

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

export async function chat(prompt: string): Promise<string> {
  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20240620",
    messages: [{ role: "user", content: prompt }],
    max_tokens: 1024,
  })
  return response.content[0].type === "text"
    ? response.content[0].text
    : ""
}
```

可以看到，两套 API 在以下方面存在差异：

| 方面 | OpenAI | Anthropic |
|------|--------|-----------|
| 创建方法 | `chat.completions.create()` | `messages.create()` |
| 模型参数 | `model: "gpt-4o"` | `model: "claude-3-5-sonnet-20240620"` |
| 最大令牌 | 默认自动 | 必须显式指定 `max_tokens` |
| 响应结构 | `response.choices[0].message.content` | `response.content[0].text` |

如果我们继续添加 Google Gemini、AWS Bedrock、OpenRouter 等提供商，每套都需要独立的 SDK 初始化、错误处理、Token 计算逻辑。代码会变成一团乱麻。

### 6.1.3 问题二：流式响应处理复杂

假设用户发起一个复杂问题，响应文本很长。如果我们等模型生成完再展示，用户会等待很久。更好的做法是流式输出。

OpenAI SDK 的流式调用：

```typescript
export async function* streamChat(prompt: string) {
  const stream = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
    stream: true,
  })

  for await (const chunk of stream) {
    const text = chunk.choices[0]?.delta?.content
    if (text) {
      yield text
    }
  }
}
```

Anthropic SDK 的流式调用：

```typescript
export async function* streamChat(prompt: string) {
  const stream = await client.messages.stream({
    model: "claude-3-5-sonnet-20240620",
    messages: [{ role: "user", content: prompt }],
    max_tokens: 1024,
  })

  for await (const chunk of stream) {
    if (chunk.type === "content_block_delta") {
      yield chunk.text
    }
  }
}
```

问题出现了：两套 SDK 的流式响应格式完全不同。如果我们在业务代码中直接处理这些差异，后续切换模型就需要大幅修改。

### 6.1.4 V1 方案的总结

```
┌─────────────────────────────────────────────────────────┐
│ 业务代码                                                │
│  chat() { openai-sdk... }                               │
│  chat() { anthropic-sdk... }  ← 重复逻辑                 │
│  streamChat() { openai-stream... }                      │
│  streamChat() { anthropic-stream... } ← 更多重复        │
└─────────────────────────────────────────────────────────┘
```

V1 方案的缺陷：

1. **代码重复**：每个提供商都需要独立的调用逻辑
2. **难以切换**：切换模型需要修改多处代码
3. **流式处理复杂**：不同提供商的流式格式不一致
4. **测试困难**：需要 mock 各个提供商的 SDK

## 6.2 V2：引入 AI SDK

### 6.2.1 为什么选择 AI SDK

为了解决 V1 的问题，我们需要一层抽象，让上层业务代码与具体的模型提供商解耦。

直觉上，我们会定义一个统一接口，然后为每个提供商实现这个接口。但这样做有两个额外成本：

1. **接口维护成本**：每次新模型发布可能需要扩展接口
2. **社区生态割裂**：无法直接复用社区中的工具链

更好的方案是选择一个已经完成这套抽象的社区层。Vercel 推出的 [AI SDK](https://sdk.vercel.ai) 已经成为 TypeScript 生态中接入大模型的事实标准。

AI SDK 的核心价值：

1. **统一接口**：`streamText` 函数无论底层是 OpenAI 还是 Anthropic，调用方式完全一致
2. **流式响应支持**：内置对 Server-Sent Events (SSE) 的处理
3. **工具调用抽象**：为 Agent 场景提供标准化的工具定义
4. **类型安全**：完整的 TypeScript 类型推导

### 6.2.2 V2 的简单实现

安装依赖：

```bash
bun add ai @ai-sdk/openai @ai-sdk/anthropic
```

用 AI SDK 重写：

```typescript
// packages/opencode/src/ai/v2.ts
import { streamText } from "ai"
import { openai } from "@ai-sdk/openai"
import { anthropic } from "@ai-sdk/anthropic"

export async function* streamChat(prompt: string, provider: "openai" | "anthropic") {
  const model = provider === "openai"
    ? openai("gpt-4o")
    : anthropic("claude-3-5-sonnet")

  const result = await streamText({
    model,
    messages: [{ role: "user", content: prompt }],
  })

  for await (const chunk of result.fullStream) {
    if (chunk.type === "text-delta") {
      yield chunk.textDelta
    }
  }
}
```

调用方式变得统一了：

```typescript
for await (const text of streamChat("什么是 SIMD 指令？", "openai")) {
  process.stdout.write(text)
}
```

切换模型只需要改一个参数：

```typescript
for await (const text of streamChat("什么是 SIMD 指令？", "anthropic")) {
  process.stdout.write(text)
}
```

### 6.2.3 V2 方案的问题

但 V2 方案还有一个问题：我们在业务代码中直接指定使用哪个模型。实际项目中，模型选择应该由配置系统决定，而不是硬编码。

```typescript
// V2 的问题：模型选择硬编码在业务代码中
const model = provider === "openai"
  ? openai("gpt-4o")  // ← 这里写死了
  : anthropic("claude-3-5-sonnet")
```

此外，随着提供商数量增加，我们需要一个统一的地方来管理所有提供商的 SDK 配置（API Key、Base URL、Timeout 等）。

## 6.3 V3：引入 Provider 层

### 6.3.1 为什么需要 Provider 层

为了解决 V2 的问题，我们需要：

1. **统一管理提供商配置**：API Key、Base URL、Timeout 等
2. **模型注册机制**：让配置系统可以通过 `provider/model` 格式选择模型
3. **动态加载**：按需加载提供商 SDK，减少包体积

查看实际项目代码 `packages/opencode/src/provider/provider.ts`，我们可以梳理出 Provider 层的核心结构：

```typescript
export class Service extends Context.Service<Service, Interface>()("@opencode/Provider") {}
```

这是一个 Effect Context Service，它依赖以下服务：

- **Auth.Service**：认证信息管理
- **Config.Service**：配置读取
- **Plugin.Service**：插件扩展
- **Env.Service**：环境变量

### 6.3.2 模型标识符的设计

在实际的 OpenCode 项目中，配置系统已经定义了模型标识符的格式：`provider/model`，例如 `openai/gpt-4o-mini` 或 `anthropic/claude-3-5-sonnet`。

这种设计有三点考量：

1. **消除歧义**：`gpt-4o` 在不同提供商可能有不同含义，但 `openai/gpt-4o` 明确指向 OpenAI 的版本
2. **配置简洁**：用户只需指定一个字符串即可选择模型和提供商
3. **成本计算**：统一格式便于实现跨提供商的成本对比

### 6.3.3 提供商注册机制

实际项目维护了一个 `BUNDLED_PROVIDERS` 注册表，包含 20+ 个预置提供商：

```typescript
// provider.ts:70-95
const BUNDLED_PROVIDERS: Record<string, () => Promise<...>> = {
  "@ai-sdk/anthropic": () => import("@ai-sdk/anthropic").then((m) => m.createAnthropic),
  "@ai-sdk/openai": () => import("@ai-sdk/openai").then((m) => m.createOpenAI),
  "@ai-sdk/google-vertex": () => import("@ai-sdk/google-vertex").then((m) => m.createVertex),
  "@ai-sdk/openai-compatible": () => import("@ai-sdk/openai-compatible").then((m) => m.createOpenAICompatible),
  "@openrouter/ai-sdk-provider": () => import("@openrouter/ai-sdk-provider").then((m) => m.createOpenRouter),
  "gitlab-ai-provider": () => import("gitlab-ai-provider").then((m) => m.createGitLab),
  // ... 共 20+ 个
}
```

每个提供商通过动态导入的方式加载。这种设计有三点好处：

1. **按需加载**：用户只需要为自己的配置加载对应提供商的 SDK
2. **减少包体积**：不会把所有提供商的 SDK 都打包进主程序
3. **版本独立**：每个提供商可以使用自己兼容的 SDK 版本

### 6.3.4 V3 的 Provider 实现

基于以上分析，我们可以设计一个简化版的 Provider：

```typescript
// packages/opencode/src/provider/provider-v3.ts
import { Effect, Context } from "effect"

// 模型标识符类型
export type ModelIdentifier = string & { readonly __brand: unique symbol }

// 提供商信息
export interface ProviderInfo {
  id: string
  npm: string  // SDK 包名
  api?: string // API ID
}

// 提供商注册表
const PROVIDERS: Map<string, ProviderInfo> = new Map([
  ["openai", { id: "openai", npm: "@ai-sdk/openai" }],
  ["anthropic", { id: "anthropic", npm: "@ai-sdk/anthropic" }],
])

// 解析模型标识符
export function parseModel(model: string): { provider: string; model: string } {
  const [provider, ...rest] = model.split("/")
  return { provider, model: rest.join("/") }
}

// Provider Service 接口
export interface Interface {
  getLanguageModel(model: string): Effect.Effect<any>
}

// 创建动态加载的 Provider Service
export const make = Effect.gen(function* () {
  const cache = new Map<string, any>()

  const getLanguageModel: Interface["getLanguageModel"] = Effect.fn(
    "Provider.getLanguageModel"
  )(function* (modelId: string) {
    // 检查缓存
    if (cache.has(modelId)) {
      return cache.get(modelId)
    }

    // 解析模型标识符
    const { provider, model } = parseModel(modelId)

    // 获取提供商信息
    const info = PROVIDERS.get(provider)
    if (!info) {
      return Effect.fail(new Error(`Unknown provider: ${provider}`))
    }

    // 动态加载 SDK
    const sdk = yield* Effect.promise(() => import(info.npm))
    const createProvider = sdk[`create${capitalize(provider)}`]
    const providerInstance = yield* Effect.promise(() => createProvider())
    const languageModel = yield* Effect.promise(() => providerInstance.languageModel(model))

    // 缓存结果
    cache.set(modelId, languageModel)
    return languageModel
  })

  return { getLanguageModel } satisfies Interface
})

function capitalize(s: string): string {
  return s.charAt(0).toUpperCase() + s.slice(1)
}
```

### 6.3.5 V3 方案的问题

V3 方案解决了提供商管理的问题，但还有一个关键缺陷：**流式响应没有复用 Effect 的 Stream 类型**。

在 V2 中，我们直接使用 `for await` 遍历 AI SDK 的异步迭代器。这在简单场景下没问题，但在复杂场景下会遇到问题：

1. **无法组合**：无法使用 `Stream.map`、`Stream.filter` 等组合子
2. **错误处理不一致**：Effect 有统一的错误抽象，但异步迭代器没有
3. **生命周期管理**：Effect 的 `acquireRelease` 模式无法直接用于异步迭代器

## 6.4 V4：流式响应与 Effect Stream

### 6.4.1 为什么需要 Effect Stream

在 CLI 环境中，用户体验至关重要。当模型生成较长的输出时，流式响应可以带来两个优势：

1. **感知到的延迟降低**：用户可以在第一个 token 生成后立即看到输出
2. **进度可见**：对于需要较长时间生成的回答，用户能感受到模型的"思考"过程

Effect 框架提供了 `Stream` 类型，它提供了更好的组合能力。调用方可以通过 `Stream.map`、`Stream.filter` 等组合子对流进行转换，而不需要直接处理异步迭代器。

### 6.4.2 LLM Service 的设计

查看实际项目代码 `packages/opencode/src/session/llm.ts`，LLM Service 提供了 `stream` 方法：

```typescript
// llm.ts:52-55
export interface Interface {
  readonly stream: (input: StreamInput) => Stream.Stream<Event, unknown>
}
```

返回值是 Effect 的 `Stream` 类型。让我们看看 `StreamInput` 的定义：

```typescript
// llm.ts:33-48
export type StreamInput = {
  user: MessageV2.User
  sessionID: string
  parentSessionID?: string
  model: Provider.Model
  agent: Agent.Info
  permission?: Permission.Ruleset
  system: string[]
  messages: ModelMessage[]
  small?: boolean
  tools: Record<string, Tool>
  retries?: number
  toolChoice?: "auto" | "required" | "none"
}
```

这里 `model` 的类型是 `Provider.Model`，它包含了提供商 ID、模型 ID、以及各种选项。

### 6.4.3 V4 的流式实现

基于以上分析，我们可以设计一个简化版的 LLM Service：

```typescript
// packages/opencode/src/session/llm-v4.ts
import { Effect, Stream, Context, Layer } from "effect"
import { streamText, type ModelMessage, type Tool } from "ai"

export type StreamInput = {
  model: any // AI SDK 的语言模型实例
  messages: ModelMessage[]
  tools?: Record<string, Tool>
  abortSignal?: AbortSignal
}

export type Event = {
  type: "text-delta" | "tool-call" | "finish"
  content: string
}

// LLM Service 接口
export interface Interface {
  readonly stream: (input: StreamInput) => Stream.Stream<Event, Error>
}

// 实现
export const make = (getLanguageModel: (modelId: string) => Effect.Effect<any>) => {
  const run = Effect.fn("LLM.run")(function* (input: StreamInput & { abort: AbortSignal }) {
    const languageModel = yield* getLanguageModel(input.modelId)

    const result = streamText({
      model: languageModel,
      messages: input.messages,
      tools: input.tools,
      abortSignal: input.abort,
    })

    return result
  })

  const stream: Interface["stream"] = (input) =>
    Stream.scoped(
      Stream.unwrap(
        Effect.gen(function* () {
          // 创建 AbortController
          const ctrl = yield* Effect.acquireRelease(
            Effect.sync(() => new AbortController()),
            (ctrl) => Effect.sync(() => ctrl.abort()),
          )

          // 执行 AI 调用
          const result = yield* run({ ...input, abort: ctrl.signal })

          // 转换为 Effect Stream
          return Stream.fromAsyncIterable(result.fullStream, (e) =>
            e instanceof Error ? e : new Error(String(e))
          )
        })
      )
    )

  return { stream }
}
```

关键改进：

1. **Stream 替代 AsyncIterable**：`Stream.Stream<Event, Error>` 提供了更好的类型安全和组合能力
2. **acquireRelease 模式**：确保 AbortController 在流结束后正确清理
3. **统一错误处理**：所有错误都通过 Effect 的错误类型处理

### 6.4.4 V4 方案的问题

V4 方案在流式响应上已经比较完善，但还有一个问题：**工具调用的处理**。

在 Agent 场景中，工具调用（Tool Calling）是核心能力。当模型决定调用一个工具时，我们需要：

1. 拦截工具调用事件
2. 执行实际的工具逻辑
3. 将结果返回给模型

这需要更复杂的状态管理和事件处理。

## 6.5 V5：工具调用与权限系统

### 6.5.1 工具调用的基本模式

AI SDK 提供了标准化的工具抽象：

```typescript
import { tool, jsonSchema } from "ai"

const tools = {
  readFile: tool({
    description: "Read the content of a file",
    inputSchema: jsonSchema({
      type: "object",
      properties: {
        path: { type: "string", description: "The file path to read" },
      },
      required: ["path"],
    }),
    execute: async ({ path }, { abortSignal }) => {
      // 实际的工具执行逻辑
      return await fs.readFile(path, "utf-8")
    },
  }),
}
```

`tool` 函数接受两个参数：

1. **`inputSchema`**：定义工具的输入参数，用于让模型理解如何调用
2. **`execute`**：实际的工具执行逻辑

### 6.5.2 权限系统集成

实际项目中，工具的注册和执行通过 `resolveTools` 函数统一管理，并且引入了权限系统（Permission）：

```typescript
// llm.ts:430-436
function resolveTools(input: Pick<StreamInput, "tools" | "agent" | "permission" | "user">) {
  const disabled = Permission.disabled(
    Object.keys(input.tools),
    Permission.merge(input.agent.permission, input.permission ?? []),
  )
  return Record.filter(input.tools, (_, k) => input.user.tools?.[k] !== false && !disabled.has(k))
}
```

权限系统允许用户或企业管理员控制哪些工具可以被调用。这是一个重要的企业级特性。

### 6.5.3 LiteLLM 代理兼容处理

实际项目中还有一个特殊处理：LiteLLM 代理的兼容性问题。

LiteLLM 和某些 Anthropic 代理要求当消息历史中包含工具调用时，必须提供 tools 参数，即使当前没有活跃的工具。为了满足这个验证要求，需要注入一个 stub tool：

```typescript
// llm.ts:200-218
const isLiteLLMProxy =
  item.options?.["litellmProxy"] === true ||
  input.model.providerID.toLowerCase().includes("litellm") ||
  input.model.api.id.toLowerCase().includes("litellm")

if (
  (isLiteLLMProxy || input.model.providerID.includes("github-copilot")) &&
  Object.keys(tools).length === 0 &&
  hasToolCalls(input.messages)
) {
  tools["_noop"] = tool({
    description: "Do not call this tool. It exists only for API compatibility.",
    inputSchema: jsonSchema({
      type: "object",
      properties: { reason: { type: "string", description: "Unused" } },
    }),
    execute: async () => ({ output: "", title: "", metadata: {} }),
  })
}
```

这是一个**兼容性处理**的典型案例：为了适配不同的 API 行为，我们需要在边界处做一些特殊处理。

## 6.6 V6：与配置系统集成

### 6.6.1 模型选择的配置化

回顾第 5 章的配置系统，每个配置级别都可以指定模型：

```json
// 全局配置
{
  "model": "openai/gpt-4o-mini"
}

// 项目配置
{
  "model": "anthropic/claude-3-5-sonnet"
}
```

实际项目中的 `ConfigModelID` Schema `packages/opencode/src/config/model-id.ts` 定义了模型标识符的校验规则：

```typescript
const ModelId = Schema.String.pipe(
  Schema.pattern(/^[^\/]+\/[^\/]+$/, { description: "provider/model format" })
)
```

这种校验确保用户输入的模型标识符格式正确，避免运行时因格式错误导致的异常。

### 6.6.2 提供商配置的层级覆盖

Provider 的配置同样支持多层级覆盖。在 `ConfigProvider.Info` Schema 中，可以为每个提供商指定独立的 API Key、Base URL 等：

```typescript
// config/provider.ts
const ProviderInfo = Schema.Struct({
  api: Schema.optional(Schema.String),
  options: Schema.optional(
    Schema.Struct({
      apiKey: Schema.optional(Schema.String),
      baseURL: Schema.optional(Schema.String),
      timeout: Schema.optional(PositiveInt),
    }),
  ),
})
```

实际加载时，Provider Service 会按照以下优先级合并配置：

1. 环境变量（`ANTHROPIC_API_KEY` 等）
2. 用户全局配置（`~/.config/opencode/opencode.json`）
3. 项目配置（`./opencode.json`）
4. 托管配置（企业强制策略）

### 6.6.3 最终的集成架构

经过 V1 到 V6 的演进，我们得到了一个完整的 AI 集成架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                      业务代码 (Agent)                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      LLM Service                                 │
│  stream(input: StreamInput): Stream<Event>                       │
│  - 统一流式响应接口                                                │
│  - 工具调用处理                                                   │
│  - 权限系统集成                                                   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Provider Service                            │
│  getLanguage(model: Provider.Model): Effect<LanguageModel>       │
│  - 动态加载提供商 SDK                                             │
│  - 提供商配置管理                                                 │
│  - 模型注册表                                                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI SDK                                      │
│  streamText({ model, messages, tools })                          │
│  - OpenAI / Anthropic / Google Vertex / ...                      │
└─────────────────────────────────────────────────────────────────┘
```

## 6.7 本章小结

### 演进路径总结

| 版本 | 问题 | 解决方案 |
|------|------|----------|
| V1 | 直接调用 SDK，代码重复，难以扩展 | - |
| V2 | 引入 AI SDK，统一接口 | AI SDK 的 `streamText` |
| V3 | 提供商管理混乱 | Provider 层，动态加载 |
| V4 | 流式响应缺乏组合能力 | Effect Stream |
| V5 | 工具调用和权限控制缺失 | 工具抽象 + Permission 系统 |
| V6 | 模型选择硬编码 | 配置系统集成 |

### 核心设计

1. **AI SDK 作为抽象层**：通过 `streamText` 和统一的 Model 接口，隔离底层提供商的差异
2. **Provider Service 负责管理**：动态加载提供商 SDK，维护模型注册表
3. **流式响应 + 工具调用**：为 Agent 场景提供标准化的能力
4. **配置驱动**：通过第 5 章的配置系统，实现模型和提供商的全层级配置

### 关键代码位置

| 功能 | 文件位置 |
|------|---------|
| Provider Service | `packages/opencode/src/provider/provider.ts` |
| LLM Service | `packages/opencode/src/session/llm.ts` |
| 模型 Schema | `packages/opencode/src/provider/models.ts` |
| Provider 配置 | `packages/opencode/src/config/provider.ts` |

### 设计权衡

| 问题 | 选择 | 替代方案 | 权衡分析 |
|------|------|----------|----------|
| 提供商抽象 | AI SDK | 自定义接口 | AI SDK 生态成熟，工具链丰富，但增加依赖 |
| 工具调用 | AI SDK tool | 独立实现 | 标准化但灵活性受限 |
| 流处理 | Effect Stream | AsyncIterable | Effect 提供更好的组合能力和错误处理 |
| SDK 加载 | 动态 import | 静态 bundle | 按需加载减少体积，但首次调用有延迟 |
| 权限控制 | Permission 系统 | 无限制 | 企业级安全，但增加复杂度 |

**下一章预告**：第 7 章 - 会话管理，我们将学习如何维护多轮对话的上下文，以及如何实现会话的持久化和恢复。
