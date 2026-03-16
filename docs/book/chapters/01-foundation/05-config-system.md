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

假设我们需要在 CLI 工具中接入 AI 大模型。最容易想到的办法是，直接在代码中硬编码默认的模型名称和提供商。但随着使用场景的扩展，有些项目需要使用 OpenAI，而有些项目为了处理更长的上下文，必须切换到 Anthropic。这意味着，我们需要一种机制，允许用户在外部定义并覆盖工具的内部默认行为。

本章我们将从零开始，一步步推导并设计一个支持多层级、具备类型安全校验的配置系统。

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

接下来，我们创建一个配置加载器，负责读取这个文件并解析为对象：

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

为什么会这样？JSON 规范不允许注释。JSON 的设计初衷是作为一种轻量级的数据交换格式，为了保证不同解析器之间的一致性，其作者 Douglas Crockford 在制定规范时舍弃了注释功能。

然而，在实际工程中，JSON 经常被用作配置文件。此时，代码的可读性与后期维护成本成了主要矛盾，注释的存在变得至关重要。

为了解决这个问题，我们引入 JSONC（JSON with Comments）。JSONC 是在 JSON 语法基础上允许使用 `//` 和 `/* */` 注释的扩展格式。VS Code 的 `settings.json` 底层就采用 JSONC 解析。

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

如果在 TypeScript 中，我们通常会定义一个接口来约束结构：

```typescript
interface Config {
  model?: string
  provider?: "openai" | "anthropic"
}
```

但这里存在一个问题：TypeScript 的类型检查仅在编译时有效，在运行时这些类型信息会被完全擦除。换句话说，`parseJsonc` 返回的对象即便完全不符合 `Config` 接口，程序在加载配置这一步也不会报错。

这种缺陷会导致错误被延后。程序可能要在执行到发起网络请求的那一刻，才因为缺少 `provider` 字段而崩溃，此时抛出的错误栈往往极其深且难以溯源。

### 5.2.2 基于 Zod 的运行时校验

既然如此，我们就需要一种新的方案：能够在运行时对外部输入进行严格的结构化校验。这正是 Zod 等 Schema 校验库的核心设计目标。

首先安装 Zod 依赖：

```bash
cd packages/opencode
bun add zod
```

接下来，我们在 `packages/opencode/src/config/` 目录下定义配置的 Schema：

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import z from "zod"
import { parse as parseJsonc } from "jsonc-parser"

const ModelId = z.string()

const BaseConfigSchema = z.object({
  model: ModelId.optional().describe("AI model to use, format: model"),
  provider: z.enum(["openai", "anthropic"]).optional().describe("AI provider"),
  apiKey: z.string().optional().describe("API key for the provider"),
})

type Config = z.infer<typeof BaseConfigSchema>

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
bun run src/config/config.ts
```

```text
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

你看，错误信息直接指出了 `providre` 字段不符合预期。这种"尽早失败"的机制极大提升了工具的健壮性。

接下来，我们继续完善配置 Schema。假设我们希望约束模型名的格式，要求在模型名前加上提供商的前缀，比如 `openai/gpt-4o-mini`。我们只需要修改 `ModelId` 定义，添加一个正则表达式校验：

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import z from "zod"

const ModelId = z.string().regex(/^[^\/]+\/[^\/]+$/, "Invalid model ID format")
```

执行代码：

```bash
bun run src/config/config.ts
```

会有如下报错：

```text
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

于是我们将 `opencode.json` 的内容修改为：

```json
{
  "model": "openai/gpt-4o-mini",
  "provider": "openai"
}
```

没有报错了，说明配置格式符合要求。

明确了配置文件的格式，接下来我们需要解决另一个问题：这些配置文件应该存放在什么位置？

## 5.3 多层级配置合并策略

### 5.3.1 配置层级的推演

当 CLI 工具在复杂的开发环境中铺开时，单一配置文件会存在一定的局限性。试想如下场景：

一名开发者参与了公司内部的 10 个项目，他希望默认使用 `gpt-4o`，但唯独项目 A 需要使用 `claude-3-5-sonnet`。如果只有项目级配置，他必须在 9 个项目里重复配置 `gpt-4o`。

为了解决这个问题，我们需要引入多层级配置。在实际的 OpenCode 项目中，配置的来源分为以下几级（优先级从低到高）：

