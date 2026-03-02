# 第 6 章：大模型 API 接入与命令交互机制

在上一章中，我们搭建了 CLI 工具的运行骨架，并成功接管了终端输入。当我们在终端键入 `opencode --version` 时，程序已经能够准确响应。本章我们将聚焦核心业务逻辑：让 CLI 与大语言模型（LLM）建立网络通信，并妥善处理鉴权、环境隔离以及工程架构的解耦。

## 6.1 基于 HTTP 协议的初步接入

当我们需要在 Node.js 或 Bun 环境中调用 OpenAI 的能力时，最直观的方案是遵循其官方的 REST API 规范，通过原生的 HTTP 请求实现。

我们先在 `packages/opencode/src/index.ts` 中注册一个 `run` 命令，并在其 Handler 中发起请求：

```typescript
// packages/opencode/src/index.ts
import yargs from "yargs"
import { hideBin } from "yargs/helpers"
// ...省略包版本导入

const cli = yargs(hideBin(process.argv))

cli
  // ...省略 version 与 help 注册
  .command(
    "run <message>",
    "Send a message to AI",
    (yargs) => yargs.positional("message", { type: "string" }),
    async (argv) => {
      // 构造符合 OpenAI 规范的请求体
      const payload = {
        model: "gpt-4o-mini",
        messages: [{ role: "user", content: argv.message }],
      }

      const response = await fetch("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          // 注意这里的鉴权方式
          "Authorization": `Bearer sk-your-api-key-here`,
        },
        body: JSON.stringify(payload),
      })

      const data = await response.json()
      // 解析出响应实体
      console.log(data.choices[0].message.content)
    }
  )
  .demandCommand(1)
  .parse()

```

在终端中执行测试：

```bash
$ opencode run "Hello, who are you?"
I'm an AI assistant created by OpenAI. How can I help you today?

```

终端正确打印了模型的响应，说明链路已经打通。但从工程安全的角度审视，这段代码存在致命缺陷：**将鉴权凭证（Credentials）硬编码在了源码中**。一旦代码被推送到公开的 Git 仓库，API Key 会立刻被安全爬虫捕获并面临盗用风险。

## 6.2 鉴权隔离：引入环境变量

处理敏感配置的业界标准机制是使用环境变量。操作系统会在进程启动时将环境字典注入到进程的内存空间中。在 Node.js 或 Bun 中，我们可以通过 `process.env` 对象访问它们。

我们需要修改 Handler 中的鉴权逻辑：

```typescript
    async (argv) => {
      // 从进程环境中读取凭证
      const apiKey = process.env.OPENAI_API_KEY
      if (!apiKey) {
        console.error("Error: OPENAI_API_KEY environment variable is not set")
        process.exit(1) // 异常退出，状态码非 0
      }

      const response = await fetch("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${apiKey}`,
        },
        // ...省略 body
      })
      // ...
    }

```

现在，必须在终端执行时显式注入环境变量：

```bash
$ OPENAI_API_KEY="sk-your-real-api-key" opencode run "Hello"

```

为了避免每次手动输入，通常的做法是在项目根目录建立 `.env` 文件。Bun 的运行时内部集成了对 `.env` 文件的自动解析机制，我们只需创建文件：

```bash
# .env
OPENAI_API_KEY=sk-your-real-api-key

```

由于 `.env` 包含了真实凭证，我们在初始化工程时已经将其加入了 `.gitignore` 规则，确保它被隔离在版本控制之外。

## 6.3 应对 API 碎片化：引入抽象层

设想一个常见的演进场景：我们需要将底层模型从 OpenAI 切换为 Anthropic 的 Claude。

如果我们查阅 Anthropic 的官方文档，会发现其接口设计与 OpenAI 存在显著差异。端点变成了 `https://api.anthropic.com/v1/messages`，必须显式传递 `max_tokens`，且鉴权协议不再使用标准的 `Bearer Token`，而是使用了自定义 Header（`x-api-key` 与 `anthropic-version`）。

如果基于现有的架构继续硬写，代码势必会退化为繁琐的分支判断：

