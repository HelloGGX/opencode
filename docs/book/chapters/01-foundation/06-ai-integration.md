# 第 6 章：AI 集成：大模型 API 的抽象与接入

## 6.0 引言：当配置系统遇到大模型

在上一章中，我们构建了一个支持多层级、具备类型安全校验的配置系统。现在，我们来解决一个更具体的问题：如何在 CLI 工具中接入大模型 API？

最容易想到的办法是，针对每个模型提供商（如 OpenAI、Anthropic）直接调用其 SDK。但这会引发一个工程问题：如果某天需要切换提供商，或者同时支持多个提供商，代码将面临大量的重复逻辑和耦合。

本章我们从第一性原理出发，探讨如何设计一个灵活、可扩展的 AI 模型接入层。

## 6.1 抽象的必要性与直觉方案

### 6.1.1 直接调用的困境

假设我们需要在 CLI 工具中接入 GPT-4。最直观的做法是：

```typescript
import OpenAI from "openai"

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

async function chat(prompt: string) {
  const response = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
  })
  return response.choices[0].message.content
}
```

这段代码在功能上是正确的。但当我们需要扩展到 Anthropic 的 Claude 模型时，问题出现了：

```typescript
import Anthropic from "@anthropic-ai/sdk"

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY })

async function chat(prompt: string) {
  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20240620",
    messages: [{ role: "user", content: prompt }],
    max_tokens: 1024,
  })
  return response.content[0].type === "text" ? response.content[0].text : ""
}
```

可以看到，两套 API 在请求格式、参数命名、响应结构上都存在显著差异。如果我们继续添加 Google Gemini、AWS Bedrock 等提供商，每套都需要独立的 SDK 初始化、错误处理、Token 计算逻辑。

当产品提出"支持根据上下文自动选择最便宜的模型"这样的需求时，直接调用的方案会变得难以维护。

### 6.1.2 统一抽象的价值

为了解决这个困境，我们需要一层抽象，让上层业务代码与具体的模型提供商解耦。

直觉上，我们会定义一个统一接口：

```typescript
interface LanguageModel {
  chat(messages: Message[], options?: ChatOptions): Promise<ChatResult>
}
```

然后为每个提供商实现这个接口：

```typescript
class OpenAIModel implements LanguageModel { ... }
class AnthropicModel implements LanguageModel { ... }
```

这种方案确实能解决问题，但存在两个额外的工程成本：

1. **接口维护成本**：每次新模型发布可能需要扩展接口
2. **社区生态割裂**：无法直接复用社区中的工具链（如 LangChain、LlamaIndex 等）

更好的方案是选择一个已经完成这套抽象的社区层。

## 6.2 AI SDK：业界的事实标准

### 6.2.1 为什么选择 AI SDK

