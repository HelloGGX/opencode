# 第二章：环境搭建与CLI实操

在第一章中，我们深入探讨了OpenCode的全景架构与设计哲学，理解了其模块化设计、Agent协作机制以及多平台适配策略。从本章开始，我们将从理论走向实践，通过亲手搭建开发环境来深化对这套架构的理解。本章将以OpenCode官方仓库为蓝本，带领读者完成从零到一的环境搭建工作，目标是让每位读者都能在自己的机器上成功运行`opencode --version`命令，并理解这背后的技术实现细节。

## 2.1 项目初始化：从空目录开始

### 2.1.1 创建项目根目录

一切伟大的工程都始于一个空目录。让我们从零开始，逐步构建与OpenCode一致的Monorepo项目。首先，我们需要在文件系统中创建一个新的项目目录，这个目录将成为我们整个项目的根节点。建议将项目放在用户主目录下的workspace文件夹中，这样既能保持整洁，又能与其他项目保持良好的隔离性。

打开终端，执行以下命令来创建项目结构：

```bash
mkdir -p ~/workspace/myopencode
cd ~/workspace/myopencode
pwd
```

执行完这些命令后，你应该看到终端显示`/Users/你的用户名/workspace/myopencode`（macOS）或类似路径。这个目录将成为我们所有后续操作的基准点。在整个学习过程中，我们将始终保持在这个目录下执行命令，这样能够避免因路径问题导致的错误。

接下来，我们需要初始化Git版本控制系统。Git是现代软件开发的基础工具，它不仅能够追踪代码变更，还支持多人协作开发。OpenCode项目使用了Git作为版本控制系统，我们也将遵循这一选择：

```bash
git init
git config user.name "你的名字"
git config user.email "你的邮箱@example.com"
```

这两条git config命令分别设置了用户名和邮箱，这些信息会嵌入到每次提交中，用于标识代码的作者身份。完成这些设置后，我们的项目就已经具备了基本的版本控制能力。

为了保持良好的代码规范，我们还需要创建一个.gitignore文件。这个文件告诉Git哪些文件或目录应该被忽略，不需要纳入版本控制。在Node.js/Bun项目中，通常需要忽略node_modules目录、构建产物、编辑器配置等：

```bash
cat > .gitignore << 'EOF'
node_modules/
dist/
*.log
.DS_Store
.idea/
.vscode/
EOF
```

创建完.gitignore文件后，让我们进行一次初始提交，将当前的空项目状态记录到版本历史中：

```bash
git add .
git commit -m "初始化项目结构"
```

执行完这些命令后，一个干净的项目骨架就已经建立起来了。这个骨架虽然简单，但它已经具备了与OpenCode项目相同的版本控制基础。

### 2.1.2 理解目录结构规划

在正式开始编写代码之前，我们需要理解OpenCode的目录结构规划，并在自己的项目中采用类似的结构。OpenCode的目录结构经过精心设计，每个目录都有明确的职责划分，这不仅有助于代码组织，也方便团队协作和维护。

OpenCode的根目录包含以下核心区域：`.github`目录存放GitHub相关的配置文件，包括Issue模板、Pull Request模板和CI/CD工作流配置；`.husky`目录存放Git hooks配置，用于在提交代码前执行代码检查等自动化任务；`.opencode`目录包含OpenCode编辑器的特定配置；`_bmad`目录存放BMad Agent框架的配置和数据；`docs`目录存放项目文档；`github`目录包含GitHub应用的源代码；`infra`目录存放基础设施代码；`nix`目录存放Nix包管理器配置；`packages`目录是最核心的区域，存放所有业务功能模块。

对于我们的项目，我们可以采用类似的结构，但可以根据实际需求进行适当简化。首要任务是创建packages目录，因为这是Monorepo的核心：

```bash
mkdir -p packages/cli
mkdir -p packages/sdk
mkdir -p packages/app
```

这三个子目录分别对应CLI核心模块、SDK模块和Web应用模块。在后续的学习中，我们将逐步填充这些目录的内容。在此之前，让我们先完成Bun的配置工作。

## 2.2 Bun环境配置详解

### 2.2.1 安装并验证Bun运行环境