1. **远程配置**：存放在组织的 `.well-known/opencode`，用于定义组织的默认策略。
2. **全局配置**：存放在用户配置目录下（如 `~/.config/opencode/opencode.json`），代表用户的默认偏好。
3. **自定义配置**：通过 `OPENCODE_CONFIG` 环境变量指定，自定义配置文件的路径。
4. **项目配置**：存放在当前工作目录下（如 `./opencode.json`），针对特定项目的重写。
5. **.opencode 目录配置**：存放在 `.opencode/` 目录下，包含 agents、commands、plugins 等子目录的配置。
6. **内联配置**：通过 `OPENCODE_CONFIG_CONTENT` 环境变量直接传入 JSON 字符串。
7. **托管配置**：存放在系统的全局共享目录（如 `/etc/opencode/opencode.json`），通常用于企业的强制管控策略，优先级最高。

> **本章实现**：为简化表述，本章我们先实现其中最核心的 4 个层级：全局配置、项目配置、环境变量指定配置、托管配置。远程配置、.opencode 目录配置、内联配置将在后续章节中逐步引入。

那么问题来了：为什么托管配置的优先级最高？

试想这样一个场景：公司采购了私有部署的大模型，要求所有内部项目必须使用内网地址。如果托管配置优先级不够高，员工可以在项目配置中覆盖它，导致数据泄露到公网。从本质上讲，这是一种企业管理的工程权衡——托管配置能够无视用户的全局或项目级设置，强制重写请求目标，从而保障数据安全。

### 5.3.2 跨平台路径管理

开发跨平台 CLI 工具绕不开的一个隐性成本，在于需要处理不同操作系统的路径规范：

- **Linux**：遵循 XDG Base Directory 规范，用户级配置通常放在 `~/.config` 目录下，系统级放在 `/etc` 目录。
- **Windows**：有一套专属的环境变量，用户级数据存放在 `APPDATA`，系统级数据存放在 `ProgramData`。
- **macOS**：底层是 Unix，但苹果有自己的应用沙盒惯例，系统级配置通常存放在 `/Library/Application Support`。

为了保证 CLI 工具的跨平台兼容性，我们需要在代码中抹平这些操作系统的底层差异。一种标准的做法是借助 `xdg-basedir` 库，它会自动处理不同平台的路径规范。

首先安装依赖：

```bash
cd packages/opencode
bun add xdg-basedir
```

这个库封装了不同操作系统的路径规范，为开发者提供统一的 API：

| 导出变量 | 用途 | Linux 默认值 | Windows 默认值 | macOS 默认值 |
|---------|------|-------------|----------------|--------------|
| xdgData | 应用数据目录 | ~/.local/share | %APPDATA% | ~/Library/Application Support |
| xdgConfig | 配置文件目录 | ~/.config | %APPDATA% | ~/Library/Application Support |
| xdgCache | 缓存目录 | ~/.cache | %LOCALAPPDATA% | ~/Library/Caches |
| xdgState | 状态目录 | ~/.local/state | %LOCALAPPDATA% | ~/Library/Application Support |

> 注意：xdg-basedir 官方文档明确指出 "This package is meant for Linux"。在 Windows 和 macOS 下，它会尝试读取对应的 XDG 环境变量，如果未设置则回退到平台默认值。

接下来，我们创建全局路径管理模块：

```typescript
// packages/opencode/src/global/index.ts
import fs from "fs/promises"
import { xdgData, xdgCache, xdgConfig, xdgState } from "xdg-basedir"
import path from "path"
import os from "os"

const app = "opencode"

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
    config,
    state,
  }
}

await Promise.all([
  fs.mkdir(Global.Path.data, { recursive: true }),
  fs.mkdir(Global.Path.config, { recursive: true }),
  fs.mkdir(Global.Path.state, { recursive: true }),
  fs.mkdir(Global.Path.log, { recursive: true }),
  fs.mkdir(Global.Path.bin, { recursive: true }),
])
```

这里我们使用 `Promise.all` 并行创建所有必需的目录。`recursive: true` 选项意味着如果父目录不存在，会自动创建整个目录链。这种在模块初始化时确保目录存在的做法，可以避免在后续使用时频繁检查目录是否存在，简化了业务逻辑。

我们可以通过一个简单的验证来确认路径是否正确：

```bash
# 在 Linux 上运行
node -e "const {xdgData, xdgConfig} = require('xdg-basedir'); console.log('data:', xdgData); console.log('config:', xdgConfig);"
```

```text
data: /home/username/.local/share
config: /home/username/.config
```

