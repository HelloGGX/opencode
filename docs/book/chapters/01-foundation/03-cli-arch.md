# 第 3 章：最小可用的 CLI

在上一章，我们理解了 `opencode --version` 背后的完整执行链路：PATH 查找、符号链接、Shebang、启动器。现在，是时候亲手实现这个命令了。

本章的目标很朴素：让用户能在终端输入 `opencode --version`，看到版本号输出。

但即使是这样一个看似简单的需求，如果我们模拟真实开发的节奏，也会经历几次迭代。让我们从最直接的方案开始。

---

## 3.1 迭代一：单文件 CLI

### 3.1.1 最直接的方案

在第 1 章，我们已经搭建了 Monorepo 环境。现在，让我们在 `packages/opencode/src/index.ts` 中写下第一行代码。

最直接的方案是什么？解析 `process.argv`，检查是否包含 `--version`。

```typescript
// packages/opencode/src/index.ts

const args = process.argv.slice(2)

if (args.includes("--version") || args.includes("-v")) {
  console.log("1.0.0")
} else {
  console.log("Unknown command")
}
```

这就是全部代码，只有 8 行。让我们运行它：

```bash
bun run packages/opencode/src/index.ts --version
```

输出：

```
1.0.0
```

成功了。但如果你尝试运行：

```bash
bun run packages/opencode/src/index.ts --help
```

输出：

```
Unknown command
```

### 3.1.2 发现问题

这个方案有两个明显的问题：

**问题 1：每次都要敲 `bun run packages/opencode/src/index.ts`，太繁琐。**

用户期望的是 `opencode --version`，而不是一长串路径。我们需要一种方式，让 `opencode` 成为一个全局可用的命令。

**问题 2：参数解析逻辑太简陋。**

当前只支持 `--version`。如果未来要支持 `--help`、`--debug`、`chat` 子命令，手动解析 `process.argv` 会变成一场噩梦。

让我们先解决问题 1。

---

## 3.2 迭代二：全局命令

### 3.2.1 package.json 的 bin 字段

在第 2 章，我们学习了 `package.json` 的 `bin` 字段如何让普通文件变成全局命令。现在，让我们应用这个知识。

首先，确保 `packages/opencode/package.json` 中有 `bin` 字段：

```json
{
  "name": "opencode",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "opencode": "./bin/opencode"
  }
}
```

然后，创建 `packages/opencode/bin/opencode` 文件：

```bash
mkdir -p packages/opencode/bin
```

```javascript
#!/usr/bin/env node
import("../src/index.ts")
```

这里有一个细节需要注意：我们使用 `#!/usr/bin/env node` 作为 Shebang，但文件内容是 `import("../src/index.ts")`。这是因为 Bun 可以直接运行 TypeScript 文件，而 Node.js 需要 `.js` 扩展名。

### 3.2.2 验证全局命令

在项目根目录执行：

```bash
cd packages/opencode
bun link
```

输出：

```
bun link v1.3.8

Linked "opencode" to ~/.bun/bin/opencode

Run `opencode` to run the linked package.
```

现在，让我们测试：

```bash
opencode --version
```

输出：

```
1.0.0
```

成功了！现在用户可以直接输入 `opencode --version`，而不需要敲一长串路径。

### 3.2.3 发现新问题

全局命令的问题解决了。但参数解析的问题还在。

让我们尝试添加 `--help` 支持：

```typescript
// packages/opencode/src/index.ts

const args = process.argv.slice(2)

if (args.includes("--version") || args.includes("-v")) {
  console.log("1.0.0")
} else if (args.includes("--help") || args.includes("-h")) {
  console.log(`
opencode - AI-powered development tool

Usage:
  opencode [command] [options]

Commands:
  chat    Start a chat session

Options:
  --version, -v    Show version
  --help, -h       Show help
`)
} else {
  console.log("Unknown command. Use --help for usage.")
}
```

代码量增加到了 20 行。看起来还行，但如果我们继续添加：

- `--debug` 参数
- `chat` 子命令
- `config` 子命令
- 每个子命令有自己的 `--help`

手动解析会变得难以维护。我们需要一个更好的方案。

---

## 3.3 迭代三：引入 yargs

### 3.3.1 为什么选择 yargs

在 Node.js 生态中，命令行参数解析有几个主流选择：

| 库 | 周下载量 | 特点 |
|---|---------|------|
| yargs | 4000万+ | 功能全面，自动生成 help |
| commander | 3500万+ | 轻量，API 简洁 |
| minimist | 3000万+ | 极简，只做解析 |

我们选择 yargs，原因有三个：

1. **自动生成 `--help`**：不需要手动维护帮助文本
2. **子命令支持**：`opencode chat`、`opencode config` 等子命令有清晰的组织方式
3. **类型推断**：配合 TypeScript，参数有类型提示

### 3.3.2 安装 yargs

```bash
cd packages/opencode
bun add yargs
bun add -d @types/yargs
```

### 3.3.3 重构代码

让我们用 yargs 重写 `index.ts`：