```typescript
// 典型的反模式：高耦合业务逻辑
async function callAI(provider: string, prompt: string) {
  if (provider === "openai") {
    // 构建 OpenAI 请求...
  } else if (provider === "anthropic") {
    // 构建 Anthropic 请求...
  }
}

```

随着支持的厂商增加，这个函数会急剧膨胀，难以维护。在软件工程中，解决异构接口的最优解是**适配器模式（Adapter Pattern）**。我们需要一个统一的抽象层，屏蔽底层各家 API 的网络层和数据结构差异。

业界已经有成熟的开源实现，这里我们引入 Vercel AI SDK。它为我们抹平了这些差异，并提供了高度一致的 I/O 接口。

安装依赖：

```bash
$ bun add ai @ai-sdk/openai @ai-sdk/anthropic

```

接下来重构 `run` 命令：

```typescript
// packages/opencode/src/index.ts
import { generateText } from "ai"
import { openai } from "@ai-sdk/openai"
import { anthropic } from "@ai-sdk/anthropic"
// ...省略其他导入

cli
  .command(
    "run <message>",
    "Send a message to AI",
    (yargs) => yargs
        .positional("message", { type: "string" })
        // 新增 option，允许用户指定厂商和模型
        .option("model", {
          type: "string",
          description: "Format: provider:model",
          default: "openai:gpt-4o-mini",
        }),
    async (argv) => {
      // 1. 解析 provider 和 模型名称
      const [provider, modelName] = argv.model.split(":")
      
      // 2. 获取对应的 SDK 实例（适配器）
      const client = provider === "anthropic" ? anthropic : openai

      // 3. 调用统一的 generateText 接口
      const { text } = await generateText({
        model: client(modelName),
        prompt: argv.message,
      })

      console.log(text)
    }
  )

```

重构后，系统不仅支持了多模型切换，还自动继承了 AI SDK 内部提供的网络重试（Retry）机制和更完善的类型提示。测试如下：

```bash
# 调用 OpenAI
$ opencode run "Hello" --model "openai:gpt-4o-mini"

# 无缝切换至 Claude
$ opencode run "Hello" --model "anthropic:claude-3-5-sonnet-20241022"

```

## 6.4 状态持久化：基于文件系统的配置管理

虽然我们通过 `--model` 参数提供了灵活的切换能力，但要求用户每次执行命令都携带长串的参数显然违背了人体工程学。我们需要为 CLI 引入**状态持久化**机制，即记录用户的偏好设置。

最简单的持久化方案是在磁盘上读写 JSON 文件。我们在项目执行的当前工作目录（`process.cwd()`）约定一个配置文件 `opencode.json`。

新增配置解析逻辑：

```typescript
// packages/opencode/src/index.ts
import fs from "fs"
import path from "path"

// 定义读取配置的机制
function loadConfig() {
  const configPath = path.join(process.cwd(), "opencode.json")
  if (fs.existsSync(configPath)) {
    const content = fs.readFileSync(configPath, "utf-8")
    return JSON.parse(content)
  }
  // 默认兜底配置
  return { model: "openai:gpt-4o-mini" }
}

const config = loadConfig()

// ...在命令注册时，将 default 值绑定为配置读取结果
.option("model", {
  type: "string",
  default: config.model, 
})

```

现在，只需创建 `opencode.json`：

```json
{
  "model": "anthropic:claude-3-5-sonnet-20241022"
}

```

再次执行 `opencode run "Hello"` 时，程序会自动向 Anthropic 发起请求。配置文件的引入，让 CLI 具备了上下文记忆能力。

## 6.5 架构演进：职责隔离

观察目前的 `index.ts`，我们会发现它同时承担了三个职责：配置解析（`loadConfig`）、CLI 路由定义（`yargs` 注册）、以及 AI 业务逻辑调用（`generateText`）。随着后续功能的增加，单文件架构将变得极其脆弱。

我们需要对系统进行模块化拆分。

### 6.5.1 剥离配置模块

在 `packages/opencode/src/config/config.ts` 中封装配置读取逻辑，并补充基本的容错处理：

