# 第 5 章：配置系统基础

> **本章目标**: 构建一个支持多层级配置的系统
>
> **起点**: 第 4 章的 CLI 框架
>
> **核心问题**:
> - 为什么一个 JSON 文件不够用？
> - 如何设计多层级配置的优先级？
> - 企业部署场景下如何强制配置？

---

在构建复杂的 CLI 工具链时，配置系统是不可或缺的基础设施。在开始讨论具体的实现之前，我们先来看一个真实的工程问题。

假设我们需要在 CLI 工具中接入 AI 大模型。最容易想到的办法是，直接在代码中硬编码默认的模型名称和提供商。但随着使用场景的扩展，有些项目需要使用 OpenAI，而有些项目为了处理更长的上下文，必须切换到 Anthropic。这意味着，我们需要一种机制，允许用户在外部定义并覆盖工具的内部默认行为。

这正是配置系统出现的原因。本章我们将从零开始，一步步推导并设计一个支持多层级、具备类型安全校验的健壮配置系统。

## 5.1 从单文件配置到格式演进

### 5.1.1 纯 JSON 的局限性

为了理解配置系统是如何工作的，我们先从最简单的实现开始。最直观的做法是在项目根目录下创建一个 `.json` 文件作为配置文件：

```bash
# 在 packages/opencode 目录下创建配置文件
cd packages/opencode
cat > opencode.json << 'EOF'
{
  "model": "gpt-4o-mini",
  "provider": "openai"
}
EOF
```

配套的读取逻辑非常简单，直接利用 Node.js 原生的文件系统模块和 `JSON.parse`：

创建一个配置加载器 `config/config.ts`，负责读取配置文件并解析为对象：
```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"

async function loadConfig() {
  const content = await fs.readFile("opencode.json", "utf-8")
  return JSON.parse(content)
}

const config = await loadConfig()
console.log(config)
```

运行上述代码：
```bash
# 在 packages/opencode 目录下执行
bun run src/config/config.ts
```
我们可以顺利拿到配置对象。但实际投入使用后，我们很快就会收到用户的反馈：他们希望在配置文件中添加注释，以说明某个参数的用途。

试想一下，当用户将配置修改为如下内容时：

```json
{
  "model": "gpt-4o-mini",
  // 临时切换为 openai 测试
  "provider": "openai"
}
```

再次执行读取逻辑，程序会直接崩溃：

```bash
SyntaxError: JSON Parse error: Unrecognized token '/'
```

为什么会这样？实际上，这并非 `JSON.parse` 的缺陷，而是 JSON 规范的有意为之。从本质上讲，JSON 被设计为纯粹的数据交换格式，其发明者 Douglas Crockford 移除了注释支持，以确保不同解析器行为的一致性。

但在配置文件的场景下，人类可读性和可维护性是刚需。这是一个典型的工程权衡：是坚持数据格式的纯粹性，还是妥协于开发体验？

### 5.1.2 引入 JSONC 支持

为了解决这个问题，我们通常会引入 JSONC（JSON with Comments）。JSONC 是在 JSON 语法基础上允许使用 `//` 和 `/* */` 注释的扩展格式，主流编辑器（如 VS Code）原生支持该格式。

首先安装 `jsonc-parser` 依赖：

```bash
cd packages/opencode
bun add jsonc-parser
```

然后替换原生的 `JSON.parse`，借助成熟的解析器来处理包含注释的文本：

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import { parse as parseJsonc } from "jsonc-parser"

async function loadConfig() {
  const content = await fs.readFile("opencode.json", "utf-8")
  // 即使包含注释，也能被正确解析为 JavaScript 对象
  return parseJsonc(content)
}

