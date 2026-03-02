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

在第 6 章调用 AI 之前，我们需要先解决一个基础问题：**如何管理配置？**

看起来很简单——读一个 JSON 文件不就行了？但实际项目中的配置需求远比这复杂：

- 用户希望有全局默认配置 (`~/.config/opencode/opencode.json`)
- 项目希望有项目级配置 (`./opencode.json`)
- 企业希望有托管配置 (`/etc/opencode/opencode.json`)
- 开发者希望用环境变量临时覆盖 (`OPENCODE_CONFIG`)
- 配置需要支持注释（JSONC 格式）
- 配置需要类型验证（Zod schema）

这些需求构成了一个完整的配置系统，值得用一章来讲解。

---

## 5.1 迭代过程

```
┌──────────────────────────────────────────────────────────────┐
│ 迭代 5.1.1: 单文件配置读取                                    │
│ ──────────────────────────────────────────────────────────── │
│ 新增: src/config/config.ts (50行)                             │
│ 功能: 读取 opencode.json                                      │
│                                                               │
│ const config = JSON.parse(                                    │
│   await fs.readFile("opencode.json", "utf-8")                 │
│ )                                                             │
│                                                               │
│ 遇到问题:                                                     │
│ - 用户写错配置导致程序崩溃                                    │
│ - 不支持注释，用户体验差                                      │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ 迭代 5.1.2: 类型验证 + JSONC 支持                             │
│ ──────────────────────────────────────────────────────────── │
│ 新增: Zod schema + jsonc-parser                               │
│ 功能: 验证配置格式，支持注释                                  │
│                                                               │
│ const ConfigSchema = z.object({                               │
│   model: z.string(),                                          │
│   provider: z.enum(["openai", "anthropic"])                   │
│ })                                                            │
│                                                               │
│ const raw = parseJsonc(content) // 支持注释                   │
│ const config = ConfigSchema.parse(raw) // 类型验证            │
│                                                               │
│ 遇到问题:                                                     │
│ - 每个项目都要写配置文件，能否有全局默认？                    │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ 迭代 5.1.3: 多层级配置合并                                    │
│ ──────────────────────────────────────────────────────────── │
│ 新增: 配置优先级系统                                          │
│ 功能: 全局配置 + 项目配置 + 环境变量                          │
│                                                               │
│ 配置加载顺序（优先级从低到高）:                               │
│ 1. 全局配置: ~/.config/opencode/opencode.json                 │
│ 2. 项目配置: ./opencode.json                                  │
│ 3. 环境变量: OPENCODE_CONFIG=/path/to/config.json             │
│                                                               │
│ 遇到问题:                                                     │
│ - 企业用户希望强制某些配置，不允许项目覆盖                    │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ 迭代 5.1.4: 托管配置支持                                      │
│ ──────────────────────────────────────────────────────────── │
│ 新增: 企业托管配置目录                                        │
│ 功能: 管理员可强制配置，覆盖所有用户配置                      │
│                                                               │
│ 最终优先级（从低到高）:                                       │
│ 1. 全局配置                                                   │
│ 2. 项目配置                                                   │
│ 3. 环境变量                                                   │
│ 4. 托管配置（企业强制，最高）                                 │
│                                                               │
│ 本章结束状态:                                                 │
│ - 支持多层级配置合并                                          │
│ - 支持 JSONC 格式（带注释）                                   │
│ - 支持 Zod 类型验证                                           │
│ - 支持企业托管配置                                            │
│ - 代码约 200 行                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 5.2 实现：单文件配置读取

最直接的做法是直接读取 JSON 文件：

```typescript
// src/config/config.ts
import fs from "fs/promises"

const config = JSON.parse(
  await fs.readFile("opencode.json", "utf-8")
)
```

这段代码有两个致命问题：用户写错一个逗号程序就崩溃，而且 JSON 不支持注释，用户体验很差。

---

## 5.3 引入类型验证与 JSONC 支持

用 Zod 做类型验证，用 `jsonc-parser` 支持注释：

```typescript
import z from "zod"
import { parse as parseJsonc } from "jsonc-parser"
import fs from "fs/promises"

const ConfigSchema = z.object({
  model: z.string().optional(),
  provider: z.enum(["openai", "anthropic"]).optional(),
  apiKey: z.string().optional(),
})

export type Config = z.infer<typeof ConfigSchema>

async function loadFile(filePath: string): Promise<Partial<Config>> {
  try {
    const content = await fs.readFile(filePath, "utf-8")
    const raw = parseJsonc(content)            // 支持注释
    return ConfigSchema.partial().parse(raw)   // 部分验证
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") {
      return {} // 文件不存在，返回空配置
    }
    throw error
  }
}
```

注意 `ConfigSchema.partial()` 的用法——每个配置文件只需要提供部分字段，最终合并后再做完整验证。

---

## 5.4 多层级配置合并

### 5.4.1 路径管理

不同操作系统的配置路径不同，需要统一管理：

```typescript
import path from "path"
import os from "os"