Vercel 推出的 [AI SDK](https://sdk.vercel.ai) 已经成为 TypeScript 生态中接入大模型的事实标准。它的核心价值在于：

1. **统一接口**：无论底层是 OpenAI、Anthropic 还是其他提供商，上层调用方式完全一致
2. **流式响应支持**：内置对 Server-Sent Events (SSE) 的处理
3. **工具调用（Tool Calling）**：为多模态 agent 提供标准化的工具抽象
4. **类型安全**：完整的 TypeScript 类型推导

对于 CLI 工具而言，AI SDK 的流式响应处理尤为重要。当模型生成较长的输出时，用户需要实时看到结果，而不是等待完整响应后再一次性展示。

### 6.2.2 核心 API 概览

AI SDK 的核心是 `streamText` 函数，它封装了流式响应的处理逻辑：

```typescript
import { streamText } from "ai"

const result = await streamText({
  model: openai("gpt-4o"),
  messages: [{ role: "user", content: "解释一下什么是 SIMD" }],
})

for await (const chunk of result.fullStream) {
  if (chunk.type === "text-delta") {
    process.stdout.write(chunk.textDelta)
  }
}
```

这里 `model` 参数接受一个统一的语言模型实例，而 `openai("gpt-4o")` 负责创建具体提供商的模型实例。这种设计让我们可以轻松切换底层提供商：

```typescript
import { streamText, wrapLanguageModel } from "ai"
import { anthropic } from "@ai-sdk/anthropic"

// 切换到 Anthropic，只需更换 model 参数
const result = await streamText({
  model: anthropic("claude-3-5-sonnet"),
  messages: [{ role: "user", content: "解释一下什么是 SIMD" }],
})
```

## 6.3 提供商抽象：Provider 层

### 6.3.1 模型标识符的设计

在实际的 OpenCode 项目中，配置系统已经定义了模型标识符的格式：`provider/model`，例如 `openai/gpt-4o-mini` 或 `anthropic/claude-3-5-sonnet`。

这种设计有三点考量：

1. **消除歧义**：`gpt-4o` 在不同提供商可能有不同含义，但 `openai/gpt-4o` 明确指向 OpenAI 的版本
2. **配置简洁**：用户只需指定一个字符串即可选择模型和提供商
3. **成本计算**：统一格式便于实现跨提供商的成本对比

### 6.3.2 Provider Service 的依赖结构

查看实际项目代码 [packages/opencode/src/provider/provider.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/provider/provider.ts)，我们可以梳理出 Provider 层的依赖关系：

```typescript
export class Service extends Context.Service<Service, Interface>()("@opencode/Provider") {}
```

Provider Service 是项目的核心模块，它依赖以下服务：

- **Auth.Service**：认证信息管理
- **Config.Service**：配置读取
- **Plugin.Service**：插件扩展
- **Env.Service**：环境变量

这种依赖注入设计让 Provider Service 可以在运行时获取完整的上下文信息，包括用户的 API Key、提供商配置、模型限额等。

### 6.3.3 提供商注册机制

实际项目维护了一个 `BUNDLED_PROVIDERS` 注册表 [provider.ts:70-95](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/provider/provider.ts#L70-L95)，包含 20+ 个预置提供商：

```typescript
const BUNDLED_PROVIDERS: Record<string, () => Promise<...>> = {
  "@ai-sdk/anthropic": () => import("@ai-sdk/anthropic").then((m) => m.createAnthropic),
  "@ai-sdk/openai": () => import("@ai-sdk/openai").then((m) => m.createOpenAI),
  "@ai-sdk/google-vertex": () => import("@ai-sdk/google-vertex").then((m) => m.createVertex),
  // ...
}
```

每个提供商通过动态导入的方式加载，这种设计有三点好处：

1. **按需加载**：用户只需要为自己的配置加载对应提供商的 SDK
2. **减少包体积**：不会把所有提供商的 SDK 都打包进主程序
3. **版本独立**：每个提供商可以使用自己兼容的 SDK 版本

## 6.4 流式响应的处理

### 6.4.1 为什么需要流式处理

在 CLI 环境中，用户体验至关重要。假设用户发起一个复杂的问题，流式响应可以带来两个优势：

1. **感知到的延迟降低**：用户可以在第一个 token 生成后立即看到输出
2. **进度可见**：对于需要较长时间生成的回答，用户能感受到模型的"思考"过程

### 6.4.2 LLM Service 的流式抽象

查看实际项目代码 [packages/opencode/src/session/llm.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/session/llm.ts)，LLM Service 提供了 `stream` 方法：

```typescript
export interface Interface {
  readonly stream: (input: StreamInput) => Stream.Stream<Event, unknown>
}
```

返回值是 Effect 的 `Stream` 类型，这提供了更好的组合能力。调用方可以通过 `Stream.map`、`Stream.filter` 等组合子对流进行转换，而不需要直接处理异步迭代器。

### 6.4.3 工具调用的处理

对于 Agent 场景，工具调用（Tool Calling）是核心能力。AI SDK 提供了标准化的工具抽象：

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

实际项目中，工具的注册和执行通过 `resolveTools` 函数 [llm.ts:430-436](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/session/llm.ts#L430-L436) 统一管理：

```typescript
function resolveTools(input: Pick<StreamInput, "tools" | "agent" | "permission" | "user">) {
  const disabled = Permission.disabled(
    Object.keys(input.tools),
    Permission.merge(input.agent.permission, input.permission ?? []),
  )
  return Record.filter(input.tools, (_, k) => input.user.tools?.[k] !== false && !disabled.has(k))
}
```

这里引入了权限系统（Permission），允许用户或企业管理员控制哪些工具可以被调用。

## 6.5 配置与 AI 的协同

### 6.5.1 模型选择的配置化

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

实际项目中的 `ConfigModelID` Schema [packages/opencode/src/config/model-id.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/config/model-id.ts) 定义了模型标识符的校验规则：

```typescript
const ModelId = Schema.String.pipe(Schema.pattern(/^[^\/]+\/[^\/]+$/, { description: "provider/model format" }))
```

这种校验确保用户输入的模型标识符格式正确，避免运行时因格式错误导致的异常。

### 6.5.2 提供商配置的层级覆盖

Provider 的配置同样支持多层级覆盖。在 `ConfigProvider.Info` Schema 中，可以为每个提供商指定独立的 API Key、Base URL 等：

```typescript
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

## 6.6 本章小结

### 核心设计

1. **AI SDK 作为抽象层**：通过 `streamText` 和统一的 Model 接口，隔离底层提供商的差异
2. **Provider Service 负责管理**：动态加载提供商 SDK，维护模型注册表
3. **流式响应 + 工具调用**：为 Agent 场景提供标准化的能力
4. **配置驱动**：通过第 5 章的配置系统，实现模型和提供商的全层级配置

### 关键代码位置

| 功能 | 文件位置 |
|------|---------|
| Provider Service | [src/provider/provider.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/provider/provider.ts) |
| LLM Service | [src/session/llm.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/session/llm.ts) |
| 模型 Schema | [src/provider/models.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/provider/models.ts) |
| Provider 配置 | [src/config/provider.ts](file:///Users/xiaoluo/Desktop/work/opencode/packages/opencode/src/config/provider.ts) |

### 设计权衡

| 问题 | 选择 | 替代方案 | 权衡分析 |
|------|------|----------|----------|
| 提供商抽象 | AI SDK | 自定义接口 | AI SDK 生态成熟，工具链丰富，但增加依赖 |
| 工具调用 | AI SDK tool | 独立实现 | 标准化但灵活性受限 |
| 流处理 | Effect Stream | AsyncIterable | Effect 提供更好的组合能力和错误处理 |
| SDK 加载 | 动态 import | 静态 bundle | 按需加载减少体积，但首次调用有延迟 |

**下一章预告**：第 7 章 - 会话管理，我们将学习如何维护多轮对话的上下文，以及如何实现会话的持久化和恢复。