const config = await loadConfig()
console.log(config)
```

现在，配置文件不仅可以被程序正确解析，还能很好地承载开发者的意图。

## 5.2 边界数据的类型安全校验

### 5.2.1 运行时擦除引发的隐患

解决了文件格式问题后，接下来我们看看数据的正确性。假设用户在编写配置时发生了拼写错误，将 `provider` 错误地写成了 `providre`：

```json
{
  "model": "gpt-4o-mini",
  "providre": "openai"
}
```

如果在 TypeScript 中，我们通常会定义一个接口（Interface）来约束结构：

```ts
interface Config {
  model?: string
  provider?: "openai" | "anthropic"
}
```

但这里存在一个致命的问题：TypeScript 的类型检查仅在编译时（Compile-time）有效，在运行时（Runtime）这些类型信息会被完全擦除。换句话说，`parseJsonc` 返回的对象即便完全不符合 `Config` 接口，程序在加载配置这一步也不会报错。

这种缺陷会导致错误被延后。程序可能要在执行到发起网络请求的那一刻，才因为缺少 `provider` 字段而崩溃，此时抛出的错误栈往往极其深且难以溯源。

### 5.2.2 基于 Zod 的运行时校验

既然如此，我们就需要一种新的方案：能够在运行时对外部输入（边界数据）进行严格的结构化校验。这正是 Zod 等 Schema 校验库的核心设计目标。

首先安装 Zod 依赖：

```bash
cd packages/opencode
bun add zod
```

接下来，我们在 `packages/opencode/src/config/` 目录下创建配置模块。首先定义配置的 Schema（模式）：

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import z from "zod"
import { parse as parseJsonc } from "jsonc-parser"

// ========== 基础类型定义 ==========

// 模型 ID 格式：model (如 gpt-4o-mini)
const ModelId = z.string()

// ========== 配置 Schema ==========

// 基础配置 Schema（本章实现）
const BaseConfigSchema = z.object({
  model: ModelId.optional().describe("AI model to use, format: model"),
  provider: z.enum(["openai", "anthropic"]).optional().describe("AI provider"),
  apiKey: z.string().optional().describe("API key for the provider"),
})

// 从 Schema 自动推导 TypeScript 类型
type Config = z.infer<typeof BaseConfigSchema>

// ========== 配置加载器 ==========

async function loadConfig(): Promise<Config> {
  const content = await fs.readFile("opencode.json", "utf-8")
  const raw = parseJsonc(content)
  return BaseConfigSchema.partial().strict().parse(raw)
}

const config = await loadConfig()
console.log(config)
```

现在，如果我们再次加载包含拼写错误的配置，程序会在第一时间拦截并抛出精准的错误：

```bash
# 执行命令
bun run src/config/config.ts

# 终端输出
ZodError: [
  {
    "code": "unrecognized_keys",
    "keys": [
      "providre"
    ],
    "path": [],
    "message": "Unrecognized key: \"providre\""
  }
]
```

可以看到，错误信息直接指出了 `provider` 字段不符合预期。这种"尽早失败（Fail Fast）"的机制极大提升了工具的健壮性。

继续完善配置 Schema，当我们希望约束用户配置的model的格式，比如我们希望在模型名前加上提供商的前缀，比如 `openai/gpt-4o-mini`。

我们只需要修改 `ModelId` 定义，添加一个正则表达式校验即可：

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import z from "zod"

// ========== 基础类型定义 ==========

// 模型 ID 格式：provider/model (如 openai/gpt-4o-mini)
const ModelId = z.string().regex(/^[^\/]+\/[^\/]+$/, "Invalid model ID format")