Bun是由Jarred Sumner用Zig语言编写的JavaScript运行时，它相较于Node.js和Deno具有显著的性能优势。OpenCode项目选择Bun作为包管理器和运行时，主要原因是其卓越的启动速度和包安装性能。在大型Monorepo项目中，依赖安装往往是开发流程中的瓶颈环节，而Bun通过原生实现npm注册表协议、利用机器码编译以及优化文件IO操作，能够将安装时间大幅缩短。

首先，我们需要检查系统中是否已经安装了Bun。打开终端，执行以下命令：

```bash
bun --version
```

如果系统已经安装了Bun，你会看到类似`1.3.5`的版本号输出。如果看到"command not found"或类似错误，说明系统中还没有安装Bun，这时需要执行安装程序。

在macOS和Linux系统上，Bun的安装非常简单。执行以下命令即可完成安装：

```bash
curl -fsSL https://bun.sh/install | bash
```

这条命令会下载Bun的安装脚本并自动执行。安装脚本会自动将bun可执行文件添加到系统的PATH环境变量中，通常是`~/.bun/bin/bun`。安装完成后，你需要重新加载shell配置或打开新的终端窗口以使PATH变更生效：

```bash
source ~/.zshrc  # 如果你使用zsh
# 或者
source ~/.bashrc  # 如果你使用bash
```

现在再次执行`bun --version`，你应该能看到Bun的版本号了。请注意，OpenCode项目指定使用`bun@1.3.5`版本，如果你的Bun版本与此不同，可能会遇到兼容性问题。在这种情况下，可以使用Bun的版本管理器安装指定版本：

```bash
bun install -g bun@1.3.5
```

对于Windows用户，Bun提供了专门的安装程序。可以通过PowerShell执行以下命令：

```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

或者从Bun的GitHub Releases页面下载预编译的Windows版本。值得注意的是，Windows版本的Bun在某些Unix特定的系统调用上可能存在差异，但OpenCode项目通过条件编译和特性检测机制确保了跨平台兼容性。

验证Bun安装成功后，我们还需要确认几个关键工具是否可用。Bun自带了包管理器、运行时和打包工具，我们可以通过以下命令验证这些功能：

```bash
bun --help
```

这个命令会显示Bun的帮助信息，列出所有可用的子命令。常见的子命令包括`bun install`（安装依赖）、`bun run`（运行脚本）、`bun test`（运行测试）和`bun build`（打包代码）等。花些时间熟悉这些命令的功能和用法，会大大提高后续的开发效率。

### 2.2.2 配置Bun包管理器设置

Bun的配置主要通过bunfig.toml文件完成。这个文件位于项目根目录，用于定义Bun的各种行为选项。OpenCode项目的bunfig.toml配置展示了生产级别的设置，让我们创建一个类似的配置文件。

在项目根目录创建bunfig.toml文件：

```bash
cat > bunfig.toml << 'EOF'
[install]
exact = true

[test]
root = "./do-not-run-tests-from-root"
EOF

cat bunfig.toml
```

`[install]`部分的`exact = true`设置是OpenCode项目的一个重要配置。这个选项确保所有依赖都精确锁定到指定版本，不会自动升级到兼容版本中的最新版本。这种严格模式在生产环境中非常重要，它可以避免因依赖版本漂移导致的构建失败或行为变化。

`[test]`部分的配置指定了测试根目录。OpenCode项目将测试根目录设置为一个不存在的路径，这是一个技巧性的做法，表示"不要在根目录运行测试"。这是因为Monorepo项目通常需要在各子包中分别运行测试，而不是在根目录统一运行。

Bun支持多种配置选项，涵盖了安装行为、打包选项、测试设置等各个方面。了解这些配置选项对于优化开发流程非常重要。以下是一些常用的配置选项：

```toml
[install]
cache = true                          # 启用依赖缓存
clean = false                         # 不自动清理缓存
dry-run = false                       # 不进行试运行
global = false                        # 不安装到全局目录
locked = true                         # 使用锁文件

