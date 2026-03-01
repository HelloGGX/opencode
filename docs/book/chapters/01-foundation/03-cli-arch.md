# 第 3 章：最小可用的 CLI

在上一章，我们理清了命令行工具的执行链路。现在我们需要在代码中具体实现它。

本章的目标是构建一个最基础的 CLI 程序：当我们在终端输入 `opencode --version` 时，程序能够正确接收该指令，并输出当前的版本号。我们将从最基础的 Node.js/Bun 进程参数解析开始，逐步推导出一个工程化的命令路由方案。

## 3.1 进程参数的传递机制

当我们在终端运行一个脚本时，用户输入的命令和参数是如何传递给执行环境的？无论是 Node.js 还是 Bun，都会将这些运行时参数收集在 `process.argv` 这个全局数组中。

为了直观地观察它的内部结构，我们在 `packages/opencode/src/index.ts` 中编写第一段代码，将其打印出来：

```typescript
// packages/opencode/src/index.ts
console.log(process.argv)

```

在终端中执行该文件，并附带一个 `--version` 参数：

```bash
bun run packages/opencode/src/index.ts --version
```

观察终端输出的结果：

```bash
[
  "/usr/local/bin/node",                                   // 索引 0: 运行时可执行文件的绝对路径
  "/Users/xxx/opencode/packages/opencode/src/index.ts",    // 索引 1: 当前执行脚本的绝对路径
  "--version"                                              // 索引 2: 用户实际传入的参数
]

```

从输出结果可以看出，`process.argv` 的前两个元素固定为底层运行时的路径和目标脚本的路径。用户真正输入的业务参数，永远从索引 `2` 开始。

明确了这一点，我们就可以通过截取数组来获取有效参数，并进行最基本的条件判断：

```typescript
// packages/opencode/src/index.ts
const args = process.argv.slice(2)

if (args.includes("--version") || args.includes("-v")) {
  console.log("1.0.0")
} else {
  console.log("Unknown command")
}

```

再次运行上述 `bun run` 命令，终端会正确输出 `1.0.0`。

不过，这种调用方式存在一个明显的工程缺陷。作为一款 CLI 工具，用户期望的调用方式是直接输入 `opencode --version`，而不是每次都手动指定运行时环境和脚本的绝对路径。我们需要将这段逻辑注册为操作系统的全局命令。

## 3.2 将脚本注册为系统命令

要将脚本转变为全局可执行命令，我们需要利用 `package.json` 中的 `bin` 字段。

首先，在 `packages/opencode/package.json` 中声明命令映射：

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

接下来，创建对应的入口文件。在 `packages/opencode` 目录下创建 `bin/opencode` 文件（不需要添加后缀）：

```bash
mkdir -p bin
touch bin/opencode

```

在其中写入以下内容：

```javascript
#!/usr/bin/env bun
import("../src/index.ts")

```

第一行的 `#!/usr/bin/env bun` 即 Shebang。它的作用是告知操作系统，在执行该文件时，应当去环境变量中查找 `bun` 作为解释器。第二行我们利用 Bun 原生支持 TypeScript 运行的特性，直接引入了源码文件。如果这里 Shebang 声明的是 `node`，程序在执行时会因为无法解析 `.ts` 文件的语法而抛出异常。

随后，我们需要让操作系统感知到这个映射关系。在 `packages/opencode` 目录下执行链接命令：

```bash
bun link

```

终端输出如下：

```bash
bun link v1.3.10
Success! Registered "opencode"

```

该命令的底层操作，是在系统环境变量包含的 `bin` 目录中，创建了一个指向我们 `./bin/opencode` 文件的符号链接（Symlink）。现在，你可以在任意目录下直接运行：

```bash
opencode --version

```

### 3.2.1 路径校验引发的执行错误

在执行 `opencode --version` 时，部分环境可能会抛出如下错误：

```bash
error: bun is not installed in $PATH

Please run the following command, or double check $PATH is right.

```

如果单独运行 `bun -v` 正常，但在符号链接调用时报错，通常是因为 Bun 的安装方式造成的路径校验失败。

许多开发者曾通过 `npm i -g bun` 安装 Bun。这种方式下载的并非由 Zig/C++ 编译的底层二进制文件，而是一个 Node.js 编写的 Wrapper（包装器）。当符号链接尝试唤起底层进程时，会严格校验真实二进制文件的绝对路径。如果包装器未能正确将真实二进制文件放置在系统的 `$PATH` 寻址路径中，就会触发此报错。

解决方法是移除 npm 安装的版本，并使用官方推荐的脚本直接安装二进制文件：

```bash
# macOS/Linux
curl -fsSL https://bun.sh/install | bash

# Windows (PowerShell)
powershell -c "irm bun.sh/install.ps1|iex"

```

重启终端后重新执行，即可看到正常的输出。

## 3.3 引入 Yargs 管理命令路由

通过手写 `if-else` 解析 `process.argv` 的方式虽然直观，但在扩展时会面临难以维护的问题。假设我们需要新增一个对话命令 `opencode chat --model claude-3`，基于数组遍历的解析逻辑会变得非常繁琐：我们需要判断索引位置、提取键值对、处理必填项校验，并且还要手动编写 `--help` 的打印逻辑。

为了解决命令路由和参数清洗的问题，我们可以引入成熟的解析库。这里我们选择 `yargs`。

