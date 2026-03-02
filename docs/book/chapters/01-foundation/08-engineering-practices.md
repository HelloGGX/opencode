# 第 8 章：工程化实践

在现代软件开发中，代码质量保证和规范的开发流程是项目成功的关键因素。本章将介绍如何配置代码格式化工具、Git钩子管理以及持续集成系统，确保团队协作时代码风格的一致性和项目交付质量。

## 8.1 代码风格统一：Prettier + EditorConfig

### 8.1.1 配置Prettier代码格式化工具

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
  }
```

同时，创建`.prettierignore`文件来指定不需要格式化的文件：

```bash
cat > .prettierignore << 'EOF'
dist**
EOF
```

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

### 8.1.2 配置EditorConfig

虽然 automated checks 能确保代码逻辑无误，但不同的操作系统（Windows vs macOS）和编辑器习惯（Tab vs Space）往往会导致许多看不见的"脏"diff（例如换行符差异）。为了让所有开发者在打开编辑器的那一刻就拥有完全一致的编码体验，我们需要配置 EditorConfig。

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

这个EditorConfig配置确保了：
- 所有文件使用 UTF-8 编码
- 文件末尾有一个换行符
- 使用 Unix 风格的换行符（LF）
- 使用 2 个空格缩进
- 每行最多 80 个字符

EditorConfig 是一种编辑器无关的标准，大多数现代 IDE（如 VSCode、WebStorm、IntelliJ IDEA）和编辑器插件都会自动读取并应用这些设置。

## 8.2 Git工作流守卫：Husky

### 8.2.1 安装与初始化Husky

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
同时`bun run husky init` 会自动在 package.json 的 scripts 中添加 `"prepare": "husky"` 命令，无需手动添加。这个命令会在依赖安装完成后自动初始化 Husky 并确保 Git Hooks 生效。

```json
{
  "scripts": {
    "prepare": "husky"
  }
}
```

### 8.2.2 配置pre-push钩子

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

这个脚本会在 `git push` 命令执行前自动运行，确保：
1. 开发者使用的 Bun 版本与项目要求的版本兼容
2. 推送前的代码通过了 TypeScript 类型检查
3. 所有开发者使用相同的开发环境，避免因版本差异导致的问题

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

## 8.3 持续集成：GitHub Actions

现代软件开发离不开持续集成和持续部署（CI/CD）。本项目采用GitHub Actions作为CI/CD平台，配置了多个专业化的工作流来确保代码质量和自动化部署。

### 8.3.1 项目CI/CD架构概览

项目采用了多工作流架构，每个工作流负责不同的职责：

- **测试工作流** (`test.yml`): 负责运行单元测试和端到端测试
- **类型检查工作流** (`typecheck.yml`): 确保TypeScript代码类型安全
- **发布工作流** (`publish.yml`): 自动化版本管理和包发布
- **OpenCode集成工作流** (`opencode.yml`): 集成AI辅助开发工具

下面我们重点讲解其中2个工作流：测试工作流和类型检查工作流。后续的工作流我们会在需要的时候再进行补充和迭代。

### 8.3.2 创建GitHub Actions配置目录

首先创建必要的工作流目录结构：

```bash
mkdir -p .github/workflows
mkdir -p .github/actions/setup-bun
```

### 8.3.3 自定义setup-bun Action

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

可以看到，我们将 using 设置为 "composite"，这告诉 GitHub Runner："嘿，不要去找 index.js，请直接执行我定义的 steps。"

在构建过程中，依赖安装往往是最耗时的环节。为了提升性能，我们必须引入缓存机制。但是，直接使用普通的 Cache Action 有时会遇到命中率低或配置繁琐的问题。

在这里，我们使用了 GitHub Actions 提供的标准缓存机制。通过引入 actions/cache@v4，可以将依赖缓存到 GitHub 的缓存服务中，在后续构建时自动恢复：

```bash
name: "Setup Bun"
description: "Setup Bun with caching and install dependencies"
runs:
  using: "composite" # 关键：声明这是一个组合式 Action
  steps:
    - name: Mount Bun Cache
      uses: actions/cache@v4
      with:
        key: ${{ github.repository }}-bun-cache
        path: ~/.bun
