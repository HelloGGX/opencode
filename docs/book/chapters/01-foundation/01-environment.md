# 第二章：环境搭建与CLI实操

在第一章中，我们深入探讨了AI辅助开发工具的全景架构与设计哲学，对现代化CLI工具的架构设计有了全面的理解。从本章开始，我们将从理论走向实践，通过亲手搭建开发环境来深化对这套架构的理解。本章将以业界成熟的Monorepo项目为蓝本，带领读者完成从零到一的环境搭建工作，目标是让每位读者都能在自己的机器上成功运行`opencode --version`命令，并理解这背后的技术实现细节。

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

我们的项目是一个基于 Typescript 的 Monorepo 项目。它包含了多个子项目，每个子项目都是一个独立的包。为了高效地管理这些复杂的依赖关系并提升开发体验，我们引入了 Bun 作为项目的核心工具链。Bun 不仅作为一个高性能的 JavaScript 运行时，更在本项目中扮演着包管理器（Package Manager）和任务执行器（Task Runner）的关键角色。利用 Bun 原生支持的 Workspaces（工作区）功能，我们可以无缝地在本地链接各个子包，实现依赖的统一管理与版本控制，从而彻底解决了传统 Monorepo 架构中常见的依赖幽灵（Phantom Dependencies）和安装速度缓慢的问题。

### 2.2.1 安装并验证Bun运行环境

Bun是由Jarred Sumner用Zig语言编写的JavaScript运行时，它相较于Node.js和Deno具有显著的性能优势。成熟的Monorepo项目选择Bun作为包管理器和运行时，主要原因是其卓越的启动速度和包安装性能。在大型Monorepo项目中，依赖安装往往是开发流程中的瓶颈环节，而Bun通过原生实现npm注册表协议、利用机器码编译以及优化文件IO操作，能够将安装时间大幅缩短。

首先，我们需要检查系统中是否已经安装了Bun。打开终端，执行以下命令：

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
这种克制的设计，有意避免了配置碎片化的问题，同时也降低了项目在不同运行时之间迁移的复杂度。当项目需要对 Bun 的默认行为进行明确约束时，可以在项目根目录创建 bunfig.toml 文件。例如：
```bash
cat > bunfig.toml << 'EOF'
[install]
registry = "https://registry.npmjs.org"
EOF

cat bunfig.toml
```
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
    "typecheck": "tsc --noEmit",
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

## 2.4 CLI入口点实现

### 2.4.1 创建CLI包的目录结构

现在，我们开始实现CLI的核心部分。首先需要创建CLI包的完整目录结构，包括入口脚本、源代码目录和构建配置：

```bash
cd packages/opencode
mkdir -p bin src
ls -la
```

我们创建了两个目录：`bin`目录用于存放入口脚本，这是用户执行`opencode`命令时首先加载的文件；`src`目录用于存放TypeScript源代码，实现CLI的具体功能。

接下来，我们创建CLI的入口脚本。这个脚本将负责检测平台、定位二进制文件并启动实际的CLI程序：

```bash
cat > bin/opencode << 'EOF'
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
const base = "opencode-" + platform + "-" + arch
const binary = platform === "windows" ? "opencode.exe" : "opencode"

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
    'It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing the "' +
      base +
      '" package',
  )
  process.exit(1)
}

run(resolved)
EOF

chmod +x bin/opencode
```

这个入口脚本与业界成熟的CLI实现几乎完全一致，它的工作原理如下：

脚本开头的`#!/usr/bin/env node`是Unix系统的shebang，告诉系统使用Node.js来执行这个脚本。

`run`函数是一个简单的进程启动器。它使用`child_process.spawnSync`同步启动目标可执行文件，并将所有命令行参数传递给它。`stdio: "inherit"`设置确保子进程的输入输出与父进程共享，用户可以直接与CLI交互。

脚本首先检查`OPENCODE_BIN_PATH`环境变量。如果设置了这个变量，脚本会直接使用它指定的路径，这是为高级用户提供的一种覆盖默认行为的机制。

接下来，脚本检测当前操作系统和CPU架构。`platformMap`和`archMap`将系统信息映射到CLI的二进制命名规范。例如，在macOS（M1芯片）上运行的系统会被映射为`opencode-darwin-arm64`。