```typescript
import fs from "fs"
import path from "path"

export interface Config {
  model: string
}

const DEFAULT_CONFIG: Config = {
  model: "openai:gpt-4o-mini",
}

export function loadConfig(): Config {
  const configPath = path.join(process.cwd(), "opencode.json")
  if (!fs.existsSync(configPath)) return DEFAULT_CONFIG

  try {
    const content = fs.readFileSync(configPath, "utf-8")
    return { ...DEFAULT_CONFIG, ...JSON.parse(content) }
  } catch (error) {
    // 捕获 JSON 格式错误等异常，保证系统不会因此崩溃
    console.warn("Failed to parse opencode.json, using defaults")
    return DEFAULT_CONFIG
  }
}

```

### 6.5.2 封装 Provider 抽象

将 AI 调用的细节收敛至独立的业务层 `packages/opencode/src/provider/provider.ts` 中：

```typescript
import { generateText } from "ai"
import { openai } from "@ai-sdk/openai"
import { anthropic } from "@ai-sdk/anthropic"

export type ProviderID = "openai" | "anthropic"

export async function chat(model: string, prompt: string): Promise<string> {
  const [provider, modelName] = model.split(":") as [ProviderID, string]
  const client = provider === "anthropic" ? anthropic : openai

  const { text } = await generateText({
    model: client(modelName),
    prompt,
  })

  return text
}

```

### 6.5.3 独立命令文件与透传支持

为了让 `index.ts` 纯粹地充当“命令注册总线”，我们将具体的命令也拆分出去。在此之前，我们先在 `src/cli/cmd/cmd.ts` 提供一个类型安全的包装器：

```typescript
import type { CommandModule } from "yargs"

// 增强 Yargs 的类型，支持 `--` 用于参数透传（如 opencode run "test" -- --verbose）
type WithDoubleDash<T> = T & { "--"?: string[] }

export function cmd<T, U>(input: CommandModule<T, WithDoubleDash<U>>) {
  return input
}

```

随后在 `src/cli/cmd/run.ts` 定义真实的命令结构：

```typescript
import { chat } from "../../provider/provider"
import { loadConfig } from "../../config/config"
import { cmd } from "./cmd"

const config = loadConfig()

export const RunCommand = cmd({
  command: "run <message>",
  describe: "Send a message to AI",
  builder: (yargs) => yargs
    .positional("message", { type: "string" })
    .option("model", { type: "string", default: config.model }),
  async handler(argv) {
    try {
      const response = await chat(argv.model, argv.message as string)
      console.log(response)
    } catch (error) {
      console.error("Error:", error instanceof Error ? error.message : error)
      process.exit(1)
    }
  },
})

```

完成拆分后，入口文件 `index.ts` 达到了高度精简，它的职责变得清晰且单一：

```typescript
import yargs from "yargs"
import { hideBin } from "yargs/helpers"
import { RunCommand } from "./cli/cmd/run"
import packageJson from "../../package.json" with { type: "json" }

yargs(hideBin(process.argv))
  .version(packageJson.version)
  .help()
  .command(RunCommand)
  .demandCommand(1, "You need at least one command before moving on")
  .parse()

```

## 6.6 局限性与思考

验证程序无误后，你会发现我们已经实现了一个标准且具备扩展架构的 AI CLI 工具。但在真实的开发场景中，这远远不够。

你看，上述的执行链路中潜藏着三个核心痛点：

1. **记忆缺失**：由于 HTTP 请求的无状态特性，每次执行 `run` 命令都是独立的上下文，模型无法结合历史输入进行多轮对话推演。
2. **终端阻塞**：API 的网络响应通常有数秒延迟。在这段时间内，终端处于假死状态，未向用户传递任何进行中的反馈。
3. **异常处理粗糙**：当触及 Token 限制或网络超时时，系统仅粗暴地输出错误堆栈。

要真正让大模型协助我们编程，需要建立持久化的“会话（Session）”机制。在下一章，我们将引入本地 SQLite 数据库，探讨如何实现多轮对话的上下文留存。
