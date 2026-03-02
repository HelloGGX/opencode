# 第二章：环境搭建与CLI实操

在第一章中，我们深入探讨了AI辅助开发工具的全景架构与设计哲学，对现代化CLI工具的架构设计有了全面的理解。从本章开始，我们将从理论走向实践，通过亲手搭建开发环境来深化对这套架构的理解。本章将以业界成熟的Monorepo项目为蓝本，带领读者完成从零到一的环境搭建工作。

## 2.1 从一个空目录开始

在动手之前，我们先明确一个问题：为什么需要Monorepo？

假设你要开发一个AI编码助手，最简单的做法是创建一个npm包，写一个CLI入口，调用AI API。但很快你会发现：

1. CLI工具、Web界面、VSCode插件都需要相同的AI调用逻辑，但它们在不同的仓库里
2. 每个项目都有自己的package.json，TypeScript版本不一致导致类型错误
3. 修改一个共享函数，需要在3个仓库里分别测试

为了解决这些问题，我们需要Monorepo架构。接下来从一个空目录开始构建。

### 2.1.1 创建项目根目录

首先创建项目目录。我们把项目放在 `~/workspace` 下，这样能与其他项目保持隔离：

```bash
mkdir -p ~/workspace/opencode
cd ~/workspace/opencode
pwd
```

你会看到类似这样的输出：

```bash
$ pwd
/Users/zhangsan/workspace/opencode
```

这个目录就是我们后续所有操作的基准点。

接下来初始化Git版本控制：

```bash
git init
```

现在尝试提交一个空的commit：

```bash
git commit --allow-empty -m "Initial commit"
```

你会看到Git拒绝提交并报错：

```bash
*** Please tell me who you are.

Run

  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"

to set your account's default identity.
```

这是因为Git需要知道是谁提交的代码。配置用户信息：

```bash
git config user.name "张三"
git config user.email "zhangsan@example.com"
```

再次提交，这次成功了：

```bash
$ git commit --allow-empty -m "Initial commit"
[main (root-commit) a1b2c3d] Initial commit
```

现在项目具备了版本控制能力。

## 2.2 为什么选择Bun？

在安装之前，我们先看看Bun解决了什么问题。

传统的Node.js项目通常需要这些工具：
- npm/yarn/pnpm（包管理器）
- tsc（TypeScript编译器）
- jest（测试框架）
- webpack/esbuild（打包工具）

每个工具都有自己的配置文件，版本兼容性也是个头疼的问题。Bun将这些功能集成到一个工具里。

我们用一个例子来验证。创建一个TypeScript文件：

```bash
echo 'const msg: string = "Hello"; console.log(msg)' > test.ts
```

如果用Node.js运行，会报错：

```bash
$ node test.ts
(node:12345) Warning: To load an ES module, set "type": "module"
SyntaxError: Unexpected token ':'
```

因为Node.js不认识TypeScript语法。你需要先用tsc编译：

```bash
$ tsc test.ts  # 生成 test.js
$ node test.js
Hello
```

但如果用Bun运行，可以直接执行：

```bash
$ bun test.ts
Hello
```

这就是Bun的优势：内置TypeScript支持，省去了编译步骤。此外，Bun 的集成设计避免了在同一项目中引入多个工具（如 npm + Jest + esbuild 等）的碎片化复杂度，从而简化了配置和维护。

### 2.2.1 安装并验证Bun

首先检查是否已安装Bun：

```bash
bun --version
```

如果看到版本号（如 `1.3.8`），说明已安装，可以跳过安装步骤。如果看到 `command not found`，需要安装。

在macOS和Linux系统上，Bun的安装非常简单。执行以下命令即可完成安装：

```bash
## Linux/macOS
curl -fsSL https://bun.sh/install | bash
```

这条命令会下载Bun的安装脚本并自动执行。安装脚本会自动将bun可执行文件添加到系统的PATH环境变量中，通常是`~/.bun/bin/bun`。安装完成后，你需要重新加载shell配置或打开新的终端窗口以使PATH变更生效：

```bash
source ~/.zshrc  # 如果使用zsh
source ~/.bashrc # 如果使用bash
```

在Windows上，使用PowerShell执行：

