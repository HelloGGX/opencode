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

Bun 的集成设计避免了在同一项目中引入多个工具（如 npm + Jest + esbuild 等）的碎片化复杂度，从而简化了配置和维护。

你可以通过 Bun 官网提供的安装脚本，在 Windows、macOS 或 Linux 上快速安装 Bun。安装后，运行以下命令确认是否安装成功：

```bash
bun --version
```

如果系统已经安装了 Bun，你会看到类似 `1.3.8` 的版本号输出。如果看到 "command not found" 或类似错误，说明系统中还没有安装 Bun，这时需要执行安装程序。

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

现在再次执行 `bun --version`，你应该能看到 Bun 的版本号了。参考项目指定使用 `bun@1.3.8` 版本，如果遇到版本兼容问题，可以使用 `bun upgrade` 升级到与项目兼容的版本（本文档编写时推荐使用 1.3.x 版本系列）。

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

这是因为bunfig.toml 的设计目标并不是成为一个"全量配置中心"。它并不试图替代 package.json，也不承担构建系统或运行时参数集中管理的职责。从设计定位上看，bunfig.toml 更接近于一个 "Bun 行为补丁层"，主要用于补充以下几类场景：

- 安装行为的默认策略（如 npm registry 来源）
- 与 Bun CLI 行为直接相关的选项
- 无法通过 package.json 合理表达的 Bun 专有设置

这种克制的设计，有意避免了配置碎片化的问题，同时也降低了项目在不同运行时之间迁移的复杂度。当项目需要对 Bun 的默认行为进行明确约束时，可以在项目根目录创建 bunfig.toml 文件。具体如下：

```bash
cat > bunfig.toml << 'EOF'
[install]
registry = "https://registry.npmjs.org"
exact = true
EOF

cat bunfig.toml
```

需要特别说明的是`exact = true`的配置项。它表示让安装时锁定确切版本（不写入 ^ 或 ~ 前缀），确保可复现构建结果。

目前，bunfig.toml 中最常用且稳定的配置节是 [install]，用于影响 bun install 及相关依赖解析行为。Bun 默认启用依赖缓存机制，但缓存行为并未通过 bunfig.toml 暴露为可自由组合的开关。Bun 的构建行为（如 bun build）以及运行时参数，通常通过：命令行参数、package.json 中的 scripts、环境变量来进行控制。因此，在工程实践中，应避免将 bunfig.toml 误用为类似 npm、yarn 或 webpack 的"集中式配置文件"。

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

bun init 生成的配置包含了项目的基本结构，但为了构建支持多包复用（Monorepo）的开发环境，并利用 Catalog 功能统一管理依赖版本，我们需要将配置文件更新为如下内容：

```bash
cat > package.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "opencode",
  "module": "index.ts",
  "description": "AI-powered development tool - Monorepo tutorial",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.8",
  "scripts": {
    "dev": "echo '开发模式启动...'",
    "typecheck": "echo '类型检查...'"
  },
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "catalog": {
      "@types/bun": "1.3.8",
      "typescript": "5.8.2"
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
  "peerDependencies": {
    "typescript": "^5"
  }
}
EOF

cat package.json
```

让我们详细分析这个配置文件的各个部分：

| 字段 | 类型 | 说明 |
|------|------|------|
| `$schema` | string | 指向JSON Schema文件，IDE可据此提供自动补全和验证功能 |
| `name` | string | 项目名称，用于npm包发布和工作区引用 |
| `module` | string | 指定项目入口文件，Bun默认使用此字段 |
| `description` | string | 项目描述，会出现在npm包的README中 |
| `private` | boolean | 设为`true`防止意外发布到npm仓库 |
| `type` | string | 设为`module`声明使用ES Modules语法 |
| `packageManager` | string | 指定包管理器及版本，如`bun@1.3.8` |
| `workspaces` | object | Monorepo配置，包含`packages`子包位置和`catalog`共享依赖 |
| `devDependencies` | object | 开发时依赖，不会在生产环境安装 |
| `repository` | object | 项目仓库信息 |
| `license` | string | 项目许可证 |
| `peerDependencies` | object | 对等依赖，定义包对其他包的版本要求 |

其中`workspaces`是Monorepo配置的核心：
- `workspaces.packages`使用glob模式匹配子包位置，`"packages/*"`表示packages目录下所有直接子目录都是独立的工作区
- `workspaces.catalog`是Bun提供的"目录版本控制"特性，只需在根目录定义一次依赖版本，子包通过`catalog:`协议引用

同时为了进一步确保依赖版本的一致性，Bun提供了`overrides`配置，可以强制所有子包使用根目录定义的特定依赖版本：