[pack]
prepend = "#!/usr/bin/env bun"        # 在打包文件中添加shebang
```

这些配置选项可以根据项目的具体需求进行调整。在后续的开发过程中，我们可以根据实际情况修改bunfig.toml文件。

## 2.3 Monorepo架构配置

### 2.3.1 创建根包配置文件

Monorepo的核心是根目录的package.json文件。这个文件不仅定义了项目的元数据，还通过workspaces配置声明了所有子包的位置。OpenCode的package.json配置展示了企业级Monorepo的最佳实践，让我们来创建一个类似的配置。

在项目根目录创建package.json文件：

```bash
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "myopencode",
  "description": "AI-powered development tool - Monorepo tutorial",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.5",
  "scripts": {
    "dev": "echo '开发模式启动...'",
    "typecheck": "echo '类型检查...'",
    "build": "echo '构建项目...'",
    "test": "echo '运行测试...'",
    "prepare": "husky"
  },
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "catalog": {
      "@types/bun": "1.3.5",
      "typescript": "5.8.2",
      "zod": "3.22.0"
    }
  },
  "devDependencies": {
    "@tsconfig/bun": "1.0.9",
    "husky": "9.1.7",
    "prettier": "3.6.2"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/myopencode"
  },
  "license": "MIT",
  "prettier": {
    "semi": false,
    "printWidth": 120
  }
}
EOF

cat package.json
```

让我们详细分析这个配置文件的各个部分：

`"$schema"`字段指向JSON Schema文件，IDE可以据此提供自动补全和验证功能。虽然这是可选的，但它能够大大提升开发体验，建议始终保留。

`"name"`字段定义了项目的名称。这个名称会用于npm包的发布（如果是公开包）和工作区引用。在Monorepo中，通常使用组织名称作为前缀，如`@myorg/myapp`。

`"private": true`设置非常重要。对于内部项目或不想发布到npm的包，必须设置这个选项为true。如果忘记设置这个选项，npm publish会拒绝发布私有包。

`"type": "module"`声明这个包使用ES Modules语法。这是Bun原生支持的模式，与现代JavaScript生态接轨。如果不使用这个选项，Bun会默认使用CommonJS语法。

`"packageManager"`字段指定了项目使用的包管理器及其版本。`"bun@1.3.5"`表示项目要求使用Bun 1.3.5版本。这个字段不仅是对人类的提示，Bun和npm也会读取这个字段以确保使用正确的包管理器版本。

`"workspaces"`是Monorepo配置的核心。它包含两部分：`packages`定义了子包的位置模式，`catalog`定义了共享依赖的版本。

`workspaces.packages`使用glob模式匹配子包位置。`"packages/*"`表示packages目录下的所有直接子目录都是独立的工作区。这意味着packages/cli、packages/sdk、packages/app都会被识别为独立的包，可以相互引用。

`workspaces.catalog`是Bun提供的一个强大特性，称为"目录版本控制"。它允许在一个中心位置定义所有依赖的版本，然后在各个子包中引用这些版本。当需要升级某个依赖时，只需修改catalog中的版本，所有引用它的包都会自动使用新版本。

### 2.3.2 创建子包结构

现在我们已经定义了Monorepo的结构，接下来需要创建各个子包的配置文件。让我们首先创建CLI包，这是我们实现`opencode --version`命令的核心。

进入packages/cli目录并创建包配置文件：

```bash
cd packages/cli
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "@myopencode/cli",
  "version": "0.1.0",
  "type": "module",
  "license": "MIT",
  "bin": {
    "myopencode": "./bin/myopencode"
  },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "build": "echo '构建CLI...'",
    "dev": "bun run --conditions=browser ./src/index.ts",
    "clean": "echo '清理构建产物...'"
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "@types/bun": "catalog:",
    "@types/node": "catalog:",
    "typescript": "catalog:"
  },
  "dependencies": {
    "typescript": "catalog:"
  }
}
EOF
```

这个配置文件有几个值得注意的点：

`"name": "@myopencode/cli"`采用了作用域命名规范。`@myopencode/`是作用域前缀，`cli`是包名。这种命名方式可以避免与npm上的其他包发生命名冲突，同时也表明了包的所有关系。

`"bin"`字段定义了CLI入口点。`"myopencode": "./bin/myopencode"`告诉包管理器，当用户安装这个包时，需要创建一个名为`myopencode`的命令，指向bin目录下的myopencode脚本文件。

`"devDependencies"`和`"dependencies"`都使用了`"catalog:"`前缀。这意味着这些依赖的版本不是直接写在子包中，而是从根目录的catalog中读取。这种方式确保了整个Monorepo使用相同版本的依赖。

接下来，我们创建SDK包和App包的配置文件：

```bash
cd ../sdk
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "@myopencode/sdk",
  "version": "0.1.0",
  "type": "module",
  "license": "MIT",
  "main": "./src/index.ts",
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "build": "echo '构建SDK...'",
    "dev": "echo '开发SDK...'",
    "clean": "echo '清理SDK构建产物...'"
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "@types/bun": "catalog:",
    "typescript": "catalog:"
  },
  "dependencies": {
    "zod": "catalog:"
  }
}
EOF