```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

再次运行 `bun --version`，应该能看到版本号了。我们使用 `1.3.8` 版本，如果版本不匹配，可以用 `bun upgrade` 升级。

验证Bun的功能：

```bash
bun --help
```

你会看到Bun提供的所有子命令：install（安装依赖）、run（运行脚本）、test（运行测试）、build（打包代码）等。花些时间熟悉这些命令的功能和用法，会大大提高后续的开发效率。

现在清理测试文件：

```bash
rm test.ts test.js
```

### 2.2.2 配置Bun的安装行为

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
```

这里的 `exact = true` 很重要。它让Bun在安装依赖时锁定确切版本，不写入 `^` 或 `~` 前缀。

我们来验证一下差异。先看默认行为（假设没有exact配置）：

```json
{
  "dependencies": {
    "lodash": "^4.17.21"  // ^ 表示允许小版本更新
  }
}
```

这意味着 `bun install` 可能安装 `4.17.22`、`4.18.0` 等版本，导致不同开发者的环境不一致。

设置 `exact = true` 后：

```json
{
  "dependencies": {
    "lodash": "4.17.21"  // 锁定确切版本
  }
}
```

现在所有人安装的都是 `4.17.21`，确保环境一致。

查看配置是否生效：

```bash
cat bunfig.toml
```

## 2.3 Monorepo架构配置

### 2.3.1 从最简单的配置开始

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
cat package.json
```

你会看到：

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

这个配置可以工作，但有几个问题：

**问题1：没有指定Bun版本**

如果团队成员使用不同版本的Bun，可能导致行为不一致。比如你用1.3.8，同事用1.2.0，某些API可能不兼容。

**问题2：没有工作区配置**

现在只能管理一个包。如果要开发CLI工具、Web界面、VSCode插件，需要创建多个独立的仓库，代码复用困难。

**问题3：没有依赖版本管理**

每个子包都要重复定义 `"typescript": "^5"`，版本不一致时会导致类型错误。

我们逐步解决这些问题。

#### 解决问题1：锁定Bun版本

添加 `packageManager` 字段：

```json
{
  "name": "opencode",
  "packageManager": "bun@1.3.8",
  ...
}
```

现在如果团队成员使用了错误的Bun版本，会收到警告。

#### 解决问题2：配置工作区

添加 `workspaces` 字段：

```json
{
  "workspaces": {
    "packages": ["packages/*"]
  }
}
```

这告诉Bun：`packages/` 目录下的每个子目录都是一个独立的包。

#### 解决问题3：统一依赖版本

Bun提供了"目录版本控制"的特性：catalog ，只需在根目录定义一次依赖版本，子包通过`catalog:`协议引用，实现依赖版本的统一控制和更简洁的依赖树结构，从而提升整个工作区的开发体验与可维护性。

```json
{
  "workspaces": {
    "packages": ["packages/*"],
    "catalog": {
      "@types/bun": "1.3.8",
      "typescript": "5.8.2"
    }
  }
}
```

现在子包可以通过 `"@types/bun": "catalog:"` 引用统一的版本。

#### 最终配置

将所有改进整合到一起：

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
```

查看配置：

```bash
cat package.json
```

让我们理解几个关键字段：

| 字段 | 说明 |
|------|------|
| `$schema` | 指向JSON Schema，IDE可据此提供自动补全 |
| `packageManager` | 指定Bun版本为1.3.8 |
| `workspaces.packages` | `packages/*` 表示packages目录下所有子目录都是独立包 |
| `workspaces.catalog` | 定义共享依赖版本，子包通过 `catalog:` 引用 |
| `private: true` | 防止意外发布到npm |

现在我们有了一个支持Monorepo的配置。同时为了进一步确保依赖版本的一致性，Bun提供了`overrides`配置，可以强制所有子包使用根目录定义的特定依赖版本：

```json
"overrides": {
  "@types/bun": "catalog:"
}
```

这个配置的作用是：即使某个子包写了 `"@types/bun": "1.2.0"`，Bun也会强制使用catalog中定义的 `1.3.8` 版本。

我们来验证一下。先查看.gitignore文件：

```bash
cat .gitignore
```

`bun init` 已经创建了完善的.gitignore，包含了node_modules、dist、.env等常见忽略项。

现在Monorepo的基础配置完成了。通过catalog和overrides，我们确保了：
- 所有子包使用相同的TypeScript版本
- 所有子包使用相同的Bun类型定义
- 避免了版本冲突和重复安装

### 2.3.2 创建第一个子包

现在创建opencode子包。这是我们实现CLI命令的核心包：