```

执行代码：
```bash
# 执行命令
bun run src/config/config.ts
```

会有如下报错：
```bash
ZodError: [
  {
    "origin": "string",
    "code": "invalid_format",
    "format": "regex",
    "pattern": "/^[^\\/]+\\/[^\\/]+$/",
    "path": [
      "model"
    ],
    "message": "Invalid model ID format"
  }
]
```

于是我们将`opencode.json`的内容修改为：
```json
{
  "model": "openai/gpt-4o-mini",
  "provider": "openai"
}
```
没有报错了，说明配置格式符合要求。

### 5.2.3 扩展配置 Schema

## 5.3 多层级配置合并策略

### 5.3.1 配置层级的推演

随着 CLI 工具被广泛使用，单点配置的局限性开始暴露。试想如下场景：

一名开发者参与了公司内部的 10 个项目，他希望默认使用 `gpt-4o`，但唯独项目 A 需要使用 `claude-3-5-sonnet`。如果只有项目级配置，他必须在 9 个项目里重复配置 `gpt-4o`。

为了解决这个问题，我们需要引入多层级配置。在实际的 OpenCode 项目中，配置的来源分为以下几级（优先级从低到高）：

1. **全局配置（Global）**：存放在用户配置目录下（如 `~/.config/opencode/opencode.json`），代表用户的默认偏好。
2. **项目配置（Project）**：存放在当前工作目录下（如 `./opencode.json`），针对特定项目的重写。
3. **环境变量配置（Environment）**：通过 `OPENCODE_CONFIG` 指定，常用于 CI/CD 流程的动态注入。
4. **托管配置（Managed）**：存放在系统的全局共享目录（如 `/etc/opencode/opencode.json`），通常用于企业的强制管控策略。

为什么托管配置的优先级最高？从本质上讲，这是一种企业管理的工程权衡。如果公司采购了特定模型的内网私有部署，托管配置能够无视用户的全局或项目级设置，强制重写请求目标，从而保障数据安全。

这里有一个值得讨论的权衡：托管配置应该优先级最高吗？

**支持的观点**：企业需要管控。比如公司购买了特定模型的授权，不允许员工使用其他模型。托管配置优先级最高，可以确保策略被执行。

**反对的观点**：开发者可能需要调试。如果托管配置锁死了所有选项，开发者就无法临时切换模型进行测试。

一个折中方案是：托管配置只覆盖特定字段，而不是全部配置。这需要更细粒度的合并策略。但在本章，我们先实现最简单的"全量覆盖"方案。

### 5.3.2 跨平台路径管理

不同的操作系统针对"应用程序数据（Application Data）"有着完全不同的底层规范：

- **Linux**：倾向于遵循 XDG Base Directory 规范，用户级配置通常放在 `~/.config` 目录下，系统级放在 `/etc` 目录。
- **Windows**：有一套专属的环境变量，用户级数据存放在 `APPDATA`，系统级数据存放在 `ProgramData`。
- **macOS**：虽然底层是 Unix，但苹果有自己的应用沙盒惯例，系统级配置通常存放在 `/Library/Application Support`。

为了保证 CLI 工具的跨平台兼容性，我们需要在代码中抹平这些操作系统的底层差异。一种标准的做法是借助 `xdg-basedir` 库，它会自动处理不同平台的路径规范。

首先安装依赖：

```bash
cd packages/opencode
bun add xdg-basedir
```

接下来，我们创建一个全局路径管理模块：

```typescript
// packages/opencode/src/global/index.ts
import fs from "fs/promises"
import { xdgData, xdgCache, xdgConfig, xdgState } from "xdg-basedir"
import path from "path"
import os from "os"

const app = "opencode"

// xdg-basedir 会根据操作系统自动返回正确的路径
// Linux/macOS: ~/.local/share/opencode, ~/.config/opencode, etc.
// Windows: %LOCALAPPDATA%/opencode, %APPDATA%/opencode, etc.
const data = path.join(xdgData!, app)
const cache = path.join(xdgCache!, app)
const config = path.join(xdgConfig!, app)
const state = path.join(xdgState!, app)

export namespace Global {
  export const Path = {
    get home() {
      return process.env.OPENCODE_TEST_HOME || os.homedir()
    },
    data,
    bin: path.join(data, "bin"),
    log: path.join(data, "log"),
    cache,
    config,  // 全局配置目录
    state,
  }
}

// 确保目录存在
await Promise.all([
  fs.mkdir(Global.Path.data, { recursive: true }),
  fs.mkdir(Global.Path.config, { recursive: true }),
  fs.mkdir(Global.Path.state, { recursive: true }),
  fs.mkdir(Global.Path.log, { recursive: true }),
  fs.mkdir(Global.Path.bin, { recursive: true }),
])
```

这个模块的核心价值在于：通过 `xdg-basedir` 库，我们不需要手动判断 `process.platform`，库会自动根据操作系统返回符合规范的路径。

接下来，我们创建配置路径管理模块，专门处理配置文件的路径解析：

```typescript
// packages/opencode/src/config/paths.ts
import path from "path"
import { parse as parseJsonc, printParseErrorCode } from "jsonc-parser"
import { Global } from "../global"

export namespace ConfigPaths {
  // 托管配置目录：企业级强制配置，优先级最高
  export function managedDir(): string {
    switch (process.platform) {
      case "darwin":
        return "/Library/Application Support/opencode"
      case "win32":
        return path.join(process.env.ProgramData || "C:\\ProgramData", "opencode")
      default:
        return "/etc/opencode"
    }
  }

  // 全局配置文件路径
  export function globalConfigFile(): string {
    return path.join(Global.Path.config, "opencode.json")
  }

  // 项目配置文件路径
  export function projectConfigFile(cwd: string): string {
    return path.join(cwd, "opencode.json")
  }

  // 解析 JSONC 文本
  export function parseText(text: string, filepath: string) {
    // 支持 {env:VAR} 环境变量替换
    text = text.replace(/\{env:([^}]+)\}/g, (_, varName) => {
      return process.env[varName] || ""
    })

