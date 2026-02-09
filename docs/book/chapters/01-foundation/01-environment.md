# 第二章：环境搭建与CLI实操

在第一章中，我们深入探讨了AI辅助开发工具的全景架构与设计哲学，对现代化CLI工具的架构设计有了全面的理解。从本章开始，我们将从理论走向实践，通过亲手搭建开发环境来深化对这套架构的理解。本章将以业界成熟的Monorepo项目为蓝本，带领读者完成从零到一的环境搭建工作。

## 2.1 项目初始化：从空目录开始

### 2.1.1 创建项目根目录

一切伟大的工程都始于一个空目录。让我们从零开始，逐步构建符合业界最佳实践的Monorepo项目。首先，我们需要在文件系统中创建一个新的项目目录，这个目录将成为我们整个项目的根节点。建议将项目放在用户主目录下的workspace文件夹中，这样既能保持整洁，又能与其他项目保持良好的隔离性。

打开终端，执行以下命令来创建项目结构：

```bash
mkdir -p ~/workspace/opencode
cd ~/workspace/opencode
pwd
```

执行完这些命令后，你应该看到终端显示`/Users/你的用户名/workspace/opencode`（macOS）或类似路径。这个目录将成为我们所有后续操作的基准点。在整个学习过程中，我们将始终保持在这个目录下执行命令，这样能够避免因路径问题导致的错误。

接下来，我们需要初始化Git版本控制系统。Git是现代软件开发的基础工具，它不仅能够追踪代码变更，还支持多人协作开发。业界成熟的CLI项目普遍使用Git作为版本控制系统，我们也将遵循这一选择：

```bash
git init
git config user.name "你的名字"
git config user.email "你的邮箱@example.com"
```

这两条git config命令分别设置了用户名和邮箱，这些信息会嵌入到每次提交中，用于标识代码的作者身份。完成这些设置后，我们的项目就已经具备了基本的版本控制能力。

## 2.2 Bun环境配置详解

我们的项目是一个基于 Typescript 的 Monorepo 项目。它包含了多个子项目，每个子项目都是一个独立的包。为了高效地管理这些复杂的依赖关系并提升开发体验，我们引入了 Bun 作为项目的核心工具链。它既是高性能的 JavaScript/TypeScript 运行时，又是包管理器和任务执行器。结合 Bun 原生支持的工作区（Workspaces）机制，可为大型 TypeScript Monorepo 提供一致、快速、可扩展的依赖管理与任务调度能力。

相比传统工具链（如 npm、yarn、pnpm 等），Bun 的一体化设计显著降低了工具之间的复杂性，提高了开发效率和 CI/CD 流水线性能。随着项目规模和依赖数量的增长，其优势愈发明显。

### 2.2.1 安装并验证Bun运行环境

前面提到，Bun 是一个集成式 JavaScript/TypeScript 工具包，它内部包含：快速运行时、包管理器、测试运行器及打包器等模块。具体来说：

- Bun 运行时（Runtime）：基于 Apple 的 JavaScriptCore 引擎开发，支持直接执行 .js、.ts、.jsx、.tsx 等文件，无需额外转译工具。其启动速度显著快于 Node.js，并且具有内置 TypeScript 支持。
- 包管理器（Package Manager）：兼容 npm/yarn 的协议，支持全局缓存和 Workspaces，在 Monorepo 环境中可实现跨包依赖自动链接、重复依赖去重及高效安装。
- 测试运行器与打包器：提供 Jest 兼容测试框架和内置打包能力，可直接运行项目测试和生成生产构建，无需额外配置。
Bun 的集成设计避免了在同一项目中引入多个工具（如 npm + Jest + esbuild 等）的碎片化复杂度，从而简化了配置和维护

你可以通过 Bun 官网提供的安装脚本，在 Windows、macOS 或 Linux 上快速安装 Bun。安装后，运行以下命令确认是否安装成功：

```bash
bun --version
```

如果系统已经安装了Bun，你会看到类似`1.3.5`的版本号输出。如果看到"command not found"或类似错误，说明系统中还没有安装Bun，这时需要执行安装程序。

在macOS和Linux系统上，Bun的安装非常简单。执行以下命令即可完成安装：

```bash
## Linux/macOS
curl -fsSL https://bun.sh/install | bash
```

这条命令会下载Bun的安装脚本并自动执行。安装脚本会自动将bun可执行文件添加到系统的PATH环境变量中，通常是`~/.bun/bin/bun`。安装完成后，你需要重新加载shell配置或打开新的终端窗口以使PATH变更生效：

```bash
# 如果您使用的是 zsh
source ~/.zshrc
# 如果您使用的是 bash
source ~/.bashrc
```

现在再次执行`bun --version`，你应该能看到Bun的版本号了。请注意，参考项目指定使用`bun@1.3.5`版本，如果你的Bun版本与此不同，可能会遇到兼容性问题。在这种情况下，可以使用Bun的版本管理器安装指定版本：

```bash
bun install -g bun@1.3.5
```
或者使用bun upgrade升级到最新版本, 笔者在写这篇文档时, Bun的最新版本是`1.3.9`