cd ../app
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "@myopencode/app",
  "version": "0.1.0",
  "type": "module",
  "license": "MIT",
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "build": "echo '构建应用...'",
    "dev": "echo '开发应用...'",
    "clean": "echo '清理应用构建产物...'"
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "@types/bun": "catalog:",
    "typescript": "catalog:"
  },
  "dependencies": {
    "@myopencode/sdk": "workspace:*",
    "@myopencode/cli": "workspace:*"
  }
}
EOF
```

注意app包的dependencies配置。`"@myopencode/sdk": "workspace:*"`表示这个包依赖同一workspace中的SDK包。`workspace:*`是一个通配符，表示使用workspace中定义的任何版本。Bun在安装时会将这个引用解析为本地包的路径。

### 2.3.3 配置TypeScript编译环境

TypeScript是OpenCode项目的核心开发语言，合理的TypeScript配置对于保证代码质量至关重要。OpenCode采用了集中化的TypeScript配置策略：根目录定义基础配置，各子包可以根据需要覆盖或扩展。

首先，在项目根目录创建基础的tsconfig.json：

```bash
cd ../..
cat > tsconfig.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "moduleResolution": "bundler",
    "target": "ESNext",
    "module": "ESNext",
    "lib": ["ESNext", "DOM"],
    "jsx": "preserve",
    "jsxImportSource": "solid-js",
    "composite": false,
    "declaration": false,
    "declarationMap": false,
    "noEmit": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["packages/*/src/**/*", "packages/*/bin/**/*"],
  "exclude": ["node_modules", "dist"]
}
EOF
cat tsconfig.json
```

这个配置继承自`@tsconfig/bun/tsconfig.json`，后者提供了Bun环境的最佳实践。`extends`机制允许我们复用社区积累的最佳实践，同时通过`compilerOptions`进行自定义配置。

关键的编译选项包括：

`"strict": true`启用所有严格类型检查选项，这有助于在编译期发现潜在的类型错误，提高代码质量。

`"moduleResolution": "bundler"`指定模块解析策略为bundler模式，这是Bun、esbuild等现代打包工具推荐的模式。

`"noEmit": true`告诉TypeScript只进行类型检查，不输出JavaScript文件。这在开发模式下很有用，因为我们通常使用Bun或esbuild来打包代码。

`"include"`定义了需要包含在编译范围内的文件模式。这个配置确保各子包的src和bin目录都被正确包含。

## 2.4 CLI入口点实现

### 2.4.1 创建CLI包的目录结构

现在，我们开始实现CLI的核心部分。首先需要创建CLI包的完整目录结构，包括入口脚本、源代码目录和构建配置：

```bash
cd packages/cli
mkdir -p bin src
ls -la
```

我们创建了两个目录：`bin`目录用于存放入口脚本，这是用户执行`myopencode`命令时首先加载的文件；`src`目录用于存放TypeScript源代码，实现CLI的具体功能。

接下来，我们创建CLI的入口脚本。这个脚本将负责检测平台、定位二进制文件并启动实际的CLI程序：

```bash
cat > bin/myopencode << 'EOF'
#!/usr/bin/env node

const childProcess = require("child_process")
const fs = require("fs")
const path = require("path")
const os = require("os")

function run(target) {
  const result = childProcess.spawnSync(target, process.argv.slice(2), {
    stdio: "inherit",
  })
  if (result.error) {
    console.error(result.error.message)
    process.exit(1)
  }
  const code = typeof result.status === "number" ? result.status : 0
  process.exit(code)
}