```json
"overrides": {
  "@types/bun": "catalog:"
}
```

前文提到，我们的项目是一个基于 Typescript 的 Monorepo 项目，我们将子包设计为独立的库，每个子包都可被外部项目直接引用。通过 Bun 工作区（workspaces）机制，将 TypeScript、Bun 类型库等核心工具集中在根层管理，避免多子包间的版本不一致和重复安装问题。借助 Bun 的 catalogs 功能，可实现依赖版本的统一控制和更简洁的依赖树结构，从而提升整个工作区的开发体验与可维护性。

### 2.3.2 创建子包opencode的目录结构

现在我们已经定义了Monorepo的结构，接下来需要创建第一个核心子包opencode的配置文件，它实现了主要的 CLI 应用、服务器和业务逻辑。这是我们实现`opencode --version`命令的核心，我们暂且只是创建其目录和配置文件，主要目的是方便说明和测试我们的Monorepo架构。

首先创建 packages/opencode 目录：

```bash
mkdir -p packages/opencode
mkdir -p packages/opencode/src
touch packages/opencode/src/index.ts
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
    "dev": "bun run --conditions=browser ./src/index.ts"
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

- `"private": true`：这个设置非常重要。opencode包只在该项目中内部使用，不会被发布到npm仓库。
- `"bin"`字段定义了CLI入口点。`"opencode": "./bin/opencode"`告诉包管理器，当用户安装这个包时，需要创建一个名为`opencode`的命令，指向bin目录下的opencode脚本文件。
- `"exports"`字段定义了包的导出路径。`"./*": "./src/*.ts"`告诉包管理器，当用户导入opencode时，应该从src目录下的对应文件中导出。
- `"devDependencies"`使用了`"catalog:"`前缀。这意味着这些依赖的版本不是直接写在子包中，而是从根目录的catalog中读取。这种方式确保了opencode包的@types/bun、typescript依赖都统一使用根目录的版本号，避免了版本冲突和不一致的问题。

**重要说明**：上面的配置是一个精简版本。在实际的大型项目中，子包通常会有更复杂的依赖列表，包括特定于该包的开发工具、语言服务器协议实现、打包器配置等。读者在跟随本教程时，可以先使用这个精简配置，随着项目复杂度的增加再逐步添加必要的依赖。

### 2.3.3 配置 Turbo 构建系统

在Monorepo项目中，随着子包数量的逐步增多，构建任务的管理往往会变得异常繁杂，因为不同的包可能配备各自独立的构建脚本，而且包与包之间常常存在复杂的依赖链条，例如A包的构建必须在B包之后才能启动，同时每次代码修改后如果盲目重新构建所有包，就会导致严重的资源和时间浪费。

Turbo作为一款专为这类场景设计的构建编排工具，正好能有效缓解这些痛点，它的核心优势体现在几个关键方面：通过增量构建机制，只针对发生变化的包及其下游依赖进行处理，从而大幅压缩整体构建时长；借助智能缓存功能，自动存储并复用未改动包的构建产物，避免无谓的重复计算；此外，它还能精细管理任务间的依赖关系，确保所有操作按逻辑顺序顺畅执行；最后，利用并行执行策略，对那些相互独立的子任务自动分配多核CPU资源，进一步提升效率。
首先安装Turbo：

```bash
bun add -D turbo
```
接着初始化Turbo配置，切换到项目根目录，然后通过命令行创建turbo.json文件：

```bash
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

让我们详细分析这个Turbo配置：

- `typecheck`任务采用空配置，这意味着Turbo会自动在所有子包中运行对应的typecheck脚本。由于它没有显式依赖，因此可以全局并行执行，提高检查速度。
- `build`任务设置了`dependsOn: ["^build"]`，这里的`^`符号巧妙地表示依赖于上游所有子包的build任务。这里的"上游"指的就是当前包的依赖项（Dependencies）。同时`outputs`指定为`["dist/**"]`，用于精准缓存构建输出如dist目录下的文件，确保增量复用。
- `opencode#test`任务专属于名为opencode的子包，它依赖于上游的build任务，通过`dependsOn ["^build"]`来保证测试前已完成必要构建，而`outputs`设为空数组，因为测试通常不产生持久文件，这有助于避免无效缓存。

当运行 `turbo run build` 时，Turbo 会执行以下步骤：
1. 构建依赖图，确定包的构建顺序
2. 先构建无依赖的包（如 util、sdk）
3. 再构建依赖它们的包（如 plugin、app）
4. 最后构建 opencode（核心包）

当运行 `turbo run test` 时：
1. 先执行所有包的 build 任务
2. 然后并行执行 opencode 的测试任务