```bash
bun upgrade
```

对于Windows用户，Bun提供了专门的安装程序。可以通过PowerShell执行以下命令：

```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

或者从Bun的GitHub Releases页面下载预编译的Windows版本。值得注意的是，Windows版本的Bun在某些Unix特定的系统调用上可能存在差异，但成熟的Monorepo项目通过条件编译和特性检测机制确保了跨平台兼容性。

验证Bun安装成功后，我们还需要确认几个关键工具是否可用。Bun自带了包管理器、运行时和打包工具，我们可以通过以下命令验证这些功能：

```bash
bun --help
```

这个命令会显示Bun的帮助信息，列出所有可用的子命令。常见的子命令包括`bun install`（安装依赖）、`bun run`（运行脚本）、`bun test`（运行测试）和`bun build`（打包代码）等。花些时间熟悉这些命令的功能和用法，会大大提高后续的开发效率。

### 2.2.2 配置Bun包管理器设置

Bun的配置主要依赖于标准的package.json和tsconfig.json文件。对于需要Bun特定配置的场景，可以通过bunfig.toml文件进行额外设置。这个文件是可选的，没有它Bun也可以正常工作，但在某些高级场景下能够提供更精细的控制。
这是因为bunfig.toml 的设计目标并不是成为一个“全量配置中心”。它并不试图替代 package.json，也不承担构建系统或运行时参数集中管理的职责。从设计定位上看，bunfig.toml 更接近于一个 “Bun 行为补丁层”，主要用于补充以下几类场景：

- 安装行为的默认策略（如 npm registry 来源）
- 与 Bun CLI 行为直接相关的选项
- 无法通过 package.json 合理表达的 Bun 专有设置

这种克制的设计，有意避免了配置碎片化的问题，同时也降低了项目在不同运行时之间迁移的复杂度。当项目需要对 Bun 的默认行为进行明确约束时，可以在项目根目录创建 bunfig.toml 文件。 具体如下：

```bash
cat > bunfig.toml << 'EOF'
[install]
registry = "https://registry.npmjs.org"
exact = true
EOF

cat bunfig.toml
```
需要特别说明的是`exact = true`的配置项。它表示让安装时锁定确切版本（不写入 ^ 或 ~ 前缀），确保可复现构建结果。
目前，bunfig.toml 中最常用且稳定的配置节是 [install]，用于影响 bun install 及相关依赖解析行为。Bun 默认启用依赖缓存机制，但缓存行为并未通过 bunfig.toml 暴露为可自由组合的开关。Bun 的构建行为（如 bun build）以及运行时参数，通常通过：命令行参数、package.json 中的 scripts、环境变量来进行控制。因此，在工程实践中，应避免将 bunfig.toml 误用为类似 npm、yarn 或 webpack 的“集中式配置文件”。

## 2.3 Monorepo架构配置

### 2.3.1 初始化项目基础配置

Bun提供了便捷的项目初始化命令bun init，它可以快速创建符合最佳实践的package.json基础配置。在Monorepo项目中，我们可以使用bun init作为起点，然后在其基础上添加工作区配置。

首先，使用bun init初始化项目基础配置：

```bash
cd ~/workspace/opencode
bun init
```

bun init会交互式地引导你设置项目的基本信息，或者你可以通过参数直接指定：

```bash
bun init --yes
```

这个命令会自动创建多个项目必需文件，包括：
- package.json - 项目配置文件
- .gitignore - Git忽略规则配置
- tsconfig.json - TypeScript编辑器语法支持
- README.md - 项目说明文档
- index.ts - TypeScript入口文件
- CLAUDE.md - Claude AI助手配置（如果使用Claude）

执行完成后，让我们查看生成的文件：

```bash
ls -la
cat package.json
cat .gitignore
```

首先，让我们查看package.json的内容：

```json
{
  "name": "opencode",
  "module": "index.ts",
  "type": "module",
  "private": true,
  "devDependencies": {
    "@types/bun": "latest"
  },
  "peerDependencies": {
    "typescript": "^5"
  }
}
```

bun init还会自动创建一份完善的.gitignore文件，覆盖了常见的环境配置：

```bash
cat .gitignore
```

你应该会看到类似以下内容：

```
# dependencies (bun install)
node_modules

# output
out
dist
*.tgz

# code coverage
coverage
*.lcov

# logs
logs
_.log
report.[0-9]_.[0-9]_.[0-9]_.[0-9]_.json

# dotenv environment variable files
.env
.env.development.local
.env.test.local
.env.production.local
.env.local

# caches
.eslintcache
.cache
*.tsbuildinfo

# IntelliJ based IDEs
.idea