    const errors: any[] = []
    const data = parseJsonc(text, errors, { allowTrailingComma: true })
    
    if (errors.length) {
      const lines = text.split("\n")
      const errorDetails = errors.map((e) => {
        const beforeOffset = text.substring(0, e.offset).split("\n")
        const line = beforeOffset.length
        const column = beforeOffset[beforeOffset.length - 1].length + 1
        return `${printParseErrorCode(e.error)} at line ${line}, column ${column}`
      }).join("\n")
      
      throw new Error(`JSONC parse error in ${filepath}:\n${errorDetails}`)
    }
    
    return data
  }
}
```

### 5.3.3 深度合并（Deep Merge）的必要性

明确了优先级后，接下来我们需要处理配置合并逻辑。最容易想到的办法是使用 JavaScript 的对象展开运算符（Spread Operator）进行浅合并（Shallow Merge）：

```ts
const mergedConfig = { ...globalConfig, ...projectConfig }
```

但这会带来一个隐蔽的缺陷。假设我们的配置结构中包含嵌套对象：

```json
// 全局配置
{
  "model": "gpt-4o",
  "provider": {
    "openai": {
      "apiKey": "sk-xxx"
    }
  }
}

// 项目配置
{
  "provider": {
    "anthropic": {
      "apiKey": "sk-yyy"
    }
  }
}
```

如果使用浅合并，`projectConfig` 中的 `provider` 对象会直接替换掉 `globalConfig` 中的 `provider` 对象，导致 OpenAI 的配置完全丢失。

为了解决这个问题，我们必须使用深度合并（Deep Merge）算法，递归地遍历对象的每一个属性并进行合并。这里我们可以借助工具库 `remeda` 提供的 `mergeDeep` 函数：

```bash
cd packages/opencode
bun add remeda
```

### 5.3.4 实现配置加载器

现在我们将所有组件整合起来，实现完整的配置加载逻辑：

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import path from "path"
import z from "zod"
import { mergeDeep } from "remeda"
import { ConfigPaths } from "./paths"

// ========== Schema 定义 ==========
const ModelId = z.string().regex(/^[^\/]+\/[^\/]+$/, "Invalid model ID format")

const ConfigSchema = z.object({
  model: ModelId.optional(),
  provider: z.enum(["openai", "anthropic"]).optional(),
  apiKey: z.string().optional(),
  agent: z.record(z.string(), z.any()).optional(),
  plugin: z.array(z.string()).optional(),
  permission: z.record(z.string(), z.any()).optional(),
  instructions: z.array(z.string()).optional(),
})

type Config = z.infer<typeof ConfigSchema>

// 加载单个配置文件
async function loadFile(filePath: string): Promise<Partial<Config>> {
  try {
    const content = await fs.readFile(filePath, "utf-8")
    const raw = ConfigPaths.parseText(content, filePath)
    return ConfigSchema.partial().parse(raw)
  } catch (error) {
    // 文件不存在时返回空对象
    if ((error as NodeJS.ErrnoException).code === "ENOENT") {
      return {}
    }
    throw error
  }
}

// 加载并合并所有层级的配置
export async function load(cwd: string = process.cwd()): Promise<Config> {
  // 按优先级加载配置（从低到高）
  let result: Config = {}

  // 1. 全局配置 (最低优先级)
  result = mergeDeep(result, await loadFile(ConfigPaths.globalConfigFile()))

  // 2. 项目配置
  result = mergeDeep(result, await loadFile(ConfigPaths.projectConfigFile(cwd)))

  // 3. 环境变量指定的配置
  if (process.env.OPENCODE_CONFIG) {
    result = mergeDeep(result, await loadFile(process.env.OPENCODE_CONFIG))
  }

  // 4. 托管配置 (最高优先级，企业强制)
  result = mergeDeep(result, await loadFile(path.join(ConfigPaths.managedDir(), "opencode.json")))

  // 验证最终配置
  return ConfigSchema.parse(result)
}
```

这里采用了最简单的实现方式：逐个加载并深度合并。配置加载完成后，通过 `ConfigSchema.parse()` 进行最终校验。

### 5.3.5 验证多层级合并

让我们用实际输出来验证优先级是否正确。

首先创建全局配置：

```bash
# 创建全局配置目录
mkdir -p ~/.config/opencode

# 写入全局配置
cat > ~/.config/opencode/opencode.json << 'EOF'
{
  "provider": "openai",
  "model": "gpt-4o-mini"
}
EOF
```