```bash
mkdir -p packages/opencode/src
touch packages/opencode/src/index.ts
```

创建子包的package.json：

```bash
cat > packages/opencode/package.json << 'EOF'
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

注意几个关键点：

- `"private": true`：这个设置非常重要。opencode包只在该项目中内部使用，不会被发布到npm仓库。
- `"bin"`字段定义了CLI入口点。`"opencode": "./bin/opencode"`告诉包管理器，当用户安装这个包时，需要创建一个名为`opencode`的命令，指向bin目录下的opencode脚本文件。
- `"exports"`字段定义了包的导出路径。`"./*": "./src/*.ts"`告诉包管理器，当用户导入opencode时，应该从src目录下的对应文件中导出。
- `"devDependencies"`使用了`"catalog:"`前缀。这意味着这些依赖的版本不是直接写在子包中，而是从根目录的catalog中读取。这种方式确保了opencode包的@types/bun、typescript依赖都统一使用根目录的版本号，避免了版本冲突和不一致的问题。

查看配置：

```bash
cat packages/opencode/package.json
```

### 2.3.3 配置 Turbo 构建系统

随着子包数量增加，构建任务会变得复杂。假设我们有3个包：

```
packages/
  util/      # 工具函数
  plugin/    # 依赖util
  opencode/  # 依赖plugin和util
```

如果手动管理构建顺序：

```bash
cd packages/util && bun run build
cd packages/plugin && bun run build
cd packages/opencode && bun run build
```

这有几个问题：

1. **顺序错误会导致失败**：如果先构建opencode，会因为找不到plugin的构建产物而失败
2. **无法并行**：util和plugin可以并行构建，但手动管理很难实现
3. **重复构建**：修改util后，需要重新构建所有依赖它的包

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
```

理解这个配置：

**typecheck任务**：
```json
"typecheck": {}
```
空配置表示在所有子包中并行运行typecheck脚本，没有依赖关系。

**build任务**：
```json
"build": {
  "dependsOn": ["^build"],
  "outputs": ["dist/**"]
}
```
- `"dependsOn": ["^build"]`：`^` 表示依赖上游包的build任务。比如opencode依赖plugin，那么会先构建plugin
- `"outputs": ["dist/**"]`：缓存dist目录，如果文件没变，直接复用缓存

**opencode#test任务**：
```json
"opencode#test": {
  "dependsOn": ["^build"],
  "outputs": []
}
```
- `opencode#test`：只在opencode包中运行test
- `"dependsOn": ["^build"]`：测试前先构建所有依赖
- `"outputs": []`：测试不产生文件，不需要缓存


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

```bash
cat tsconfig.json
```

你会看到很多配置项：`moduleResolution: "bundler"`、`module: "Preserve"`、`strict: true` 等。这些都是Bun推荐的配置。

但有个问题：如果每个子包都复制这些配置，会导致：
1. 配置重复，维护困难
2. 版本不一致，容易出错

为了遵循 DRY (Don't Repeat Yourself) 原则并确保所有子包始终与 Bun 的最新最佳实践保持同步，更好的做法是使用官方预设:`@tsconfig/bun`。它将上述所有针对 Bun 优化的编译选项（如 `moduleResolution: "bundler"`、`module: "Preserve"` 等）封装在内，使我们能够通过继承的方式大幅简化项目配置。


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
  "@types/bun": "catalog:"
}
```

你看，根目录下的tsconfig.json配置内容可以简化为：

```bash
cat > tsconfig.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {}
}
EOF
```

现在配置只有3行，所有Bun优化的选项都继承自 `@tsconfig/bun`。

接着我们为opencode子包创建tsconfig.json：

```bash
cat > packages/opencode/tsconfig.json << 'EOF'
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {}
}
EOF
```

子包也继承了相同的配置，确保类型检查行为一致。后续章节会根据需要添加自定义配置。

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

下一章我们将实现 `opencode --version` 命令，创建第一个工具函数，并编写单元测试。

继续阅读：[第三章：CLI实操](./03-cli.md)

## 附录：完整配置文件

如果你想直接使用最终配置，可以参考以下内容。

**根目录package.json**：

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
    "packages": ["packages/*"],
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

**bunfig.toml**：

```toml
[install]
registry = "https://registry.npmjs.org"
exact = true
```

**turbo.json**：

```json
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
```

**tsconfig.json**：

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {}
}
```