# Finder (MacOS) folder config
.DS_Store
```

这份.gitignore配置非常全面，涵盖了Node.js/Bun项目的所有常见场景，比手动创建的要完善得多。

bun init 生成的配置包含了项目的基本结构，但为了构建支持多包复用（Monorepo）的开发环境，并利用 Catalog 功能统一管理依赖版本，我们需要将配置文件更新为如下内容

```bash
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "opencode",
  "module": "index.ts",
  "description": "AI-powered development tool - Monorepo tutorial",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.5",
  "scripts": {
    "dev": "echo '开发模式启动...'",
    "typecheck": "echo '类型检查...'",
  },
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "catalog": {
      "@types/bun": "1.3.5",
      "typescript": "5.8.2",
    }
  },
  "devDependencies": {
    "husky": "9.1.7",
    "prettier": "3.6.2"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/opencode"
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

- `"$schema"`字段指向JSON Schema文件，IDE可以据此提供自动补全和验证功能。虽然这是可选的，但它能够大大提升开发体验，建议始终保留。
- `"name"`字段定义了项目的名称。这个名称会用于npm包的发布（如果是公开包）和工作区引用。在Monorepo中，通常使用组织名称作为前缀，如`@myorg/myapp`。
- `"module": "index.ts"`字段指定项目的入口文件。这个字段是可选的，但建议保留，因为Bun会默认使用这个字段作为项目的入口文件。
- `"description"`字段描述项目的用途。这是一个重要的字段，因为它会出现在npm包的README中，帮助其他开发者了解项目的功能。
- `"private": true`设置非常重要。对于内部项目或不想发布到npm的包，必须设置这个选项为true。如果忘记设置这个选项，npm publish会拒绝发布私有包。
- `"type": "module"`声明这个包使用ES Modules语法。这是Bun原生支持的模式，与现代JavaScript生态接轨。如果不使用这个选项，Bun会默认使用CommonJS语法。
- `"packageManager"`字段指定了项目使用的包管理器及其版本。`"bun@1.3.5"`表示项目要求使用Bun 1.3.5版本。这个字段不仅是对人类的提示，Bun和npm也会读取这个字段以确保使用正确的包管理器版本。
- `"workspaces"`是Monorepo配置的核心。它包含两部分：`packages` 定义了子包的位置模式，`catalog`定义了共享依赖的版本。
  - `workspaces.packages`使用glob模式匹配子包位置。`"packages/*"`表示packages目录下的所有直接子目录都是独立的工作区。这意味着packages/opencode、packages/sdk、packages/app都会被识别为独立的包，可以相互引用。
  - `workspaces.catalog`是Bun提供的一个强大特性，称为"目录版本控制"。开发者只需在根目录的 `package.json` 文件中定义一次依赖版本。子包会通过 `catalog:` 协议来引用这些版本。开发者在一处修改版本号，该修改会在全局生效。这种机制能确保所有包都使用相同版本的依赖，避免了版本冲突和不一致的问题。
- `"devDependencies"`字段定义了项目开发时需要的依赖。这些依赖在生产环境中不会被安装，也不会被打包到最终的输出中。在这个配置中，我们暂时只需要`@tsconfig/bun`、`husky`和`prettier`这三个开发工具。
- `"dependencies"`字段定义了项目运行时需要的依赖。。
- `"repository"`字段定义了项目的存储库信息，用于发布和下载。在这个配置中，我们使用了GitHub存储库`https://github.com/yourusername/opencode`。
- `"license"`字段定义了项目的许可证。这个字段是可选的，但建议每个项目都包含一个有效的许可证。在这个配置中，我们使用了MIT许可证。
- `"prettier"`字段定义了项目的Prettier配置。Prettier是一个代码格式化工具，它可以自动格式化代码，保持一致的代码风格。在这个配置中，我们设置了`"semi": false`表示不使用分号，`"printWidth": 120`表示每行代码最多120个字符。
- `"peerDependencies"`字段定义了项目运行时需要的依赖，但这些依赖不是直接被项目使用，而是被其他包引用。大白话：它不是“我需要什么”，而是“我希望你已经有什么”。从更工程化的视角来看，peerDependencies 本质上是在做一件事：把版本控制权上移一层。它让库的作者放弃对某些关键依赖的控制权，换取整个生态的一致性与可组合性。

前文提到，我们的项目是一个基于 Typescript 的 Monorepo 项目，我们将子包设计为独立的库，每个子包都可被外部项目直接引用。通过 Bun 工作区（workspaces）机制，将 TypeScript、Bun 类型库等核心工具集中在根层管理，避免多子包间的版本不一致和重复安装问题。借助 Bun 的 catalogs 功能，可实现依赖版本的统一控制和更简洁的依赖树结构，从而提升整个工作区的开发体验与可维护性。因此，我们建议将 @types/bun 、 typescript 等类型相关的核心依赖纳入 Catalog 管理。


### 2.3.2 创建子包opencode的目录结构

现在我们已经定义了Monorepo的结构，接下来需要创建第一个核心子包opencode的配置文件，它实现了主要的 CLI 应用、服务器和业务逻辑。这是我们实现`opencode --version`命令的核心，我们暂且只是创建其目录和配置文件，主要目的是方便说明和测试我们的Monorepo架构。

进入packages/opencode目录并创建包配置文件：

```bash
cd packages/opencode
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "version": "0.0.1",
  "name": "opencode",
  "type": "module",
  "license": "MIT",
  "private": true,
  "scripts": {
    "typecheck": "tsgo --noEmit",
    "test": "bun test",
    "build": "bun run script/build.ts",
    "dev": "bun run --conditions=browser ./src/index.ts",
  },
  "bin": {
    "opencode": "./bin/opencode"
  },
  "exports": {
    "./*": "./src/*.ts"
  },
  "devDependencies": {
    "@types/bun": "catalog:",
    "typescript": "catalog:"
  }
}
EOF
```

这个配置文件有几个值得注意的点：

`"private": true`, 这个设置非常重要。opencode包只该项目中内部使用，不会被发布到npm仓库。
`"bin"`字段定义了CLI入口点。`"opencode": "./bin/opencode"`告诉包管理器，当用户安装这个包时，需要创建一个名为`opencode`的命令，指向bin目录下的opencode脚本文件。
`"exports"`字段定义了包的导出路径。`"./*": "./src/*.ts"`告诉包管理器，当用户导入opencode时，应该从src目录下的对应文件中导出。
`"devDependencies"`使用了`"catalog:"`前缀。这意味着这些依赖的版本不是直接写在子包中，而是从根目录的catalog中读取。这种方式确保了opencode包的@types/bun和typescript依赖都统一使用根目录的版本号，避免了版本冲突和不一致的问题。

### 2.3.3 配置 Turbo 构建系统

在Monorepo项目中，随着子包数量的逐步增多，构建任务的管理往往会变得异常繁杂，因为不同的包可能配备各自独立的构建脚本，而且包与包之间常常存在复杂的依赖链条，例如A包的构建必须在B包之后才能启动，同时每次代码修改后如果盲目重新构建所有包，就会导致严重的资源和时间浪费。Turbo作为一款专为这类场景设计的构建编排工具，正好能有效缓解这些痛点，它的核心优势体现在几个关键方面：通过增量构建机制，只针对发生变化的包及其下游依赖进行处理，从而大幅压缩整体构建时长；借助智能缓存功能，自动存储并复用未改动包的构建产物，避免无谓的重复计算；此外，它还能精细管理任务间的依赖关系，确保所有操作按逻辑顺序顺畅执行；最后，利用并行执行策略，对那些相互独立的子任务自动分配多核CPU资源，进一步提升效率。
要初始化Turbo配置，首先切换到项目根目录如~/workspace/opencode，然后通过命令行创建turbo.json文件，内容包括一个标准的JSON schema引用，以及tasks字段来定义核心构建任务

```bash
cd ~/workspace/opencode
cat > turbo.json << 'EOF'
{
  "$schema": "https://turborepo.com/schema.json",
  "tasks": {
    "typecheck": {},
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "opencode#test": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
EOF
cat turbo.json
```
我们可以看到 typecheck任务采用空配置，这意味着Turbo会自动在所有子包中运行对应的typecheck脚本，通常对应package.json中的类型检查命令，由于它没有显式依赖，因此可以全局并行执行，提高检查速度。build任务则设置了dependsOn为["^build"]，这里的^符号巧妙地表示依赖于上游所有子包的build任务，这里的“上游”指的就是当前包的依赖项（Dependencies），同时outputs指定为["dist/**"]，用于精准缓存构建输出如dist目录下的文件，确保增量复用。opencode#test任务专属于名为opencode的子包，它依赖于上游的build任务，通过dependsOn ["^build"]来保证测试前已完成必要构建，而outputs设为空数组，因为测试通常不产生持久文件，这有助于避免无效缓存。整个配置完成后，可以通过cat turbo.json验证文件内容，确保一切就绪。

当运行 turbo run build 时，Turbo 会执行以下步骤：
1. 构建依赖图，确定包的构建顺序
2. 先构建无依赖的包（如 util、sdk）
3. 再构建依赖它们的包（如 plugin、app）
4. 最后构建 opencode（核心包）

当运行 turbo run test 时：
1. 先执行所有包的 build 任务
2. 然后并行执行 opencode 的测试任务


接下来，更新根目录的 package.json 添加 Turbo 相关的脚本,这里我们先替换script中的typecheck脚本为`"typecheck": "turbo run typecheck"`
```json
{
  "scripts": {
    "typecheck": "turbo run typecheck",
  }
}
```
该脚本会读取 turbo.json 配置，找到所有包含 typecheck 脚本的包，并根据依赖图并行运行所有子包的 TypeScript 类型检查。后面我们的各子包会普遍定义命令 "typecheck": "tsgo --noEmit"

### 2.3.4 配置TypeScript编译环境

合理的TypeScript配置对于保证本项目代码质量至关重要。bun init会自动创建一个基础的tsconfig.json用于编辑器智能提示:

```json
{
  "compilerOptions": {
    // Environment setup & latest features
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleDetection": "force",
    "jsx": "react-jsx",
    "allowJs": true,

    // Bundler mode
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,

    // Best practices
    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    // Some stricter flags (disabled by default)
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noPropertyAccessFromIndexSignature": false
  }
}
```

上面的tsconfig.json文件定义了TypeScript的编译选项。它包含了Bun的最佳实践配置，并添加了Monorepo项目需要的编译选项。这对本项目来说，这是一个重要的配置，因为它确保了所有子包的TypeScript代码都能在Bun环境中正确运行和编译。虽然上述 tsconfig.json 详尽地定义了适配 Bun 运行时环境所需的各项参数，但在 Monorepo 架构中，如果在每个子包内都完整复制这一大段配置，不仅造成代码冗余，后续升级维护也极易导致配置不一致。
为了遵循 DRY (Don't Repeat Yourself) 原则并确保所有子包始终与 Bun 的最新最佳实践保持同步，我们可以引入官方维护的预设配置包 @tsconfig/bun。它将上述所有针对 Bun 优化的编译选项（如 moduleResolution: "bundler", module: "Preserve" 等）封装在内，使我们能够通过继承的方式大幅简化项目配置。”

安装@tsconfig/bun预设配置包：

```bash
bun install @tsconfig/bun
```

同时考虑到各个子包都需要依赖@tsconfig/bun，我们需要确保各个子包的"@tsconfig/bun"版本和根目录的版本保持一致，因此在根目录的package.json中添加`"catalog": { "@tsconfig/bun": "1.0.9" }`。并确保在根目录和子包的package.json中添加`"devDependencies": {"@tsconfig/bun": "catalog:"}`

接下来根目录下的tsconfig.json配置内容可以简化为：
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {}
}
```
同时各子包中的tsconfig.json配置可以继承@tsconfig/bun的配置,并添加自定义的编译选项，我们以opencode包为例：

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "@opentui/solid",
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
    "types": [],
    "noUncheckedIndexedAccess": false,
    "customConditions": ["browser"],
    "paths": {
      "@/*": ["./src/*"],
      "@tui/*": ["./src/cli/cmd/tui/*"]
    }
  }
}
```
后续的章节在讲到开发opencode包时，会详细说明以上配置的具体细节。

## 2.4 代码质量与Git工作流配置

在现代软件开发中，代码质量保证和规范的Git工作流程是项目成功的关键因素。本节将介绍如何配置Prettier代码格式化工具、Husky Git钩子管理工具以及EditorConfig编辑器配置，确保团队协作时代码风格的一致性。

### 2.4.1 配置Prettier代码格式化工具

Prettier是一个强大的代码格式化工具，能够自动格式化JavaScript、TypeScript、CSS、HTML等多种文件类型。在Monorepo项目中，统一的代码风格对于维护代码质量至关重要。

首先，我们需要安装Prettier作为开发依赖：

```bash
bun add -d prettier
```
接下来创建Prettier配置文件`.prettierrc`，定义项目的代码格式化规则。虽然Prettier有合理的默认配置，但我们可以根据项目需求进行微调：

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "avoid"
}
```
在我们的Monorepo项目中，我们希望所有子包的代码都符合相同的格式化规则。因此，我们只需要在根目录的package.json中配置一个全局的Prettier配置：

```json
"prettier": {
    "semi": false,
    "printWidth": 120
  },
