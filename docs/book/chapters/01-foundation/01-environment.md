# 第二章：环境搭建与CLI实操
在第一章中，我们深入探讨了OpenCode的全景架构与设计哲学，对opencode的架构设计有了全面的理解。从本章开始，我们将从理论走向实践，通过亲手搭建开发环境来深化对这套架构的理解。本章将以OpenCode官方仓库为蓝本，带领读者完成从零到一的环境搭建工作，目标是让每位读者都能在自己的机器上成功运行`opencode --version`命令，并理解这背后的技术实现细节。

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

### 2.1.3 补充OpenCode核心目录

除了基础的项目结构，我们还需要创建与OpenCode一致的核心目录。这些目录虽然在本教程中不会立即使用，但为后续章节的功能扩展打下基础。

```bash
mkdir -p .opencode/agent
mkdir -p .opencode/command
mkdir -p .opencode/tool

mkdir -p _bmad/_config
mkdir -p _bmad/core
mkdir -p _bmad/bmm/agents
mkdir -p _bmad/bmm/teams

mkdir -p infra

mkdir -p .github/workflows

# mkdir -p nix
```

这些目录结构的创建遵循OpenCode项目的最佳实践，为后续的Agent配置、多平台支持等功能预留了扩展空间。

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
source ~/.zshrc
source ~/.bashrc
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
cache = true
clean = false
dry-run = false
global = false
locked = true

[pack]
prepend = "#!/usr/bin/env bun"
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

`findBinary`函数负责在node_modules目录中查找正确的二进制文件。它从脚本所在目录开始，向上遍历目录树，直到找到匹配的二进制文件为止。这种设计确保了无论CLI包安装在什么位置，脚本都能正确定位二进制文件。

### 2.4.2 实现真正的版本命令

为了使`myopencode --version`命令正常工作，我们需要在CLI包中实现真正的版本输出逻辑：

```bash
cd packages/cli
mkdir -p src/commands

cat > src/commands/version.ts << 'EOF'
import { Command } from "commander"

export function createVersionCommand(): Command {
  const cmd = new Command("version")
    .alias("--version")
    .description("显示MyOpenCode版本信息")
    .action(() => {
      const pkg = JSON.parse(
        await Bun.file(import.meta.dir + "/../../package.json").text()
      )
      console.log(`MyOpenCode version ${pkg.version}`)
      console.log(`Environment: ${process.platform}/${process.arch}`)
      console.log(`Node.js: ${process.version}`)
    })
  
  return cmd
}
EOF

cat > src/index.ts << 'EOF'
import { Command } from "commander"
import { createVersionCommand } from "./commands/version"

const program = new Command()

program.addCommand(createVersionCommand())

await program.parseAsync()
EOF
```

为了使用Commander库，我们需要更新CLI包的依赖：

```bash
cd packages/cli
bun add commander
```

现在可以测试版本命令：

```bash
cd ../../..
bun install
myopencode --version
```

如果一切正常，你应该看到类似这样的输出：

```
MyOpenCode version 0.1.0
Environment: darwin/arm64
Node.js: v21.0.0
```

## 2.5 安装依赖并验证环境

### 2.5.1 执行依赖安装

现在我们已经完成了所有的配置工作，是时候安装项目的依赖包了。在Monorepo中，依赖安装是一个重要的步骤，它会解析所有工作区的依赖关系，并创建正确的符号链接。

首先，返回项目根目录：

```bash
cd ~/workspace/myopencode
```

然后，执行Bun的依赖安装命令：

```bash
bun install
```

这个命令会执行以下操作：

首先，解析整个Monorepo的依赖关系图，包括根目录和所有子包的依赖。

其次，从npm注册表下载所需的包到根目录的node_modules中。

然后，根据package.json中的catalog配置，为各个子包安装正确版本的依赖。

最后，创建workspace:协议的符号链接，使各子包能够相互引用。

安装过程可能会持续几分钟，具体时间取决于网络速度和依赖数量。如果遇到网络问题，可以尝试使用镜像源或代理。

安装完成后，我们可以验证依赖是否正确安装：

```bash
ls node_modules/ | head -20
```

这个命令会列出node_modules目录中的前20个包。你应该能看到根目录的依赖（如bun、typescript等）以及各子包的依赖。

### 2.5.2 链接CLI命令

为了能够在终端中直接使用`myopencode`命令，我们需要将CLI包链接到全局环境中。Bun提供了`bun link`命令来实现这个功能：

```bash
cd packages/cli
bun link
```

执行这个命令后，Bun会在全局bin目录（通常是~/.bun/bin）中创建一个指向当前CLI包的符号链接。现在，你可以在任何位置执行`myopencode`命令了。

让我们验证CLI命令是否正常工作：

```bash
cd ../..
myopencode --help
```

如果一切正常，你应该能看到CLI的帮助信息。这表明我们的CLI入口脚本和依赖链接都配置正确。

## 2.6 版本控制与提交

### 2.6.1 检查并提交项目状态

我们已经完成了环境搭建的所有步骤，现在应该将项目状态提交到Git版本控制中。首先，让我们检查当前的Git状态：

```bash
git status
```

这个命令会显示所有已修改、新增或删除的文件。你应该能看到我们创建的所有配置文件和目录。

为了更好地理解项目的变更，让我们查看具体的差异：

```bash
git diff --stat
```

这个命令会显示每个文件的变更统计信息。

现在，让我们提交这些变更：