const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath)
}

const scriptPath = fs.realpathSync(__filename)
const scriptDir = path.dirname(scriptPath)

const platformMap = {
  darwin: "darwin",
  linux: "linux",
  win32: "windows",
}
const archMap = {
  x64: "x64",
  arm64: "arm64",
  arm: "arm",
}

let platform = platformMap[os.platform()]
if (!platform) {
  platform = os.platform()
}
let arch = archMap[os.arch()]
if (!arch) {
  arch = os.arch()
}
const base = "myopencode-" + platform + "-" + arch
const binary = platform === "windows" ? "myopencode.exe" : "myopencode"

function findBinary(startDir) {
  let current = startDir
  for (;;) {
    const modules = path.join(current, "node_modules")
    if (fs.existsSync(modules)) {
      const entries = fs.readdirSync(modules)
      for (const entry of entries) {
        if (!entry.startsWith(base)) {
          continue
        }
        const candidate = path.join(modules, entry, "bin", binary)
        if (fs.existsSync(candidate)) {
          return candidate
        }
      }
    }
    const parent = path.dirname(current)
    if (parent === current) {
      return
    }
    current = parent
  }
}

const resolved = findBinary(scriptDir)
if (!resolved) {
  console.error(
    'It seems that your package manager failed to install the right version of the myopencode CLI for your platform. You can try manually installing the "' +
      base +
      '" package',
  )
  process.exit(1)
}

run(resolved)
EOF

chmod +x bin/myopencode
```

这个入口脚本与OpenCode的实现几乎完全一致，它的工作原理如下：

脚本开头的`#!/usr/bin/env node`是Unix系统的shebang，告诉系统使用Node.js来执行这个脚本。

`run`函数是一个简单的进程启动器。它使用`child_process.spawnSync`同步启动目标可执行文件，并将所有命令行参数传递给它。`stdio: "inherit"`设置确保子进程的输入输出与父进程共享，用户可以直接与CLI交互。

脚本首先检查`OPENCODE_BIN_PATH`环境变量。如果设置了这个变量，脚本会直接使用它指定的路径，这是为高级用户提供的一种覆盖默认行为的机制。

接下来，脚本检测当前操作系统和CPU架构。`platformMap`和`archMap`将系统信息映射到OpenCode的二进制命名规范。例如，在macOS（M1芯片）上运行的系统会被映射为`myopencode-darwin-arm64`。

`findBinary`函数负责在目录树中搜索实际的二进制文件。它采用向上遍历的策略，从当前脚本所在目录开始，不断向父目录搜索，直到找到包含正确二进制文件的node_modules目录。

### 2.4.2 创建TypeScript核心实现

现在，我们创建CLI的核心TypeScript实现。这个文件将处理命令行参数并执行相应的操作：

```bash
mkdir -p src/cli
cat > src/index.ts << 'EOF'
#!/usr/bin/env bun

import { parseArgs } from "util"

interface CliOptions {
  version: boolean
  help: boolean
}

function printVersion(): void {
  console.log("myopencode version 0.1.0")
}

function printHelp(): void {
  console.log(`
MyOpenCode - AI-powered Development Tool

Usage: myopencode [options] [command]

Options:
  -V, --version    output the version number
  -h, --help       display help for command

Commands:
  start            Start the development server
  build            Build the project for production
  test             Run tests
  help [command]   display help for a specific command

For more information, visit https://github.com/yourusername/myopencode
`)
}

function main(): void {
  const args = parseArgs({
    args: Bun.argv,
    strict: true,
    allowPositionals: true,
  })

  const options: CliOptions = {
    version: false,
    help: false,
  }

  const positionals: string[] = []

  for (const arg of args.positionals) {
    if (arg === "--version" || arg === "-V") {
      options.version = true
    } else if (arg === "--help" || arg === "-h") {
      options.help = true
    } else {
      positionals.push(arg)
    }
  }

  if (options.version) {
    printVersion()
    process.exit(0)
  }

  if (options.help) {
    printHelp()
    process.exit(0)
  }

  if (positionals.length === 0) {
    printHelp()
    process.exit(0)
  }

  const command = positionals[0]
  const commandArgs = positionals.slice(1)

  switch (command) {
    case "start":
      console.log("Starting development server...")
      break
    case "build":
      console.log("Building project...")
      break
    case "test":
      console.log("Running tests...")
      break
    default:
      console.error(`Unknown command: ${command}`)
      printHelp()
      process.exit(1)
  }
}

main()
EOF

cat src/index.ts
```