```

同时，创建`.prettierignore`文件来指定不需要格式化的文件, 在实际的opencode项目中， 仅仅忽略了以下文件：

```bash
sst-env.d.ts
desktop/src/bindings.ts
```
sst-env.d.ts 是自动生成的 TypeScript 声明文件，不需要格式化。而desktop/src/bindings.ts 是一个由 Tauri Specta 自动生成的文件，它提供了前端（TypeScript/JavaScript）与后端（Rust）之间的类型安全通信接，也不需要格式化。

在package.json中添加格式化脚本：

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

现在你可以使用`bun run format`命令格式化所有代码，或者使用`bun run format:check`检查代码格式是否符合规范。

### 2.4.2 配置Husky Git钩子管理工具

Husky是一个强大的Git钩子管理工具，可以在Git操作（如提交、推送）时自动执行特定脚本，确保代码质量。

安装Husky：

```bash
bun add -d husky
```

初始化Husky配置：

```bash
bun run husky init
```

这会在项目根目录自动创建`.husky`文件夹，该文件夹中包含了一些默认的 Git 钩子脚本，如 pre-commit、pre-push 等。
但并不是每个开发者都会知道需要手动执行 `bun run husky install` 命令来初始化Husky。因此，我们可以在项目根目录的 package.json 的scripts手动添加命令 `"prepare": "husky"`，它会在依赖安装完成后自动初始化 Husky 并确保 Git Hooks 生效。

接着我们创建pre-push钩子，确保在推送代码前进行类型检查：

```bash
cat > .husky/pre-push << 'EOF'
#!/bin/sh
set -e
# Check if bun version matches package.json
# keep in sync with packages/script/src/index.ts semver qualifier
bun -e '
import { semver } from "bun";
const pkg = await Bun.file("package.json").json();
const expectedBunVersion = pkg.packageManager?.split("@")[1];
if (!expectedBunVersion) {
  throw new Error("packageManager field not found in root package.json");
}
const expectedBunVersionRange = `^${expectedBunVersion}`;
if (!semver.satisfies(process.versions.bun, expectedBunVersionRange)) {
  throw new Error(`This script requires bun@${expectedBunVersionRange}, but you are using bun@${process.versions.bun}`);
}
if (process.versions.bun !== expectedBunVersion) {
  console.warn(`Warning: Bun version ${process.versions.bun} differs from expected ${expectedBunVersion}`);
}
'
bun typecheck
EOF