```bash
# 在 Windows 上运行
node -e "const {xdgData, xdgConfig} = require('xdg-basedir'); console.log('data:', xdgData); console.log('config:', xdgConfig);"
```

```text
data: C:\Users\Administrator\.local\share
config: C:\Users\Administrator\.config
```
可以看到，库自动处理了操作系统差异，我们只需要在此基础上拼接应用子目录即可。

接下来，我们创建配置路径管理模块，专门处理配置文件的路径解析，我们引入 `Global`模块中的`Global.Path` 来获取跨平台的全局配置路径:

```typescript
// packages/opencode/src/config/paths.ts
import path from "path"
import { parse as parseJsonc } from "jsonc-parser"
import { Global } from "../global"

export namespace ConfigPaths {
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

  export function globalConfigFile(): string {
    return path.join(Global.Path.config, "opencode.json")
  }

  export function projectConfigFile(cwd: string): string {
    return path.join(cwd, "opencode.json")
  }

  export function parseText(text: string, filepath: string) {
    const data = parseJsonc(text)
    return data
  }
}
```

有了跨平台的路径管理模块，接下来我们就可以实现配置加载的核心逻辑了。但在此之前，我们需要解决一个关键问题：多层级的配置应该如何合并？

### 5.3.3 深度合并的必要性

要实现多层级配置的合并，最容易想到的办法是使用 JavaScript 的对象展开运算符进行浅合并，比如我们要把全局配置和项目配置合并起来：

```typescript
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

为了解决这个问题，我们必须使用深度合并算法，递归地遍历对象的每一个属性并进行合并。这里我们可以借助工具库 `remeda` 提供的 `mergeDeep` 函数：

```bash
cd packages/opencode
bun add remeda
```

接下来，我们在5.2节实现的基础上，继续完善配置加载逻辑，我们引入上一节实现的路径管理模块：`ConfigPaths`。

```typescript
// packages/opencode/src/config/config.ts
import fs from "fs/promises"
import path from "path"
import z from "zod"
import { mergeDeep } from "remeda"
import { ConfigPaths } from "./paths"

const ModelId = z.string().regex(/^[^\/]+\/[^\/]+$/, "Invalid model ID format")

const ConfigSchema = z.object({
  model: ModelId.optional(),
  provider: z.enum(["openai", "anthropic"]).optional(),
  apiKey: z.string().optional(),
})

type Config = z.infer<typeof ConfigSchema>

async function loadFile(filePath: string): Promise<Partial<Config>> {
  try {
    const content = await fs.readFile(filePath, "utf-8")
    const raw = ConfigPaths.parseText(content, filePath)
    return ConfigSchema.partial().parse(raw)
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") {
      return {}
    }
    throw error
  }
}

export async function load(cwd: string = process.cwd()): Promise<Config> {
  let result: Config = {}

  result = mergeDeep(result, await loadFile(ConfigPaths.globalConfigFile()))

  result = mergeDeep(result, await loadFile(ConfigPaths.projectConfigFile(cwd)))

  if (process.env.OPENCODE_CONFIG) {
    result = mergeDeep(result, await loadFile(process.env.OPENCODE_CONFIG))
  }

  result = mergeDeep(result, await loadFile(path.join(ConfigPaths.managedDir(), "opencode.json")))

  return ConfigSchema.parse(result)
}
```

`load函数`实现了优先级从低到高逐个加载全局配置、项目配置、环境变量指定配置、托管配置深度合并。配置加载完成后，通过 `ConfigSchema.parse()` 进行最终校验。

### 5.3.4 验证多层级合并

接下来，我们来验证优先级是否正确。

首先创建全局配置：

```bash
# 创建全局配置目录
mkdir -p ~/.config/opencode

# 写入全局配置
cat > ~/.config/opencode/opencode.json << 'EOF'
{
  "provider": "openai",
  "model": "openai/gpt-4o-mini"
}
EOF
```

然后创建项目配置：

```bash
# 在 packages/opencode 目录下
cat > ./opencode.json << 'EOF'
{
  "model": "claude/claude-3-5-sonnet"
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
{ provider: "openai", model: "claude/claude-3-5-sonnet" }
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
| 合并策略 | 深度合并 | 浅合并 | 深度合并保留嵌套结构；浅合并会丢失数据 |
| 路径管理 | xdg-basedir | 手写跨平台逻辑 | 库封装了平台差异；手写逻辑维护成本高 |

**下一章预告**: 第 6 章 - 大模型 API 接入，我们将使用这个配置系统来管理 AI 模型的选择和 API 密钥。