```

这样做的本质，是将易失性的构建环境与持久化的依赖存储分离开来。无论 Runner 如何销毁重建，~/.bun 目录下的内容都能像"快照"一样被恢复，从而极大缩短了 bun install 的时间。

环境准备好之后，接下来的工作就顺理成章了。我们需要安装 Bun，并执行依赖安装。

这里有一个细节值得注意：在设置 Bun 版本时，我们并没有硬编码版本号，而是指定了 bun-version-file: package.json,它表示从 package.json 中读取 Bun 版本。

```bash
name: "Setup Bun"
description: "Setup Bun with caching and install dependencies"
runs:
  using: "composite" # 关键：声明这是一个组合式 Action
  steps:
    - name: Mount Bun Cache
      uses: actions/cache@v4
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
      uses: actions/cache@v4
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

### 8.3.4 多平台测试工作流

在上一节中，我们封装了通用的 setup-bun。现在，我们面临的第一个问题是：如何确保我们的代码在不同操作系统下表现一致？

通常情况下，开发者习惯在 macOS 或 Linux 本地环境中开发，但这往往会掩盖 Windows 环境下特有的路径分隔符、换行符或 shell 差异问题。

如果分别为 Linux 和 Windows 编写独立的 Workflow，不仅会引入大量重复配置，还会在后续维护中不断放大改动成本，这显然违背了 DRY 原则。为了解决这一问题，GitHub Actions 提供了 matrix（矩阵）策略，使我们能够在保留统一执行流程的前提下，对不同运行环境进行参数化配置。

在这种模式下，测试流程本身只需要定义一次，而操作系统、运行节点、依赖安装方式以及具体的测试命令等平台差异，则通过 matrix 作为配置项传入。GitHub Actions 会基于 matrix 中的每一组配置，自动生成并执行对应的测试任务。

接下来我们创建完整的 test.yml 配置文件：

```bash
# 创建 test.yml 配置文件
cat > .github/workflows/test.yml << 'EOF'
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
      fail-fast: false # 关键：避免单点失败导致整个矩阵立即终止，我们需要看到所有平台的测试结果
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

      - name: Install Playwright dependencies
        run: ${{ matrix.settings.playwright }}

      - name: Run tests
        run: |
          cd ${{ matrix.settings.workdir }}
          ${{ matrix.settings.command }}
EOF
```

首先是 **触发机制**。我们定义了三种触发时机：
* 当有代码推送到 `dev` 分支时；
* 当有针对 `dev` 分支的 Pull Request 被创建或更新时；
* 以及通过 `workflow_dispatch` 允许手动触发。

这种设计体现了 **"尽早发现"** 的原则。在 Pull Request 阶段就拦截错误，可以避免污染主分支的稳定性，将问题解决在合并之前。

在该 matrix 中，我们定义了两个测试配置：linux 和 windows。它们共享同一套测试流程，但在运行节点（host）、Playwright 的安装方式、工作目录以及最终执行的测试命令上各自独立，从而准确反映不同操作系统下的真实运行环境。

这里有三个细节值得注意：
1. 我们将 fail-fast 设置为 false。这是因为在 CI 环境中，Windows 任务通常比 Linux 慢。如果 Linux 任务失败了，我们通常仍希望看到 Windows 任务的结果，以便判断这是否是一个特定平台的 Bug，还是通用逻辑的错误。
2. Linux 环境下运行 Playwright 通常需要额外安装系统依赖（--with-deps），而 Windows 环境通常不需要或已预置，Matrix 让我们能轻松处理这种差异。
3. 我们在linux测试命令前添加了 git config --global user.email 和 git config --global user.name，这是因为在执行bun turbo test时，子包中可能存在git操作，Git 要求必须配置 user.email 和 user.name 否则会报错, 在后续的子包的单元测试的编写中，我们会用到这些配置。