chmod +x .husky/pre-push
```
这个脚本会在 git push 命令执行前 自动运行，确保开发者使用的 Bun 版本与项目要求的版本兼容，推送前的代码通过了 TypeScript 类型检查，所有开发者使用相同的开发环境，避免因版本差异导致的问题。

现在我们测试一下pre-push钩子是否正常工作：

```bash
git add .
git commit -m "Initial commit"
git push
```

你会看到在push之前会执行命令typecheck，确保代码通过了类型检查。

```bash
$ bun turbo typecheck
turbo 2.5.6

• Packages in scope: opencode
• Running typecheck in 1 packages
• Remote caching disabled
opencode:typecheck: cache hit, replaying logs 179e8ac7911d4908
opencode:typecheck: 
opencode:typecheck: $ tsc --noEmit

 Tasks:    1 successful, 1 total
Cached:    1 cached, 1 total
  Time:    61ms >>> FULL TURBO
```
可以看到配置成功了。


### 2.4.3 配置EditorConfig
虽然 automated checks 能确保代码逻辑无误，但不同的操作系统（Windows vs macOS）和编辑器习惯（Tab vs Space）往往会导致许多看不见的‘脏’ diff（例如换行符差异）。为了让所有开发者在打开编辑器的那一刻就拥有完全一致的编码体验，我们需要配置 EditorConfig。”
EditorConfig帮助维护跨不同编辑器和IDE的代码风格一致性。创建`.editorconfig`文件：

```ini
root = true