```bash
git add .
git commit -m "完成环境搭建：Monorepo配置、Bun环境、CLI基础结构"
git log --oneline -5
```

提交完成后，Git会显示提交历史的最后5条记录。我们可以看到刚刚完成的提交已经记录在案。

## 2.7 功能验证与总结

### 2.7.1 开发环境验证清单

让我们验证本章实现的所有功能，确保环境搭建工作已经完成：

**Bun环境验证：**

```bash
bun --version
bun --help
```

确保Bun版本为1.3.5，并且能看到所有可用的子命令。

**项目结构验证：**

```bash
tree -L 2 -I node_modules
```

这个命令会以树形结构显示项目目录，排除node_modules目录。你应该能看到根目录的结构以及packages下各子包的目录。

**CLI命令验证：**

```bash
myopencode --help
```

确保CLI命令能够正确执行并显示帮助信息。

**TypeScript配置验证：**

```bash
bun run typecheck
```

这个命令会运行根目录package.json中定义的typecheck脚本，验证TypeScript配置是否正确。

**构建脚本验证：**

```bash
bun run build
```

这个命令会运行构建脚本，验证构建配置是否正确。

### 2.7.2 本章知识点总结

通过本章的学习，我们掌握了以下核心知识点：

**Monorepo架构原理**：我们理解了Monorepo的核心概念，包括工作区（workspace）的定义、依赖解析机制以及子包之间的相互引用方式。通过使用Bun的catalog功能，我们实现了集中化的依赖版本管理。

**Bun包管理器特性**：我们深入了解了Bun的各种特性，包括其卓越的安装性能、bunfig.toml配置文件的用法以及与其他包管理器的差异。Bun的catalog机制为大型项目提供了强大的版本控制能力。

**TypeScript工程化配置**：我们学习了如何配置生产级别的TypeScript环境，包括继承社区最佳实践、配置严格的类型检查以及设置合理的包含和排除规则。

**CLI开发基础**：我们实现了与OpenCode一致的CLI入口脚本，理解了平台检测、路径解析和进程启动的技术细节。

**版本控制最佳实践**：我们建立了规范的Git提交流程，包括合理的.gitignore配置和语义化的提交信息。

### 2.7.3 后续章节预告

完成环境搭建后，读者已经具备了继续探索OpenCode各项功能的基础。在下一章中，我们将深入CLI的具体实现，学习如何使用参数解析库、创建命令架构、设计交互式界面等。通过这些学习，你将能够为MyOpenCode添加更多实用的命令和功能，真正打造一个属于自己的AI辅助开发工具。

## 2.8 GitHub Actions CI/CD配置

为了使项目具备持续集成和持续部署的能力，我们需要配置GitHub Actions工作流。虽然这些配置在本教程中不会立即使用，但它们对于后续章节的自动化测试和部署至关重要。

首先，创建GitHub工作流配置目录：

```bash
mkdir -p .github/workflows
```

然后，创建基础的CI工作流：

```bash
cat > .github/workflows/ci.yml << 'EOF'
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Bun
      uses: oven-sh/setup-bun@v1
      with:
        bun-version: 1.3.5
    
    - name: Install dependencies
      run: bun install
    
    - name: Typecheck
      run: bun run typecheck
    
    - name: Build
      run: bun run build
    
    - name: Run tests
      run: bun run test
EOF
```

创建类型检查工作流：

```bash
cat > .github/workflows/typecheck.yml << 'EOF'
name: TypeCheck

on:
  push:
    paths:
      - '**.ts'
      - '**.tsx'
      - 'tsconfig.json'
      - 'bunfig.toml'

jobs:
  typecheck:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Bun
      uses: oven-sh/setup-bun@v1
      with:
        bun-version: 1.3.5
    
    - run: bun install
    
    - name: Run typecheck
      run: bun run typecheck
EOF
```

提交这些配置到版本控制：

```bash
git add .
git commit -m "添加GitHub Actions CI/CD配置"
git push origin main
```

配置完成后，每次推送代码到main分支或创建Pull Request时，GitHub会自动运行类型检查、构建和测试流程。

## 2.9 常见问题解答

**问：Bun安装失败怎么办？**

如果Bun安装失败，首先检查网络连接。curl命令需要能够访问bun.sh网站。如果网络有问题，可以尝试使用镜像源或代理。也可以从GitHub Releases页面下载预编译的二进制文件手动安装。

**问：myopencode命令找不到怎么办？**

首先确认已经执行了`bun link`命令。然后检查PATH环境变量是否包含Bun的全局bin目录（通常是~/.bun/bin）。可以使用`which myopencode`或`where myopencode`命令来定位命令位置。

**问：类型检查报错怎么办？**

首先确保已经在项目根目录执行了`bun install`，所有依赖都已正确安装。检查TypeScript配置文件tsconfig.json是否存在且格式正确。确保使用了正确版本的TypeScript（通过catalog配置）。

**问：Monorepo依赖安装失败怎么办？**

检查网络连接是否正常。尝试删除node_modules目录和bun.lockb锁文件后重新安装。确保各子包的package.json配置正确，特别是workspace:协议的引用格式。

**问：如何清理构建产物？**

可以使用各子包中的clean脚本，或者手动删除dist目录和构建产物。对于node_modules目录，通常不需要手动清理，Bun会自动处理。

通过本章的学习，我们不仅完成了开发环境的搭建，更重要的是理解了OpenCode背后的设计哲学和架构决策。这些知识将在后续的实践中发挥重要作用，帮助你更好地理解和使用OpenCode提供的各项功能。