export namespace ConfigPaths {
  export function global(): string {
    switch (process.platform) {
      case "darwin":
      case "linux":
        return path.join(os.homedir(), ".config", "opencode")
      case "win32":
        return path.join(process.env.APPDATA || "", "opencode")
      default:
        return path.join(os.homedir(), ".opencode")
    }
  }

  export function managed(): string {
    switch (process.platform) {
      case "darwin":
        return "/Library/Application Support/opencode"
      case "linux":
        return "/etc/opencode"
      case "win32":
        return path.join(process.env.ProgramData || "C:\\ProgramData", "opencode")
      default:
        return "/etc/opencode"
    }
  }

  export function project(cwd: string): string {
    return path.join(cwd, "opencode.json")
  }
}
```

### 5.4.2 配置加载与合并

```typescript
import { mergeDeep } from "remeda"

export async function load(cwd: string = process.cwd()): Promise<Config> {
  const configs: Partial<Config>[] = []

  // 1. 全局配置（最低优先级）
  configs.push(await loadFile(path.join(ConfigPaths.global(), "opencode.json")))

  // 2. 项目配置
  configs.push(await loadFile(ConfigPaths.project(cwd)))

  // 3. 环境变量指定的配置
  if (process.env.OPENCODE_CONFIG) {
    configs.push(await loadFile(process.env.OPENCODE_CONFIG))
  }

  // 4. 托管配置（最高优先级，企业强制）
  configs.push(await loadFile(path.join(ConfigPaths.managed(), "opencode.json")))

  const merged = configs.reduce((acc, cfg) => mergeDeep(acc, cfg), {})
  return ConfigSchema.parse(merged)
}
```

配置加载流程：

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

---

## 5.5 关键决策点

| 问题 | 方案A | 方案B | 选择 | 理由 |
|------|-------|-------|------|------|
| 配置格式 | JSON | YAML | JSONC | 原生支持，用户可写注释 |
| 类型验证 | 运行时检查 | Zod | Zod | 编译时+运行时双重保障 |
| 合并策略 | 浅合并 | 深度合并 | 深度合并 | 支持嵌套配置 |
| 密钥存储 | 配置文件 | 环境变量 | 环境变量 | 安全，不进 git |

---

## 5.6 常见错误

### 错误1：配置文件写错导致程序崩溃

```typescript
// ❌ 直接 JSON.parse，没有错误处理
const config = JSON.parse(await fs.readFile("opencode.json", "utf-8"))
```

```typescript
// ✅ 使用 Zod 验证，提供友好错误信息
try {
  const raw = parseJsonc(content)
  const config = ConfigSchema.parse(raw)
} catch (error) {
  if (error instanceof z.ZodError) {
    console.error("配置文件格式错误:")
    console.error(error.format())
    process.exit(1)
  }
}
```

### 错误2：浅合并覆盖数组

```typescript
// ❌ 浅合并会覆盖整个数组
const merged = { ...global, ...project }
// global.plugins = ["a", "b"]，project.plugins = ["c"]
// 结果: merged.plugins = ["c"]  ← 丢失了 "a", "b"
```

```typescript
// ✅ 深度合并，数组拼接
function mergeConfigs(target: Config, source: Config): Config {
  const merged = mergeDeep(target, source)
  if (target.plugins && source.plugins) {
    merged.plugins = [...target.plugins, ...source.plugins]
  }
  return merged
}
```

### 错误3：API Key 写在配置文件里

```jsonc
// ❌ 危险！会被提交到 git
{
  "apiKey": "sk-1234567890abcdef"
}
```

```bash
# ✅ 使用环境变量
export OPENAI_API_KEY=sk-1234567890abcdef
```

---

## 5.7 实际项目对应

OpenCode 的配置系统位于 `packages/opencode/src/config/`：

```
packages/opencode/src/config/
├── config.ts              # 主配置逻辑
├── paths.ts               # 路径管理
├── tui.ts                 # TUI 专属配置
├── tui-schema.ts          # TUI 配置 Schema
├── markdown.ts            # Markdown 渲染配置
└── migrate-tui-config.ts  # 配置迁移
```

实际项目在此基础上还支持：
- 远程配置（`.well-known/opencode`）
- 配置热重载（文件监听）
- 配置版本迁移

---

## 本章小结

我们构建了一个完整的配置系统：

1. **单文件读取** → 引入 JSONC 支持和 Zod 验证
2. **多层级合并** → 全局/项目/环境变量/托管，优先级递增
3. **企业就绪** → 托管配置支持强制覆盖

这个配置系统将成为后续所有功能的基础设施。

---

**下一章预告**: 第 6 章 - 大模型 API 接入，我们将使用这个配置系统来管理 AI 模型的选择和 API 密钥。