[*]
charset = utf-8
insert_final_newline = true
end_of_line = lf
indent_style = space
indent_size = 2
max_line_length = 80
```



## 2.5 GitHub Actions CI/CD配置

现代软件开发离不开持续集成和持续部署（CI/CD）。本项目采用GitHub Actions作为CI/CD平台，配置了多个专业化的工作流来确保代码质量和自动化部署。

### 2.5.1 项目CI/CD架构概览

项目采用了多工作流架构，每个工作流负责不同的职责：

- **测试工作流** (`test.yml`): 负责运行单元测试和端到端测试
- **类型检查工作流** (`typecheck.yml`): 确保TypeScript代码类型安全
- **发布工作流** (`publish.yml`): 自动化版本管理和包发布
- **OpenCode集成工作流** (`opencode.yml`): 集成AI辅助开发工具

下面我们重点讲解其中2个工作流：测试工作流和类型检查工作流。后续的工作流我们会在需要的时候再进行补充和迭代。

### 2.5.2 创建GitHub Actions配置目录

首先创建必要的工作流目录结构：

```bash
mkdir -p .github/workflows
mkdir -p .github/actions/setup-bun
```

### 2.5.3 配置自定义Bun设置Action
Composite Action 允许我们将一组步骤封装为一个独立的 Action。对于使用者来说，它就是一个黑盒，我们只需要关注输入和输出，而无需关心内部的实现细节。
接下来，我们就以 setup-bun 为例，看看如何设计一个包含环境准备、缓存挂载和依赖安装的通用 Action。
首先，我们需要定义 Action 的元数据。与常规的 JavaScript Action 不同，Composite Action 的核心在于 runs 字段：

```bash
name: "Setup Bun"
description: "Setup Bun with caching and install dependencies"
runs:
  using: "composite" # 关键：声明这是一个组合式 Action
  steps:
    # ... 具体步骤
```
可以看到，我们将 using 设置为 "composite"，这告诉 GitHub Runner：“嘿，不要去找 index.js，请直接执行我定义的 steps。”
在构建过程中，依赖安装往往是最耗时的环节。为了提升性能，我们必须引入缓存机制。但是，直接使用普通的 Cache Action 有时会遇到命中率低或配置繁琐的问题。
在这里，我们选择了一种更为激进但也更高效的方案：将缓存挂载为磁盘。我们引入了 useblacksmith/stickydisk，它的作用类似于将一块持久化的硬盘挂载到 Runner 上：

```bash
name: "Setup Bun"
description: "Setup Bun with caching and install dependencies"
runs:
  using: "composite" # 关键：声明这是一个组合式 Action
  steps:
    - name: Mount Bun Cache
      uses: useblacksmith/stickydisk@v1
      with:
        key: ${{ github.repository }}-bun-cache
        path: ~/.bun