接下来，更新根目录的 package.json 添加 Turbo 相关的脚本：

```json
{
  "scripts": {
    "typecheck": "bun turbo typecheck"
  }
}
```

该脚本会读取 turbo.json 配置，找到所有包含 typecheck 脚本的包，并根据依赖图并行运行所有子包的 TypeScript 类型检查。子包中通常定义 `"typecheck": "tsc --noEmit"` 命令来执行类型检查。

**重要说明**：这里使用 `bun turbo` 而不是直接使用 `turbo` 命令。原因是：
1. `bun turbo` 会自动使用项目指定的 Bun 版本
2. 确保在不同开发环境中行为一致
3. 利用 Bun 的执行效率加速 Turbo 的启动

### 2.3.4 配置TypeScript编译环境

合理的TypeScript配置对于保证本项目代码质量至关重要。bun init会自动创建一个基础的tsconfig.json用于编辑器智能提示：

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

上面的tsconfig.json文件定义了TypeScript的编译选项。虽然上述配置详尽地定义了适配 Bun 运行时环境所需的各项参数，但在 Monorepo 架构中，如果在每个子包内都完整复制这一大段配置，不仅造成代码冗余，后续升级维护也极易导致配置不一致。

为了遵循 DRY (Don't Repeat Yourself) 原则并确保所有子包始终与 Bun 的最新最佳实践保持同步，我们可以引入官方维护的预设配置包 @tsconfig/bun。它将上述所有针对 Bun 优化的编译选项（如 `moduleResolution: "bundler"`、`module: "Preserve"` 等）封装在内，使我们能够通过继承的方式大幅简化项目配置。

首先，安装 @tsconfig/bun 预设配置包：

```bash
bun add -d @tsconfig/bun
```

同时考虑到各个子包都需要依赖@tsconfig/bun，我们需要确保各个子包的"@tsconfig/bun"版本和根目录的版本保持一致，因此在根目录的package.json中添加`"catalog": { "@tsconfig/bun": "1.0.9" }`。并确保在根目录和未来新增的子包的package.json中添加`"devDependencies": {"@tsconfig/bun": "catalog:"}`

```json
"workspaces": {
    "packages": ["packages/*"],
    "catalog": {
      "@types/bun": "1.3.8",
      "typescript": "5.8.2",
      "@tsconfig/bun": "1.0.9"
    }
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "@types/bun": "catalog:",
  }
```
根目录下的tsconfig.json配置内容可以简化为：

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {}
}
```

各子包中的tsconfig.json可以继承@tsconfig/bun的配置，并添加自定义的编译选项。我们以opencode包为例：

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

## 2.4 本章小结

本章我们完成了OpenCode项目开发环境的基础搭建，主要内容包括：

**项目初始化与版本控制**
- 创建项目根目录并初始化Git版本控制系统
- 配置Git用户名和邮箱

**Bun环境配置**
- 安装并验证Bun运行环境
- 配置bunfig.toml进行包管理器设置

**Monorepo架构配置**
- 使用`bun init`初始化项目基础配置
- 配置`workspaces`和`catalog`实现多包管理
- 创建opencode子包的目录结构和package.json
- 安装并配置Turbo构建系统
- 配置TypeScript编译环境，使用@tsconfig/bun预设

现在我们已经具备了完整的Monorepo开发环境，可以开始进行实际的代码开发工作。

## 2.5 下一步

恭喜！你已经完成了 OpenCode 项目的基础环境搭建。现在你拥有了：
- ✅ 完整的 Monorepo 项目结构
- ✅ Bun + Turbo 构建系统
- ✅ TypeScript 类型检查

在下一章中，我们将：
1. 实现 `opencode --version` 命令
2. 创建第一个工具函数
3. 编写单元测试
4. 发布第一个版本

继续阅读：[第三章：CLI实操](./03-cli.md)

## 附录：完整的package.json配置

```json
{
  "$schema": "https://json.schemastore.org/package.json",
  "name": "opencode",
  "description": "AI-powered development tool",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.8",
  "scripts": {
    "dev": "bun run --cwd packages/opencode src/index.ts",
    "typecheck": "bun turbo typecheck"
  },
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "catalog": {
      "typescript": "5.8.2",
      "@types/bun": "1.3.8",
      "@tsconfig/bun": "1.0.9"
    }
  },
  "devDependencies": {
    "@tsconfig/bun": "catalog:",
    "turbo": "2.5.6"
  },
  "dependencies": {
    "typescript": "catalog:"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/anomalyco/opencode"
  },
  "license": "MIT",
  "overrides": {
    "@types/bun": "catalog:"
  },
  "peerDependencies": {
    "typescript": "^5"
  }
}
```