```bash
bun add yargs

```

回到 `index.ts`，我们将前文手动解析数组的代码移除，使用 `yargs` 重构：

```typescript
// packages/opencode/src/index.ts
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

// 1. 拦截并清洗参数
const cli = yargs(hideBin(process.argv))

cli
  // 2. 注册基础命令
  .version("1.0.0")
  .help()
  // 3. 定义子命令
  .command(
    "chat",
    "Start a chat session",
    (yargs) => {
      // 定义 chat 命令所需的选项
      return yargs.option("model", {
        type: "string",
        description: "AI model to use",
        default: "gpt-4",
      })
    },
    (argv) => {
      // 命令匹配时的执行回调
      console.log(`Starting chat with model: ${argv.model}`)
    },
  )
  // 4. 兜底策略：未输入具体命令时进行提示
  .demandCommand(1, "You need at least one command before moving on")
  .parse() // 触发解析逻辑

```

观察上述代码的几个核心调整：

* `hideBin(process.argv)`：该工具函数的底层逻辑即 `process.argv.slice(2)`，它负责剥离运行时路径，将纯净的用户参数传递给 yargs 实例。
* `command()` 方法：它将命令分为描述、参数定义（builder）和逻辑执行（handler）三个部分，使得参数解析与业务逻辑彻底解耦。

在终端中输入不带参数的命令，触发默认行为：

```bash
opencode

```

yargs 会自动拦截并生成格式化的帮助文档：

```bash
Commands:
  opencode chat  Start a chat session

Options:
  --version  Show version number                                       [boolean]
  --help     Show help                                                 [boolean]

You need at least one command before moving on

```

输入带参数的子命令进行验证：

```bash
opencode chat --model claude-3.5-sonnet
# 输出：Starting chat with model: claude-3.5-sonnet

```

功能运行正常。但如果此时查看编辑器，会发现 `process` 和引入的 `yargs` 模块存在 TypeScript 缺失类型的报错。

## 3.4 补充类型与路径映射

编辑器报错提示 `process` 未定义，以及 `yargs` 隐式具有 `any` 类型。这是由于工程中尚未安装对应的 `.d.ts` 类型声明文件。

由于 `process` 属于底层的 Node.js 环境 API，其类型声明应当在 Monorepo 根目录进行版本锁定，以防止不同子包引入不一致的版本引发类型冲突。我们在根目录的 `package.json` 的 `catalog` 字段中统一声明版本：

```json
// 根目录 package.json
{
  "workspaces": {
    "packages": ["packages/*"],
    "catalog": {
      "@types/bun": "1.3.5",
      "@types/node": "25.3.3"
    }
  },
  "overrides": {
    "@types/bun": "catalog:",
    "@types/node": "catalog:"
  }
}

```

随后在项目根目录运行 `bun install` 更新依赖树。

对于仅在当前 CLI 模块使用的 `yargs`，我们直接在 `packages/opencode` 目录下安装其专属类型：

```bash
cd packages/opencode
bun add -d @types/yargs

```

安装完类型后，我们需要为 TypeScript 编译器提供一份配置文件，指导其如何解析这些类型以及处理模块的路径映射。在 `packages/opencode` 目录下新建 `tsconfig.json`：

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}

```

此配置包含两个关键设定：

1. `extends`：直接继承 Bun 官方维护的基准配置，确保 TypeScript 的编译行为与 Bun 运行时的模块解析规则严格对齐。
2. `paths`：定义 `@/*` 指向 `./src/*`。这能避免在多层级目录中出现 `../../../` 这种脆弱的相对路径引用。

保存文件后，编辑器中的类型报错会自动消除。

## 3.5 完善状态输出

在实际的工程交付中，CLI 的版本号必须与 `package.json` 中的 `version` 字段保持一致。前文中我们通过 `.version("1.0.0")` 硬编码了版本号，一旦发版极易造成信息不同步。

我们可以利用规范中支持的 JSON 模块导入功能，直接读取配置文件。同时，为了便于后续排查大模型 SDK 运行时的环境问题，我们再补充一个 `debug` 命令用于输出当前的运行状态。

更新 `index.ts` 如下：

```typescript
// packages/opencode/src/index.ts
import yargs from "yargs"
import { hideBin } from "yargs/helpers"
// 引入 package.json
import packageJson from "../../package.json" with { type: "json" }

const cli = yargs(hideBin(process.argv))

cli
  // 动态读取并设置版本号
  .version(packageJson.version)
  .help()
  .command(
    "chat",
    "Start a chat session",
    // ... 前文的 chat 代码保持不变
  )
  // 新增 debug 命令
  .command(
    "debug",
    "Print environment info for debugging",
    () => {},
    () => {
      console.log("--- Debug Info ---")
      console.log(`Version  : ${packageJson.version}`)
      console.log(`Platform : ${process.platform}`)
      console.log(`Node/Bun : ${process.version}`)
      console.log(`CWD      : ${process.cwd()}`)
    }
  )
  .demandCommand(1, "You need at least one command before moving on")
  .parse()

```

此时运行 `opencode debug`，你将看到当前进程真实的运行上下文输出。至此，一个具备扩展性、类型安全的最小可用 CLI 骨架已经搭建完毕。在下一章，我们将在这套骨架内正式接入 AI SDK。