这个TypeScript实现展示了CLI的基本框架：

`parseArgs`函数来自Bun的util模块，用于解析命令行参数。Bun提供了原生的参数解析支持，这比使用第三方库如yargs更加轻量和快速。

`printVersion`和`printHelp`函数分别处理版本查询和帮助信息显示。在`myopencode --version`命令被调用时，`printVersion`函数会输出当前版本号。

`main`函数是CLI的入口点。它首先解析命令行参数，然后根据参数执行相应的操作。参数解析遵循常见的CLI约定：`-V`或`--version`输出版本，`-h`或`--help`显示帮助信息。

switch语句处理各个子命令。目前的实现只打印消息，真正的实现会在后续章节中逐步添加。

### 2.4.3 创建平台特定的构建脚本

为了让我们的CLI能够在不同平台上运行，我们需要创建一个构建脚本，将TypeScript代码编译为可执行文件：

```bash
mkdir -p script
cat > script/build.ts << 'EOF'
import { mkdir, writeFile, rm, cp, exec } from "node:fs/promises"
import { existsSync } from "node:fs"
import { join, dirname } from "node:path"
import { fileURLToPath } from "node:url"

const __dirname = dirname(fileURLToPath(import.meta.url))
const rootDir = join(__dirname, "..")
const srcDir = join(rootDir, "src")
const distDir = join(rootDir, "dist")

const platforms = [
  { os: "darwin", arch: "x64", exe: "myopencode-darwin-x64" },
  { os: "darwin", arch: "arm64", exe: "myopencode-darwin-arm64" },
  { os: "linux", arch: "x64", exe: "myopencode-linux-x64" },
  { os: "linux", arch: "arm64", exe: "myopencode-linux-arm64" },
  { os: "windows", arch: "x64", exe: "myopencode-windows-x64.exe" },
]

async function build() {
  console.log("Building MyOpenCode CLI...")

  if (existsSync(distDir)) {
    await rm(distDir, { recursive: true })
  }
  await mkdir(distDir, { recursive: true })

  for (const platform of platforms) {
    const platformDir = join(distDir, platform.exe)
    await mkdir(platformDir, { recursive: true })

    const executableContent = `#!/bin/bash
exec bun "${join(dirname(platformDir), "..", "..", "src", "index.ts")}" "$@"
`

    await writeFile(join(platformDir, "myopencode"), executableContent)
    await chmod(join(platformDir, "myopencode"), "755")

    console.log(`Built for ${platform.os}-${platform.arch}`)
  }

  console.log("Build complete!")
}

async function chmod(path: string, mode: string) {
  const { chmodSync } = await import("node:fs")
  chmodSync(path, mode)
}

build().catch(console.error)
EOF

cat script/build.ts
```

这个构建脚本展示了如何为不同平台生成可执行文件。实际的构建过程会更加复杂，需要编译TypeScript代码、捆绑依赖等，但这个示例展示了基本的架构。

## 2.5 依赖安装与功能验证

### 2.5.1 安装项目依赖

现在，我们已经完成了CLI包的基本配置，接下来需要安装所有依赖。回到项目根目录并执行安装命令：

```bash
cd ../..
bun install
```

Bun会读取根目录的package.json文件，处理workspaces配置，然后依次安装所有声明的依赖。这个过程会生成bun.lockb文件，记录所有依赖的确切版本。

安装过程中，Bun会输出进度信息，让我们可以监控安装进度。如果遇到网络问题或依赖冲突，Bun会提供详细的错误信息。

依赖安装完成后，我们可以通过以下命令验证安装结果：

```bash
ls -la node_modules
```

你应该能看到node_modules目录已经创建，其中包含了所有依赖包。由于我们使用的是Bun的catalog机制，实际安装的依赖版本会与根目录package.json中catalog定义的版本一致。

### 2.5.2 链接CLI命令