```typescript
// packages/opencode/src/index.ts
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

yargs(hideBin(process.argv))
  .version("1.0.0")
  .command(
    "chat",
    "Start a chat session",
    (yargs) => {
      return yargs.option("model", {
        type: "string",
        description: "AI model to use",
        default: "gpt-4",
      })
    },
    (argv) => {
      console.log(`Starting chat with model: ${argv.model}`)
      console.log("TODO: implement chat")
    }
  )
  .demandCommand(1, "You need at least one command before moving on")
  .help()
  .parse()
```

代码量是 25 行，但功能比手动解析强得多。让我们验证：

```bash
opencode --version
```

输出：

```
1.0.0
```

```bash
opencode --help
```

输出：

```
opencode

Commands:
  opencode chat  Start a chat session

Options:
  --version  Show version number                                       [boolean]
  --help     Show help                                                 [boolean]
```

```bash
opencode chat --help
```

输出：

```
opencode chat

Start a chat session

Options:
  --model  AI model to use                    [string] [default: "gpt-4"]
  --help   Show help                                           [boolean]
```

```bash
opencode chat --model claude-3
```

输出：

```
Starting chat with model: claude-3
TODO: implement chat
```

### 3.3.4 关键代码解析

让我们逐行理解这段代码：

```typescript
yargs(hideBin(process.argv))
```

`hideBin(process.argv)` 是一个工具函数，它返回 `process.argv.slice(2)`，即去掉 `node` 和脚本路径，只保留用户输入的参数。

```typescript
.version("1.0.0")
```

自动添加 `--version` 和 `-v` 支持，版本号从 `package.json` 读取或直接指定。

```typescript
.command(
  "chat",
  "Start a chat session",
  (yargs) => { ... },  // builder: 定义子命令的参数
  (argv) => { ... }    // handler: 子命令的执行逻辑
)
```

定义一个子命令。`builder` 函数用于定义参数，`handler` 函数用于执行逻辑。

```typescript
.demandCommand(1, "You need at least one command before moving on")
```

要求用户必须提供至少一个命令。如果用户只输入 `opencode`，会显示错误提示。

```typescript
.help()
```

自动添加 `--help` 和 `-h` 支持。

```typescript
.parse()
```

解析参数并执行对应的 handler。

---

## 3.4 本章小结

### 3.4.1 迭代过程回顾

我们经历了三次迭代：

| 迭代 | 问题 | 解决方案 | 代码量 |
|-----|------|---------|--------|
| 3.1 | 如何解析 `--version`？ | 手动解析 `process.argv` | 8 行 |
| 3.2 | 如何变成全局命令？ | `package.json` bin 字段 + `bun link` | 20 行 |
| 3.3 | 如何支持更多参数？ | 引入 yargs | 25 行 |

每次迭代都是因为遇到了具体的问题，而不是一开始就追求"完美架构"。这是真实开发的节奏。

### 3.4.2 最终代码

```typescript
// packages/opencode/src/index.ts
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

yargs(hideBin(process.argv))
  .version("1.0.0")
  .command(
    "chat",
    "Start a chat session",
    (yargs) => {
      return yargs.option("model", {
        type: "string",
        description: "AI model to use",
        default: "gpt-4",
      })
    },
    (argv) => {
      console.log(`Starting chat with model: ${argv.model}`)
      console.log("TODO: implement chat")
    }
  )
  .demandCommand(1, "You need at least one command before moving on")
  .help()
  .parse()
```

### 3.4.3 验证清单

在进入下一章之前，请确保以下命令都能正常工作：

```bash
opencode --version
# 期望输出: 1.0.0

opencode --help
# 期望输出: 帮助信息，包含 chat 命令

opencode chat --help
# 期望输出: chat 命令的帮助信息，包含 --model 参数

opencode chat --model claude-3
# 期望输出: Starting chat with model: claude-3
#          TODO: implement chat
```

---

## 3.5 读者练习

### 练习 1：添加 debug 命令

添加一个 `opencode debug` 命令，输出以下信息：

- Node.js 版本：`process.version`
- 平台：`process.platform`
- 架构：`process.arch`
- 当前工作目录：`process.cwd()`

**提示**：参考 `chat` 命令的定义方式。

### 练习 2：版本号从 package.json 读取

当前版本号是硬编码的 `"1.0.0"`。尝试从 `package.json` 动态读取版本号。

**提示**：可以使用 `import { readFileSync } from "fs"` 读取 JSON 文件，或者使用 Bun 的 `Bun.file()` API。

### 练习 3：添加 config 命令

添加一个 `opencode config` 命令，支持以下子命令：

- `opencode config list`：列出所有配置
- `opencode config get <key>`：获取指定配置
- `opencode config set <key> <value>`：设置配置

**提示**：yargs 支持嵌套子命令，可以查阅文档了解 `command` 的更多用法。

---

## 3.6 延伸阅读

- [yargs 官方文档](https://yargs.js.org/)
- [Node.js process.argv 详解](https://nodejs.org/docs/latest/api/process.html#process_process_argv)
- [npm package.json bin 字段规范](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#bin)