```

这样做的本质，是将易失性的构建环境与持久化的依赖存储分离开来。无论 Runner 如何销毁重建，~/.bun 目录下的内容都能像“快照”一样被恢复，从而极大缩短了 bun install 的时间。
环境准备好之后，接下来的工作就顺理成章了。我们需要安装 Bun，并执行依赖安装。
这里有一个细节值得注意：在设置 Bun 版本时，我们并没有硬编码版本号，而是指定了 bun-version-file: package.json,它表示从 package.json 中读取 Bun 版本。

```bash
name: "Setup Bun"
description: "Setup Bun with caching and install dependencies"
runs:
  using: "composite" # 关键：声明这是一个组合式 Action
  steps:
    - name: Mount Bun Cache
      uses: useblacksmith/stickydisk@v1
      with:
        key: ${{ github.repository }}-bun-cache
        path: ~/.bun

    - name: Setup Bun
      uses: oven-sh/setup-bun@v2
      with:
        bun-version-file: package.json
```
oven-sh/setup-bun@v2 是 Bun 官方维护的 Action。这样做可以确保本地开发、CI环境、不同的workflow三者使用完全一致的Bun版本。
最后，我们将所有这些逻辑整合到一个 Shell 脚本中，通过自动化命令生成最终的 action.yml 文件。这就得到了我们最终的实现方案：

```bash
cat > .github/actions/setup-bun/action.yml << 'EOF'
name: "Setup Bun"
description: "Setup Bun with caching and install dependencies"
runs:
  using: "composite"
  steps:
    - name: Mount Bun Cache
      uses: useblacksmith/stickydisk@v1
      with:
        key: ${{ github.repository }}-bun-cache
        path: ~/.bun

    - name: Setup Bun
      uses: oven-sh/setup-bun@v2
      with:
        bun-version-file: package.json

    - name: Install dependencies
      run: bun install
      shell: bash
EOF
```

### 2.5.4 配置多平台测试工作流

在上一节中，我们封装了通用的 setup-bun。现在，我们面临的第一个问题是：如何确保我们的代码在不同操作系统下表现一致？
通常情况下，开发者习惯在 macOS 或 Linux 本地环境中开发，但这往往会掩盖 Windows 环境下特有的路径分隔符、换行符或 shell 差异问题。

如果分别为 Linux 和 Windows 编写独立的 Workflow，不仅会引入大量重复配置，还会在后续维护中不断放大改动成本，这显然违背了 DRY 原则。为了解决这一问题，GitHub Actions 提供了 matrix（矩阵）策略，使我们能够在保留统一执行流程的前提下，对不同运行环境进行参数化配置。

在这种模式下，测试流程本身只需要定义一次，而操作系统、运行节点、依赖安装方式以及具体的测试命令等平台差异，则通过 matrix 作为配置项传入。GitHub Actions 会基于 matrix 中的每一组配置，自动生成并执行对应的测试任务。首先我们创建文件`test.yml` 并添加以下内容：

首先是 **触发机制**。我们定义了三种触发时机：
* 当有代码推送到 `dev` 分支时；
* 当有针对 `dev` 分支的 Pull Request 被创建或更新时；
* 以及通过 `workflow_dispatch` 允许手动触发。

这种设计体现了**“尽早发现”**的原则。在 Pull Request 阶段就拦截错误，可以避免污染主分支的稳定性，将问题解决在合并之前。

```yaml
name: test

on:
  push:
    branches:
      - dev
  pull_request:
  workflow_dispatch:
jobs:
  test:
    strategy:
      fail-fast: false # 关键：避免单点失败导致整个矩阵立即终止，我们需要看到所有平台的测试结果
      matrix:
        settings:
          - name: linux
            host: ubuntu-latest
            playwright: bunx playwright install --with-deps
            workdir: .
            command: |
              git config --global user.email "XXXX"
              git config --global user.name "opencode"
              bun turbo test
          - name: windows
            host: windows-latest
            playwright: bunx playwright install
            workdir: packages/app
            command: bun test:e2e:local
```


在该 matrix 中，我们定义了两个测试配置：linux 和 windows。它们共享同一套测试流程，但在运行节点（host）、Playwright 的安装方式、工作目录以及最终执行的测试命令上各自独立，从而准确反映不同操作系统下的真实运行环境。这里有两个细节值得注意：
1. 我们将 fail-fast 设置为 false。这是因为在 CI 环境中，Windows 任务通常比 Linux 慢。如果 Linux 任务失败了，我们通常仍希望看到 Windows 任务的结果，以便判断这是否是一个特定平台的 Bug，还是通用逻辑的错误。
2. Linux 环境下运行 Playwright 通常需要额外安装系统依赖（--with-deps），而 Windows 环境通常不需要或已预置，Matrix 让我们能轻松处理这种差异。

定义好了矩阵，下一步是将这些配置映射到真实的虚拟机上。

```yaml
    runs-on: ${{ matrix.settings.host }}
    defaults:
      run:
        shell: bash