`findBinary`函数负责在node_modules目录中查找正确的二进制文件。它从脚本所在目录开始，向上遍历目录树，直到找到匹配的二进制文件为止。这种设计确保了无论CLI包安装在什么位置，脚本都能正确定位二进制文件。

### 2.4.2 实现真正的版本命令

为了使`opencode --version`命令正常工作，我们需要在CLI包中实现真正的版本输出逻辑：

```bash
cd packages/opencode
mkdir -p src/commands

cat > src/commands/version.ts << 'EOF'
import { Command } from "commander"

export function createVersionCommand(): Command {
  const cmd = new Command("version")
    .alias("--version")
    .description("显示OpenCode版本信息")
    .action(() => {
      const pkg = JSON.parse(
        await Bun.file(import.meta.dir + "/../../package.json").text()
      )
      console.log(`OpenCode version ${pkg.version}`)
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
cd packages/opencode
bun add commander
```

现在可以测试版本命令：

```bash
cd ../../..
bun install
opencode --version
```

如果一切正常，你应该看到类似这样的输出：

```
OpenCode version 0.1.0
Environment: darwin/arm64
Node.js: v21.0.0
```

## 2.5 安装依赖并验证环境

### 2.5.1 执行依赖安装

现在我们已经完成了所有的配置工作，是时候安装项目的依赖包了。在Monorepo中，依赖安装是一个重要的步骤，它会解析所有工作区的依赖关系，并创建正确的符号链接。

首先，返回项目根目录：

```bash
cd ~/workspace/opencode
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

为了能够在终端中直接使用`opencode`命令，我们需要将CLI包链接到全局环境中。Bun提供了`bun link`命令来实现这个功能：

```bash
cd packages/opencode
bun link
```

执行这个命令后，Bun会在全局bin目录（通常是~/.bun/bin）中创建一个指向当前CLI包的符号链接。现在，你可以在任何位置执行`opencode`命令了。

让我们验证CLI命令是否正常工作：

```bash
cd ../..
opencode --help
```

如果一切正常，你应该能看到CLI的帮助信息。这表明我们的CLI入口脚本和依赖链接都配置正确。

## 2.6 版本控制与提交

### 2.6.1 初始化Git并提交项目状态

bun init已经帮我们创建了项目的基础文件，现在需要将这些文件纳入Git版本控制。执行初始提交：

```bash
git status
git add .
git commit -m "初始化项目：使用bun init创建基础配置"
git log --oneline -5
```

Git会显示我们刚刚完成的提交已经记录在案。

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
opencode --help
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

**CLI开发基础**：我们实现了与成熟CLI项目一致的入口脚本，理解了平台检测、路径解析和进程启动的技术细节。

**版本控制最佳实践**：我们建立了规范的Git提交流程，包括合理的.gitignore配置和语义化的提交信息。

### 2.7.3 后续章节预告

完成环境搭建后，读者已经具备了继续探索现代化CLI工具各项功能的基础。在下一章中，我们将深入CLI的具体实现，学习如何使用参数解析库、创建命令架构、设计交互式界面等。通过这些学习，你将能够为OpenCode添加更多实用的命令和功能，真正打造一个属于自己的AI辅助开发工具。

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

**问：opencode命令找不到怎么办？**

首先确认已经执行了`bun link`命令。然后检查PATH环境变量是否包含Bun的全局bin目录（通常是~/.bun/bin）。可以使用`which opencode`或`where opencode`命令来定位命令位置。

**问：类型检查报错怎么办？**

首先确保已经在项目根目录执行了`bun install`，所有依赖都已正确安装。检查TypeScript配置文件tsconfig.json是否存在且格式正确。确保使用了正确版本的TypeScript（通过catalog配置）。

**问：Monorepo依赖安装失败怎么办？**

检查网络连接是否正常。尝试删除node_modules目录和bun.lockb锁文件后重新安装。确保各子包的package.json配置正确，特别是workspace:协议的引用格式。

**问：如何清理构建产物？**

可以使用各子包中的clean脚本，或者手动删除dist目录和构建产物。对于node_modules目录，通常不需要手动清理，Bun会自动处理。

通过本章的学习，我们不仅完成了开发环境的搭建，更重要的是理解了现代化CLI工具背后的设计哲学和架构决策。这些知识将在后续的实践中发挥重要作用，帮助你更好地理解和使用Monorepo项目的各项功能。