定义好了矩阵，下一步是将这些配置映射到真实的虚拟机上。

通过将 runs-on 设置为 ${{ matrix.settings.host }}，GitHub Actions 会为矩阵中的每组配置创建一个独立的 Job，并在指定的操作系统（如 Linux 或 Windows）上运行相同的测试步骤。这样，Linux 和 Windows 的差异就直接由运行环境来处理，而不需要在代码中用 if-else 分支来手动区分。

但这里更值得玩味的是 `defaults.run.shell: bash`。我们知道，Windows 的原生 Shell 是 PowerShell 或 CMD，而 Linux 是 Bash。如果任由默认行为发生，我们在编写后续的 steps 时，就需要区分这两者的差异。通过设置 `shell: bash`，强制 GitHub Actions 在 Windows 环境中也使用 Git Bash 来执行命令, 这使得我们可以放心地在 steps 中使用 rm -rf、export 等标准 Linux 命令，而无需为 Windows 编写繁琐的 PowerShell 替代方案。

接下来我们看看steps：

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

后续步骤涉及测试库的安装和执行测试命令。根据不同平台的配置，我们需要在不同的工作目录下执行命令。从前面可以看出，我们使用 playwright 作为测试库，它是微软开发的现代化端到端测试工具，用于测试 Web 应用程序在不同浏览器中的行为。

### 8.3.5 类型检查工作流

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

首先是 **运行环境**。这里我们指定了 `runs-on: ubuntu-latest`。这是 GitHub Actions 提供的标准 Linux 运行环境，由于类型检查（Typecheck）通常是 CPU 密集型任务，标准运行环境已能满足大多数基础构建需求。如果后续项目规模扩大，我们也可以轻松切换到更高性能的 Runner。

整个过程分为三步：
1. **检出代码**：这是所有 CI 流程的基石。
2. **环境准备**：通过 `setup-bun` 初始化运行时环境。这里假设我们已经封装了一个复用的 Action 来统一管理 Bun 的版本和配置，这符合我们在软件工程中推崇的 DRY（Don't Repeat Yourself）原则。
3. **执行检查**：运行 `bun typecheck`。这会触发项目中的类型检查脚本，确保代码符合 TypeScript 类型规范。

需要注意的是，这里的 `bun typecheck` 通常是在 `package.json` 中定义的脚本，其底层往往调用了 `"typecheck": "tsc --noEmit"`。`--noEmit` 标志非常关键，它告诉编译器："我们只需要检查类型是否正确，不需要输出任何编译后的文件。"

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

## 8.4 本章小结

本章我们完成了工程化实践的配置，建立了一套完整的代码质量和持续集成体系。

**代码质量保障**

1. **Prettier**：配置了统一的代码格式化规则，通过 `bun run format` 自动格式化所有代码
2. **EditorConfig**：确保所有开发者在打开编辑器时就拥有一致的编码环境
3. **Husky**：通过 pre-push 钩子在代码推送前自动进行类型检查

**持续集成体系**

4. **自定义 setup-bun Action**：封装了 Bun 环境准备、缓存挂载和依赖安装的完整流程
5. **多平台测试工作流**：通过 Matrix 策略在 Linux 和 Windows 上并行运行测试
6. **类型检查工作流**：独立验证 TypeScript 类型安全

这套体系确保了：
- 代码风格一致性：团队成员提交的都是格式化后的统一风格代码
- 版本兼容性：推送前自动检查 Bun 版本和类型检查
- 跨平台兼容：在多个操作系统上验证代码正确性
- 自动化流程：所有质量检查都自动执行，无需人工干预

下一章我们将学习配置系统基础，掌握多层级配置的管理方法。