要让`myopencode`命令在系统中全局可用，我们需要将CLI包链接到全局。这可以通过Bun的link命令完成：

```bash
cd packages/cli
bun link
```

这个命令会创建一个符号链接，将`myopencode`命令指向当前包的bin/myopencode脚本。链接创建完成后，我们可以在任何位置执行`myopencode --version`。

如果bun link失败，可以尝试使用npm link：

```bash
cd packages/cli
npm link
```

### 2.5.3 验证版本命令

现在，让我们验证CLI是否正常工作。首先检查版本：

```bash
myopencode --version
```

如果一切配置正确，你应该能看到输出：`myopencode version 0.1.0`。

接下来，验证帮助命令：

```bash
myopencode --help
```

你应该能看到帮助信息，显示可用的命令和选项。

最后，尝试一个简单的命令：

```bash
myopencode start
```

你应该能看到`Starting development server...`的输出。

如果这些命令都能正常工作，说明我们的CLI框架已经搭建成功！

## 2.6 进阶配置与优化

### 2.6.1 添加Git Hooks

OpenCode项目使用了Husky来管理Git hooks，这允许我们在提交代码前执行各种检查。让我们也为自己的项目添加Git hooks支持。

首先，在项目根目录初始化Husky：

```bash
cd ../..
npm install -D husky
npx husky init
```

这会在项目根目录创建.husky目录和prepare脚本。

接下来，创建一个pre-commit hook来运行代码检查：

```bash
cat > .husky/pre-commit << 'EOF'
#!/bin/bash
echo "Running pre-commit checks..."
echo "Type checking..."
bun run typecheck
EOF

chmod +x .husky/pre-commit
```

现在，每次执行git commit时，Husky都会自动运行pre-commit脚本，执行类型检查。

### 2.6.2 配置代码格式化

良好的代码格式是团队协作的基础。OpenCode项目使用了Prettier来格式化代码。让我们配置Prettier：

```bash
cat > .prettierrc << 'EOF'
{
  "semi": false,
  "printWidth": 120,
  "tabWidth": 2,
  "useTabs": false,
  "singleQuote": true,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always"
}
EOF
```

创建.prettierignore文件来排除不需要格式化的文件：

```bash
cat > .prettierignore << 'EOF'
node_modules/
dist/
*.log
.DS_Store
EOF
```

现在，我们可以使用Prettier格式化代码：

```bash
bun run format
```

在package.json中添加format脚本：

```bash
cd packages/cli
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "@myopencode/cli",
  "version": "0.1.0",
  "type": "module",
  "license": "MIT",
  "bin": {
    "myopencode": "./bin/myopencode"
  },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "build": "bun run script/build.ts",
    "dev": "bun run --conditions=browser ./src/index.ts",
    "clean": "echo '清理构建产物...'",
    "format": "prettier --write src/",
    "lint": "echo '代码检查...'",
    "random": "echo 'Random script updated at $(date)' && echo 'Change queued successfully' && echo 'Another change made' && echo 'Yet another change' && echo 'One more change' && echo 'Final change' && echo 'Another final change' && echo 'Yet another final change'"
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "@types/bun": "catalog:",
    "@types/node": "catalog:",
    "prettier": "catalog:",
    "typescript": "catalog:"
  },
  "dependencies": {
    "typescript": "catalog:"
  }
}
EOF
```

### 2.6.3 创建开发工作流脚本

为了提高开发效率，我们可以在根目录的package.json中添加工作流脚本：

```bash
cd ../..
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "myopencode",
  "description": "AI-powered development tool - Monorepo tutorial",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.5",
  "scripts": {
    "dev": "bun run --cwd packages/cli --conditions=browser src/index.ts",
    "typecheck": "bun turbo typecheck",
    "build": "bun turbo build",
    "test": "echo '运行测试...'",
    "prepare": "husky",
    "format": "prettier --write 'packages/**/*.{ts,json,toml}'",
    "lint": "echo '运行代码检查...'",
    "clean": "bun turbo clean",
    "random": "echo 'Random script updated at $(date)' && echo 'Change queued successfully' && echo 'Another change made' && echo 'Yet another change' && echo 'One more change' && echo 'Final change' && echo 'Another final change' && echo 'Yet another final change'"
  },
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "catalog": {
      "@types/bun": "1.3.5",
      "@types/node": "22.13.9",
      "prettier": "3.6.2",
      "typescript": "5.8.2",
      "zod": "3.22.0"
    }
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "husky": "9.1.7",
    "prettier": "catalog:",
    "turbo": "2.5.6"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/myopencode"
  },
  "license": "MIT",
  "prettier": {
    "semi": false,
    "printWidth": 120
  }
}
EOF
```