```
通过将 runs-on 设置为 ${{ matrix.settings.host }}，GitHub Actions 会为矩阵中的每组配置创建一个独立的 Job，并在指定的操作系统（如 Linux 或 Windows）上运行相同的测试步骤。这样，Linux 和 Windows 的差异就直接由运行环境来处理，而不需要在代码中用 if-else 分支来手动区分。
但这里更值得玩味的是 `defaults.run.shell: bash`。我们知道，Windows 的原生 Shell 是 PowerShell 或 CMD，而 Linux 是 Bash。如果任由默认行为发生，我们在编写后续的 steps 时，就需要区分这两者的差异。通过设置 `shell: bash`，强制 GitHub Actions 在 Windows 环境中也使用 Git Bash 来执行命令, 这使得我们可以放心地在 steps 中使用 rm -rf、export 等标准 Linux 命令，而无需为 Windows 编写繁琐的 PowerShell 替代方案。。

接下来我们配置steps：

```yaml
 steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Bun
        uses: ./.github/actions/setup-bun
```
第一步，我们需要检出代码仓库到CI运行环境中。使用官方的checkout Action版本4，确保获取最新代码和完整的Git历史。默认的checkout使用只读权限，为了能够在后续步骤中执行如bun install等需要写入权限的操作，我们需要传递token参数，将GITHUB_TOKEN作为写入权限的凭证。
第二步，我们使用自定义的setup-bun Action，将Bun安装到CI运行环境中。该Action会利用缓存机制，避免重复安装，显著提升构建效率。

后续步骤涉及测试库的安装和执行测试命令。根据不同平台的配置，我们需要在不同的工作目录下执行命令。从前面可以看出，我们使用playwright作为测试库，它是微软开发的现代化端到端测试工具，用于测试Web应用程序在不同浏览器中的行为。我们这里暂时不编写后续的steps，等到我们开发子包opencode时， 开始编写测试用例，才会涉及该部分的CI/CD流程，因此放到后续完善。

当前的测试工作流配置如下：

```yaml
name: test

on:
  push:
    branches:
      - dev
  pull_request:
  workflow_dispatch:
jobs:
  test:
    name: test (${{ matrix.settings.name }})
    strategy:
      fail-fast: false
      matrix:
        settings:
          - name: linux
            host: ubuntu-latest
            playwright: bunx playwright install --with-deps
            workdir: .
            command: |
              git config --global user.email "你的邮箱"
              git config --global user.name "你的用户名"
              bun turbo test
          - name: windows
            host: windows-latest
            playwright: bunx playwright install
            workdir: packages/app
            command: bun test:e2e:local
    runs-on: ${{ matrix.settings.host }}
    defaults:
      run:
        shell: bash
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Bun
        uses: ./.github/actions/setup-bun
```

#### 2.5.5 配置类型检查工作流
我们在 `.github/workflows` 目录下创建一个名为 `typecheck.yml` 的配置文件。这个文件定义了 GitHub Actions 的类型检查工作流，其核心内容如下：

```bash
cat > .github/workflows/typecheck.yml << 'EOF'
name: typecheck
on:
  push:
    branches: [dev]
  pull_request:
    branches: [dev]
  workflow_dispatch:

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Bun
        uses: ./.github/actions/setup-bun

      - name: Run typecheck
        run: bun typecheck

EOF
```
首先是 **运行环境**。这里我们指定了 `runs-on: blacksmith-4vcpu-ubuntu-2404`。选择合适的运行环境是 CI 优化的重要一环，使用性能更强的 Runner（如 Blacksmith 提供的实例）可以显著减少 CI 的排队和执行时间，从而加速开发者的反馈循环。整个过程分为三步：
1. **检出代码**：这是所有 CI 流程的基石。
2. **环境准备**：通过 `setup-bun` 初始化运行时环境。这里假设我们已经封装了一个复用的 Action 来统一管理 Bun 的版本和配置，这符合我们在软件工程中推崇的 DRY（Don't Repeat Yourself）原则。
3. **执行检查**：运行 `bun typecheck`。

需要注意的是，这里的 `bun typecheck` 通常是在 `package.json` 中定义的脚本，其底层往往调用了 `"typecheck": "tsgo --noEmit"`。`--noEmit` 标志非常关键，它告诉编译器：“我们只需要检查类型是否正确，不需要输出任何编译后的文件。”

通过这样一个独立且严谨的工作流，我们成功地将类型安全检查与构建过程解耦。


这种配置方式具有以下优势：

1. **环境一致性**: 使用自定义Action确保所有工作流使用相同的Bun版本和缓存策略
2. **多平台支持**: 在Linux和Windows上运行测试，确保跨平台兼容性
3. **智能触发**: 根据文件变更路径智能触发相应工作流，提高CI效率
4. **版本管理**: 支持手动和自动版本管理，灵活应对不同发布需求
5. **资源优化**: 使用矩阵策略并行执行测试，充分利用CI资源

提交这些配置到版本控制：

```bash
git add .
git commit -m "添加专业化的GitHub Actions CI/CD配置"
git push origin dev
```

配置完成后，项目将具备完整的自动化测试、类型检查能力, 现在我们去github 仓库中，点击Actions按钮，你应该可以看到我们自定义的工作流test、typecheck已经配置好了，并且已经在开始运行了。

后续我们讲从实际问题出发，逐步补充迭代剩余的工作流，例如发布工作流。