然后创建项目配置：

```bash
# 在 packages/opencode 目录下
cat > ./opencode.json << 'EOF'
{
  "model": "claude-3-5-sonnet-20241022"
}
EOF
```

创建测试脚本：

```typescript
// packages/opencode/src/config/test-load.ts
import { load } from "./config"

const config = await load()
console.log(config)
```

运行测试：

```bash
bun run packages/opencode/src/config/test-load.ts
```

```text
# 输出:
{ provider: "openai", model: "claude-3-5-sonnet-20241022" }
```

可以看到：
- `provider` 来自全局配置
- `model` 被项目配置覆盖

这验证了我们的设计：项目配置的优先级高于全局配置。

## 5.4 配置加载流程

完整的配置加载流程如下：

```
  全局配置              项目配置           环境变量          托管配置
~/.config/opencode/  ./opencode.json  OPENCODE_CONFIG  /etc/opencode/
opencode.json                                          opencode.json
      │                   │                 │                │
      └───────────────────┴─────────────────┴────────────────┘
                                   │
                            mergeDeep() 深度合并
                                   │
                          ConfigSchema.parse() 类型验证
                                   │
                              最终配置对象
```

**优先级从低到高**：
1. 全局配置 `~/.config/opencode/opencode.json`
2. 项目配置 `./opencode.json`
3. 环境变量 `OPENCODE_CONFIG`
4. 托管配置 `/etc/opencode/opencode.json`（最高优先级，企业强制）

## 5.5 设计权衡总结

| 问题 | 选择 | 替代方案 | 权衡分析 |
|------|------|----------|----------|
| 配置格式 | JSONC | YAML、TOML | JSONC 兼容 JSON 生态，VS Code 用户熟悉；YAML 缩进敏感易出错 |
| 类型验证 | Zod | JSON Schema、手写 | Zod 提供编译时+运行时双重保障；JSON Schema 写法繁琐 |
| 合并策略 | 深度合并 | 浅合并 | 深度合并支持嵌套配置；浅合并会意外覆盖嵌套对象 |
| 托管优先级 | 最高 | 可被覆盖 | 企业管控需求优先；但牺牲了开发者的调试灵活性 |
| 路径管理 | xdg-basedir | 手动判断平台 | 库自动处理跨平台差异；手动判断容易遗漏边界情况 |

## 5.6 实际项目对应

OpenCode 的配置系统位于 `packages/opencode/src/config/`：

```
packages/opencode/src/config/
├── config.ts              # 主配置逻辑（1459 行）
├── paths.ts               # 路径管理（174 行）
├── tui.ts                 # TUI 专属配置
├── tui-schema.ts          # TUI 配置 Schema
├── markdown.ts            # Markdown 渲染配置
└── migrate-tui-config.ts  # 配置迁移
```

实际项目在此基础上还支持：
- 远程配置（`.well-known/opencode`）
- 配置热重载（文件监听）
- 配置版本迁移
- `{env:VAR}` 和 `{file:path}` 变量替换
- 插件、Agent、MCP 等高级配置

## 5.7 本章文件结构

完成本章后，你的项目结构应该是：

```
packages/opencode/
├── src/
│   ├── index.ts              # 第 4 章的 CLI 入口
│   ├── global/
│   │   └── index.ts          # 全局路径管理
│   └── config/
│       ├── config.ts         # 配置加载逻辑
│       ├── paths.ts          # 配置路径管理
│       └── test-load.ts      # 测试脚本
├── opencode.json             # 项目配置文件
├── package.json
└── tsconfig.json
```

---

## 本章小结

我们从"用户在多个项目间切换"这个真实场景出发，逐步推导出多层级配置系统的设计：

1. **单文件配置** → JSONC 支持注释，Zod 提供类型验证
2. **多层级合并** → 全局/项目/环境变量/托管，优先级递增
3. **跨平台路径** → xdg-basedir 自动处理操作系统差异
4. **设计权衡** → 托管配置优先级最高，牺牲调试灵活性换取企业管控能力

现在我们可以得出一个结论：配置系统不是一个简单的"读文件"操作，而是一个需要考虑多层级、跨平台、类型安全、企业管控的复杂系统。

这个配置系统将成为后续所有功能的基础设施。

---

**下一章预告**: 第 6 章 - 大模型 API 接入，我们将使用这个配置系统来管理 AI 模型的选择和 API 密钥。