这些脚本提供了一个统一的开发界面。`bun run dev`会启动CLI的开发模式，`bun run typecheck`会对所有包进行类型检查，`bun run format`会格式化所有代码。

## 2.7 本章小结与项目状态

### 2.7.1 完成的项目结构

经过本章的学习，我们已经成功搭建了一个与OpenCode架构一致的Monorepo项目。项目结构如下：

```
myopencode/
├── .git/                    # Git版本控制
├── .gitignore              # Git忽略规则
├── .husky/                  # Git hooks
│   └── pre-commit
├── bun.lockb               # Bun锁文件
├── bunfig.toml             # Bun配置
├── package.json            # 根包配置
├── tsconfig.json           # TypeScript配置
├── .prettierrc             # Prettier配置
├── .prettierignore         # Prettier忽略规则
└── packages/
    ├── cli/
    │   ├── bin/
    │   │   └── myopencode  # CLI入口脚本
    │   ├── script/
    │   │   └── build.ts    # 构建脚本
    │   ├── src/
    │   │   └── index.ts    # CLI核心实现
    │   ├── package.json    # CLI包配置
    │   └── tsconfig.json   # CLI TypeScript配置
    ├── sdk/
    │   ├── src/
    │   │   └── index.ts
    │   └── package.json
    └── app/
        ├── src/
        │   └── index.ts
        └── package.json
```

### 2.7.2 功能验证清单

让我们验证本章实现的所有功能：

**Bun环境验证：**

```bash
bun --version  # 应输出 1.3.5
```

**CLI命令验证：**

```bash
myopencode --version  # 应输出 myopencode version 0.1.0
myopencode --help     # 应显示帮助信息
myopencode start      # 应显示 Starting development server...
```

**项目结构验证：**

```bash
ls -la packages/cli/
ls -la packages/cli/bin/
ls -la packages/cli/src/
```

**依赖安装验证：**

```bash
ls node_modules/ | head -20
```

如果所有这些验证都通过了，恭喜你！你已经成功完成了本章的学习，掌握了Monorepo项目的搭建方法。

### 2.7.3 后续章节预告

完成环境搭建后，读者已经具备了继续探索OpenCode各项功能的基础。在下一章中，我们将深入CLI的具体实现，学习如何使用参数解析库、创建命令架构、设计交互式界面等。通过这些学习，你将能够为MyOpenCode添加更多实用的命令和功能，真正打造一个属于自己的AI辅助开发工具。

### 2.7.4 常见问题解答

**问：Bun安装失败怎么办？**

如果Bun安装失败，首先检查网络连接。curl命令需要能够访问bun.sh网站。如果网络有问题，可以尝试使用镜像源或代理。也可以从GitHub Releases页面下载预编译的二进制文件手动安装。

**问：myopencode命令找不到怎么办？**

首先确认已经执行了`bun link`或`npm link`命令。然后检查PATH环境变量是否包含Bun的全局bin目录（通常是~/.bun/bin）。可以使用`which myopencode`或`where myopencode`命令来定位命令位置。

**问：类型检查报错怎么办？**

首先确保已经运行了`bun install`安装所有依赖。然后检查tsconfig.json配置是否正确。如果是从头开始的项目，可能需要调整include和exclude模式。

**问：如何添加新的子包？**

在packages目录下创建新目录，添加package.json配置文件，然后在根目录的package.json workspaces.packages中添加对应的模式匹配即可。

**问：如何升级依赖版本？**

修改根目录package.json中catalog定义的版本号，然后运行`bun install`更新所有子包的依赖。

通过本章的学习，我们不仅完成了开发环境的搭建，更重要的是理解了OpenCode背后的设计哲学和架构决策。这些知识将在后续的实践中发挥重要作用，帮助你更好地理解和使用OpenCode提供的各项功能。
