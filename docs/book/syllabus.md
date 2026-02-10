《从零构建 OpenCode：AI 编码助手》
一本资深工程师视角的实战指南，还原敏捷开发迭代过程

## 📖 阅读指南

### 小节结构

- **子节目标**：具体输出，明确每节要实现的功能
- **决策还原**：Why & Trade-off，理解技术选型背后的思考
- **代码伴读**：贴合实际文件，Context + Problem + Insight
- **练习**：读者在 fork 仓库中实现，动手实践
- **技术难点** & **企业级价值**：独立总结，提炼关键点

### 代码库信息

- **项目版本**：1.1.39
- **代码行数**：50,000+ 行
- **测试文件**：40+ 个
- **支持语言**：14 种（i18n）
- **默认分支**：dev

### 学习路径

本书共 14 章，分为 4 个阶段：
- **基础篇**（第 1-5 章）：环境搭建、配置系统、AI 集成、工具系统、权限控制
- **核心篇**（第 6-10 章）：会话处理、TUI 界面、LSP 集成、MCP 协议、多端部署
- **工程篇**（第 11-13 章）：测试体系、国际化、CI/CD 自动化
- **优化篇**（第 14 章）：性能优化实战

---

## 第一部分：基础篇

### 第1章：Monorepo环境与CLI骨架（Iteration 1: Foundation）

**迭代目标**：构建完整 monorepo 环境，实现基础 CLI 框架。输出：可安装依赖、运行 28 个命令的开发环境。

---

#### 1.0 全景架构与设计哲学

在深入代码之前，我们需要理解 OpenCode 的整体架构和设计理念。这将帮助你建立全局视角，理解每个模块的定位和相互关系。

##### 1.0.1 OpenCode 架构全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenCode 架构全景                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    用户界面层 (UI Layer)                      │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  TUI (Terminal)  │  Web App  │  Desktop  │  VSCode Extension │   │
│  │   (OpenTUI)      │  (React)  │  (Tauri)  │   (Extension API) │   │
│  └──────────┬───────────────┬───────────┬──────────────┬────────┘   │
│             │               │           │              │            │
│  ┌──────────┴───────────────┴───────────┴──────────────┴────────┐   │
│  │                    通信层 (Communication)                      │   │
│  ├───────────────────────────────────────────────────────────────┤   │
│  │  HTTP Server  │  RPC (Worker)  │  SSE Events  │  WebSocket   │   │
│  └──────────┬───────────────┬───────────┬──────────────┬────────┘   │
│             │               │           │              │            │
│  ┌──────────┴───────────────┴───────────┴──────────────┴────────┐   │
│  │                    核心业务层 (Core Layer)                     │   │
│  ├───────────────────────────────────────────────────────────────┤   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │
│  │  │   Session   │  │   Agent     │  │  Permission │           │   │
│  │  │   会话管理   │  │   代理系统   │  │   权限控制   │           │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │   │
│  │         │                │                │                   │   │
│  │  ┌──────┴────────────────┴────────────────┴──────┐           │   │
│  │  │              EventBus (事件总线)                │           │   │
│  │  │        30+ 事件类型，模块间解耦通信              │           │   │
│  │  └──────┬────────────────┬────────────────┬──────┘           │   │
│  │         │                │                │                   │   │
│  │  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐           │   │
│  │  │   Message   │  │    Tool     │  │   Question  │           │   │
│  │  │   消息处理   │  │   工具系统   │  │   交互问答   │           │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                             │                                       │
│  ┌──────────────────────────┴───────────────────────────────────┐   │
│  │                    基础设施层 (Infrastructure)                 │   │
│  ├───────────────────────────────────────────────────────────────┤   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │
│  │  │   Storage   │  │  FileWatch  │  │   Snapshot  │           │   │
│  │  │   存储抽象   │  │   文件监控   │  │   快照系统   │           │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │
│  │  │   Config    │  │   Instance  │  │  Scheduler  │           │   │
│  │  │   配置系统   │  │   实例管理   │  │   定时任务   │           │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                             │                                       │
│  ┌──────────────────────────┴───────────────────────────────────┐   │
│  │                    扩展层 (Extension Layer)                    │   │
│  ├───────────────────────────────────────────────────────────────┤   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │
│  │  │     LSP     │  │     MCP     │  │   Plugin    │           │   │
│  │  │  语言服务器  │  │  模型上下文  │  │   插件系统   │           │   │
│  │  │  (15+语言)  │  │   协议集成   │  │  (工具/认证) │           │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │   │
│  │  │    Skill    │  │   Command   │  │  Worktree   │           │   │
│  │  │   技能系统   │  │   命令系统   │  │  沙盒管理   │           │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘           │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                             │                                       │
│  ┌──────────────────────────┴───────────────────────────────────┐   │
│  │                    AI 提供商层 (AI Provider)                   │   │
│  ├───────────────────────────────────────────────────────────────┤   │
│  │                                                                 │   │
│  │  ┌──────────────────────────────────────────────────────────┐ │   │
│  │  │              Provider 抽象层 (@ai-sdk)                     │ │   │
│  │  ├──────────────────────────────────────────────────────────┤ │   │
│  │  │  OpenAI │ Anthropic │ Google │ Mistral │ ... (20+ 提供商) │ │   │
│  │  └──────────────────────────────────────────────────────────┘ │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

##### 1.0.2 核心设计哲学

**1. 事件驱动架构 (Event-Driven Architecture)**

OpenCode 的核心是 EventBus，所有模块通过事件解耦：

```typescript
// 发布者：不关心谁在监听
Bus.publish(Session.Event.Created, { sessionID, info })

// 订阅者：不关心谁发布
Bus.subscribe(Session.Event.Created, async (evt) => {
  await syncToCloud(evt.properties.info)
})
```

**设计理念**：
- ✅ 模块间松耦合，易于扩展
- ✅ 异步通信，不阻塞主流程
- ✅ 可观测性强，所有事件可追踪
- ⚠️ 调试复杂度增加（事件链路追踪）

**2. 实例隔离 (Instance Isolation)**

每个项目目录对应一个独立实例，状态完全隔离：

```typescript
// 使用 Symbol 确保状态隔离
const state = Instance.state(() => ({
  sessions: new Map(),
  tools: []
}))

// 不同项目的状态互不影响
// /project-a → Instance A
// /project-b → Instance B
```

**设计理念**：
- ✅ 多项目并行开发
- ✅ 状态不污染
- ✅ 资源自动清理（dispose）
- ⚠️ 内存开销增加

**3. 类型安全优先 (Type-Safe First)**

使用 Zod 实现运行时类型验证 + TypeScript 编译时检查：

```typescript
// 配置定义
export const Config = z.object({
  model: z.string().optional(),
  agent: z.record(z.string(), AgentConfig).optional()
})

// 自动推导类型
type Config = z.infer<typeof Config>

// 运行时验证
const config = Config.parse(data) // 类型错误会抛出异常
```

**设计理念**：
- ✅ 编译时 + 运行时双重保障
- ✅ 自动生成 TypeScript 类型
- ✅ 配置错误早发现
- ⚠️ 学习成本（Zod API）

**4. 渐进式增强 (Progressive Enhancement)**

从 MVP 到完整系统，每个迭代都可独立运行：

```
Iteration 1: CLI 骨架 → 可运行 28 个命令
Iteration 2: 配置系统 → 可加载配置文件
Iteration 3: AI 集成 → 可调用 LLM
Iteration 4: 工具系统 → 可执行文件操作
...
```

**设计理念**：
- ✅ 快速验证核心功能
- ✅ 降低开发风险
- ✅ 易于理解和学习
- ⚠️ 需要良好的架构规划

**5. 约定优于配置 (Convention over Configuration)**

合理的默认值 + 灵活的配置：

```typescript
// 默认配置
const defaults = {
  model: "anthropic/claude-3-5-sonnet-20241022",
  agent: "build",
  permission: { edit: { "*": "ask" } }
}

// 用户只需配置差异
// opencode.md
---
model: "openai/gpt-4"
---
```

**设计理念**：
- ✅ 开箱即用
- ✅ 降低配置复杂度
- ✅ 保持灵活性
- ⚠️ 默认值需要精心设计

##### 1.0.3 技术栈选型理由

| 技术 | 选型理由 | Trade-off |
|------|---------|-----------|
| **Bun** | 3-5x 快于 npm，原生 TypeScript | 生态成熟度 < Node.js |
| **Turbo** | 增量构建，10x 提升大型项目 | 配置复杂度增加 |
| **Zod** | 类型安全，运行时验证 | 学习成本 |
| **@ai-sdk** | 统一接口，20+ 提供商 | 抽象层性能损耗 |
| **OpenTUI** | 终端原生组件，高性能 | 学习曲线陡峭 |
| **SolidJS** | 细粒度响应式，小体积 | 生态 < React |
| **Playwright** | 跨浏览器 E2E 测试 | 测试速度较慢 |

##### 1.0.4 数据流向图

```
用户输入 (Prompt)
    ↓
TUI/Web/VSCode
    ↓
HTTP/RPC 通信层
    ↓
Session.prompt()
    ↓
Permission.ask() ← 权限检查
    ↓
LLM.stream() ← AI 提供商
    ↓
Tool.execute() ← 工具调用
    ↓
Bus.publish() ← 事件通知
    ↓
Storage.write() ← 持久化
    ↓
SSE 推送 → 前端更新
```

##### 1.0.5 前置知识清单

在开始学习本书之前，你需要掌握以下基础知识：

**必备知识（⭐⭐⭐）：**
- ✅ **TypeScript 基础**
  - 类型注解、接口、泛型
  - 类型推断、联合类型、交叉类型
  - `async/await`、Promise
  - 命名空间（namespace）
  
- ✅ **Git 基础**
  - `clone`、`commit`、`push`、`pull`
  - 分支管理（`branch`、`checkout`）
  - 基本概念：工作区、暂存区、仓库
  
- ✅ **终端操作**
  - 基本命令：`cd`、`ls`、`mkdir`、`rm`
  - 环境变量设置
  - 包管理器使用（npm/yarn/bun）

**推荐知识（⭐⭐）：**
- ⭕ **Node.js 生态**
  - package.json 配置
  - 模块系统（ESM/CommonJS）
  - 常用工具链（ESLint、Prettier）
  
- ⭕ **设计模式**
  - 观察者模式（EventBus）
  - 工厂模式（Provider）
  - 单例模式（Instance）
  
- ⭕ **函数式编程**
  - 高阶函数、闭包
  - 不可变数据
  - 纯函数

**加分项（⭐）：**
- 🔹 Monorepo 经验（Turborepo/Nx）
- 🔹 LSP 协议了解
- 🔹 终端 UI 开发（Ink/Blessed）
- 🔹 AI/LLM 基础知识

**自测题**：

```typescript
// 1. 你能理解这段代码吗？
export function state<T>(
  init: () => T | Promise<T>,
  dispose?: (state: T) => void
): () => T {
  const key = Symbol()
  return () => {
    const instance = getCurrentInstance()
    if (!instance.state.has(key)) {
      const value = init()
      instance.state.set(key, value)
      if (dispose) instance.disposers.push(() => dispose(value))
    }
    return instance.state.get(key)
  }
}

// 2. 你能解释 Symbol 的作用吗？
// 3. 你能解释泛型 <T> 的作用吗？
// 4. 你能解释闭包在这里的应用吗？
```

如果你对以上代码感到困惑，建议先学习 TypeScript 进阶知识。

##### 1.0.6 学习路线图

```
第 1 周：环境搭建 + 配置系统（第 1-2 章）
    ↓
第 2-3 周：AI 集成 + 工具系统（第 3-5 章）
    ↓
第 4-5 周：会话处理 + TUI 界面（第 6-8 章）
    ↓
第 6-7 周：LSP + MCP 集成（第 9-11 章）
    ↓
第 8 周：测试 + 国际化 + CI/CD（第 12-14 章）
    ↓
第 9-12 周：完整项目实践 + 性能优化（第 15 章）
```

**里程碑验证**：
- ✅ Week 2: 能运行 `opencode --version`
- ✅ Week 4: 能创建会话并发送消息
- ✅ Week 6: 能使用 TUI 界面交互
- ✅ Week 8: 能运行完整测试套件
- ✅ Week 12: 能部署生产环境

##### 1.0.7 从零开始：完整项目初始化

**步骤 1：克隆仓库**

```bash
# 克隆 OpenCode 仓库
git clone https://github.com/anomalyco/opencode.git
cd opencode

# 查看项目结构
ls -la
# 输出：
# .git/
# packages/
# package.json
# turbo.json
# ...
```

**步骤 2：安装 Bun**

```bash
# macOS/Linux
curl -fsSL https://bun.sh/install | bash

# Windows (WSL)
curl -fsSL https://bun.sh/install | bash

# 验证安装
bun --version
# 输出：1.3.5
```

**步骤 3：安装依赖**

```bash
# 安装所有 workspace 依赖
bun install

# 等待安装完成（约 1-2 分钟）
# 输出：
# + 1234 packages installed [12.34s]
```

**步骤 4：构建项目**

```bash
# 使用 Turbo 构建所有包
bun run build

# 输出：
# • packages/opencode:build: cache hit, replaying logs
# • packages/ui:build: cache hit, replaying logs
# ...
```

**步骤 5：验证安装**

```bash
# 运行 OpenCode CLI
bun run packages/opencode/src/index.ts --version

# 输出：
# opencode version 1.1.39

# 查看可用命令
bun run packages/opencode/src/index.ts --help

# 输出：
# Commands:
#   acp              Start ACP server
#   mcp              Manage MCP servers
#   tui              Launch TUI interface
#   session          Manage sessions
#   ...（28 个命令）
```

**步骤 6：配置开发环境**

```bash
# 创建配置文件
mkdir -p .opencode
cat > .opencode/opencode.md << 'EOF'
---
model: "anthropic/claude-3-5-sonnet-20241022"
provider:
  anthropic:
    apiKey: "your-api-key-here"
---
EOF

# 设置环境变量（可选）
export ANTHROPIC_API_KEY="your-api-key-here"
```

**步骤 7：运行第一个会话**

```bash
# 启动 TUI 界面
bun run packages/opencode/src/index.ts tui

# 或者使用命令行模式
bun run packages/opencode/src/index.ts session new --agent build
# 输出：Session created: sess_abc123

bun run packages/opencode/src/index.ts session prompt sess_abc123 "Hello, OpenCode!"
# 输出：AI 响应...
```

**常见问题排查**：

```bash
# 问题 1：Bun 安装失败
# 解决：使用 npm 安装
npm install -g bun

# 问题 2：依赖安装失败
# 解决：清理缓存重试
rm -rf node_modules bun.lock
bun install

# 问题 3：构建失败
# 解决：检查 Node.js 版本（需要 >= 18）
node --version

# 问题 4：API 密钥未配置
# 解决：设置环境变量
export ANTHROPIC_API_KEY="sk-ant-..."
```

**开发工具推荐**：

```bash
# VSCode 扩展
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension oven.bun-vscode

# 配置 VSCode
cat > .vscode/settings.json << 'EOF'
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "typescript.tsdk": "node_modules/typescript/lib"
}
EOF
```

现在你已经完成了开发环境搭建，可以开始深入学习各个模块的实现细节了！

---

#### 1.1 Monorepo 架构设计
**贴合文件**：`turbo.json`、`package.json`、`packages/*/package.json`

**决策还原**：
- **Why Turbo**：增量构建、任务依赖管理、缓存优化
- **Trade-off**：配置复杂度 vs. 构建效率（大型项目提升 10x）
- **Why Bun workspaces**：原生 TypeScript、快速安装、workspace 协议

**代码伴读**：
```json
// turbo.json - 任务依赖配置
{
  "tasks": {
    "build": {
      // ^ 表示依赖上游包（dependencies）先构建
      // 确保构建顺序正确，避免依赖未构建的包
      "dependsOn": ["^build"],
      
      // 指定构建输出目录，用于缓存判断
      // Turbo 会检查这些文件是否变化来决定是否需要重新构建
      "outputs": ["dist/**"]
    }
  }
}

// package.json - workspace 配置
{
  "workspaces": {
    // 定义 workspace 包的位置
    // packages/* 匹配 packages 下的所有一级目录
    // packages/console/* 匹配 console 下的所有子包
    "packages": ["packages/*", "packages/console/*"]
  }
}
```

**练习**：添加新的 workspace 包，配置 Turbo 任务依赖。

#### 1.2 Bun 生态系统
**贴合文件**：`bunfig.toml`、`package.json`

**决策还原**：
- **Why Bun**：3-5x 快于 npm/yarn、原生 TypeScript、内置测试框架
- **Trade-off**：生态成熟度 vs. 性能提升
- **安全机制**：trustedDependencies 防止恶意脚本

**代码伴读**：
```json
// package.json - Bun 特性
{
  // 锁定包管理器版本，确保团队使用相同版本
  // Bun 会检查版本是否匹配，不匹配时会报错
  "packageManager": "bun@1.3.5",
  
  // 安全机制：只允许列出的包执行 postinstall 脚本
  // 防止恶意包在安装时执行任意代码（供应链攻击）
  // 未列出的包的 postinstall 脚本会被忽略
  "trustedDependencies": [
    "esbuild",        // 需要下载平台特定的二进制文件
    "tree-sitter",    // 需要编译原生模块
    "web-tree-sitter" // 需要下载 WASM 文件
  ]
}
```

**Insight**：trustedDependencies 是 Bun 的安全特性，只允许明确信任的包执行 postinstall 脚本，有效防止供应链攻击。

#### 1.3 Yargs CLI 框架
**贴合文件**：`packages/opencode/src/index.ts`

**决策还原**：
- **Why Yargs**：成熟稳定、中间件支持、自动生成帮助
- **Trade-off**：vs. commander.js（Yargs 更灵活，commander 更简洁）

**代码伴读**：
```typescript
// src/index.ts - 28 个命令注册
const cli = yargs(hideBin(process.argv))
  // 中间件：在所有命令执行前运行
  .middleware(async (opts) => {
    // 初始化日志系统
    await Log.init({ /* ... */ })
    
    // 设置环境变量，标识当前是 Agent 模式
    // 某些功能会根据此变量调整行为
    process.env.AGENT = "1"
  })
  // 注册命令：每个命令对应一个功能模块
  .command(AcpCommand)       // ACP 协议服务器（Agent Communication Protocol）
  .command(McpCommand)       // MCP 服务器管理（Model Context Protocol）
  .command(TuiThreadCommand) // TUI 主界面（Terminal User Interface）
  // ... 还有 25 个命令（session、project、tool 等）
```

**练习**：实现自定义命令，添加中间件处理。

#### 1.4 开发工具链
**贴合文件**：`tsconfig.json`、`.prettierignore`、`.husky/`

**代码伴读**：
```json
// tsconfig.json - TypeScript 配置
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler"
  }
}
```

**练习**：配置 Husky pre-commit hook，添加代码格式化检查。

#### 1.5 补丁管理机制
**贴合文件**：`patches/ghostty-web@0.3.0.patch`

**技术难点**：patchedDependencies 处理流程
```json
{
  "patchedDependencies": {
    "ghostty-web@0.3.0": "patches/ghostty-web@0.3.0.patch"
  }
}
```

**Insight**：Bun 自动应用补丁，修复上游依赖 bug 而不等待官方发布。

**企业级价值**：
- 团队协作标准化（统一工具链）
- CI/CD 基础（可复现构建）
- 依赖安全管理（trustedDependencies）

**本章小结**：
本章构建了完整的 Monorepo 开发环境，使用 Turbo 实现增量构建和任务依赖管理，使用 Bun 提供高性能的包管理和原生 TypeScript 支持。通过 Yargs 实现了 28 个 CLI 命令的注册和中间件处理。补丁管理机制确保了依赖的安全性和可维护性。这些基础设施为后续的配置系统、AI 集成和工具开发提供了坚实的基础。

### 第2章：配置系统与项目实例（Iteration 2: Configuration）

**迭代目标**：实现配置加载、项目实例管理、事件驱动架构。输出：可配置的 AI 代理系统。

#### 2.1 配置系统架构
**贴合文件**：`src/config/config.ts`、`src/config/markdown.ts`

**决策还原**：
- **Why Zod**：类型安全、运行时验证、自动生成 TypeScript 类型
- **Trade-off**：vs. JSON Schema（Zod 更适合 TypeScript 生态）

**代码伴读**：
```typescript
// src/config/config.ts - Zod 配置验证
export const Config = z.object({
  // 默认模型 ID（如 "anthropic/claude-3-5-sonnet-20241022"）
  model: z.string().optional(),
  
  // Agent 配置：键为 agent 名称，值为配置对象
  // 例如：{ "build": { permission: {...} } }
  agent: z.record(z.string(), AgentConfig).optional(),
  
  // 权限配置：键为权限类型，值为规则
  // 例如：{ "edit": { "*.ts": "allow" } }
  permission: z.record(z.string(), z.any()).optional(),
  
  // MCP 服务器配置：键为服务器名称，值为连接配置
  // 例如：{ "filesystem": { command: "npx", args: [...] } }
  mcp: z.record(z.string(), McpConfig).optional(),
  
  // AI 提供商配置：键为提供商 ID，值为 API 配置
  // 例如：{ "openai": { apiKey: "..." } }
  provider: z.record(z.string(), ProviderConfig).optional()
})

// markdown 配置解析
// 读取 opencode.md 文件内容
const content = await Bun.file("opencode.md").text()

// 使用 gray-matter 解析 frontmatter（YAML 格式的元数据）
// frontmatter 位于文件开头的 --- 分隔符之间
const { data } = matter(content)

// 使用 Zod 验证配置，确保类型正确
// 如果验证失败，会抛出详细的错误信息
const config = Config.parse(data)
```

**练习**：添加自定义配置项，实现验证逻辑。

#### 2.2 项目实例管理
**贴合文件**：`src/project/instance.ts`

**决策还原**：
- **Why 实例隔离**：多项目支持、状态隔离、资源管理
- **Trade-off**：内存开销 vs. 灵活性

**代码伴读**：
```typescript
// src/project/instance.ts - 实例状态管理
/**
 * 创建实例级状态
 * @param init 初始化函数，返回状态值
 * @param dispose 清理函数，在实例销毁时调用（可选）
 * @returns 状态访问函数
 */
export function state<T>(
  init: () => T | Promise<T>,
  dispose?: (state: T) => void | Promise<void>
): () => T {
  // 使用 Symbol 作为 key，确保状态在不同实例间隔离
  // Symbol 是唯一的，不会与其他状态冲突
  const key = Symbol()
  
  // 返回状态访问函数
  return () => {
    // 获取当前实例（每个项目目录对应一个实例）
    const instance = getCurrentInstance()
    
    // 懒加载：首次访问时才初始化
    if (!instance.state.has(key)) {
      // 执行初始化函数
      const value = init()
      
      // 存储状态值
      instance.state.set(key, value)
      
      // 注册清理函数（如果提供）
      // 实例销毁时会自动调用所有 disposers
      if (dispose) {
        instance.disposers.push(() => dispose(value))
      }
    }
    
    // 返回状态值
    return instance.state.get(key)
  }
}
```

**Insight**：使用 Symbol 作为 key 确保状态隔离，每个实例有独立的状态空间。dispose 函数在实例销毁时自动调用，确保资源正确清理（如关闭数据库连接、停止定时器等）。

#### 2.3 事件驱动架构（EventBus）⭐
**贴合文件**：`src/bus/index.ts`、`src/bus/bus-event.ts`、`src/bus/global.ts`

**决策还原**：
- **Why EventBus**：解耦模块、异步通信、可扩展性
- **Trade-off**：调试复杂度 vs. 架构灵活性

**详细文档**：参见 `docs/book/eventbus-architecture.md`

**代码伴读**：
```typescript
// src/bus/bus-event.ts - 事件定义
export namespace BusEvent {
  /**
   * 定义类型安全的事件
   * @param type 事件类型（字符串标识符）
   * @param properties 事件属性的 Zod schema
   * @returns 事件定义对象
   */
  export function define<Type extends string, Properties extends ZodType>(
    type: Type, 
    properties: Properties
  ) {
    // 返回事件定义，包含类型和属性 schema
    // 用于发布和订阅时的类型检查
    return { type, properties }
  }
}

// 定义会话相关事件
export const Event = {
  // 会话创建事件
  Created: BusEvent.define(
    "session.created",  // 事件类型标识符
    z.object({
      sessionID: z.string(),      // 会话 ID
      info: Session.Info          // 会话信息（包含 agent、model 等）
    })
  ),
  
  // 会话更新事件
  Updated: BusEvent.define(
    "session.updated",
    z.object({
      sessionID: z.string(),
      info: Session.Info
    })
  )
}
```

// src/bus/index.ts - 发布订阅实现
export namespace Bus {
  // 发布事件
  export async function publish<Definition extends BusEvent.Definition>(
    def: Definition,
    properties: z.output<Definition["properties"]>
  ) {
    const payload = { type: def.type, properties }
    
    // 通知本地订阅者
    for (const key of [def.type, "*"]) {
      const subscribers = state().subscriptions.get(key)
      for (const sub of subscribers ?? []) {
        await sub(payload)
      }
    }
    
    // 通知全局订阅者（跨实例）
    GlobalBus.emit("event", {
      directory: Instance.directory,
      payload
    })
  }
  
  // 订阅事件
  export function subscribe<Definition extends BusEvent.Definition>(
    def: Definition,
    callback: (event: { 
      type: Definition["type"]
      properties: z.infer<Definition["properties"]> 
    }) => void
  ) {
    const subscriptions = state().subscriptions
    let match = subscriptions.get(def.type) ?? []
    match.push(callback)
    subscriptions.set(def.type, match)
    
    // 返回取消订阅函数
    return () => {
      const index = match.indexOf(callback)
      if (index !== -1) match.splice(index, 1)
    }
  }
  
  // 订阅所有事件
  export function subscribeAll(callback: (event: any) => void) {
    return raw("*", callback)
  }
}
```

**事件类型**（30+ 个）：
```typescript
// 会话事件
Session.Event.Created
Session.Event.Updated
Session.Event.Deleted
Session.Event.Error

// 消息事件
MessageV2.Event.Updated
MessageV2.Event.Removed
MessageV2.Event.PartUpdated
MessageV2.Event.PartRemoved

// 权限事件
Permission.Event.Asked
Permission.Event.Replied

// 文件事件
File.Event.Edited
FileWatcher.Event.Updated

// 命令事件
Command.Event.Executed

// TUI 事件
TuiEvent.PromptAppend
TuiEvent.CommandExecute
TuiEvent.ToastShow
```

**使用场景**：

1. **会话状态同步**
```typescript
// src/session/index.ts - 发布会话创建事件
// 1. 先将会话信息写入存储
await Storage.write(["session", projectID, sessionID], session)

// 2. 发布事件通知其他模块
// 订阅者可以响应此事件（如同步到云端、更新 UI 等）
Bus.publish(Event.Created, {
  sessionID,
  info: session
})

// src/share/share.ts - 订阅并同步到云端
// 监听会话更新事件
Bus.subscribe(Session.Event.Updated, async (evt) => {
  // 将会话信息同步到云端存储
  await sync("session/info/" + evt.properties.info.id, evt.properties.info)
})
```

2. **文件变更通知**
```typescript
// src/tool/write.ts - 发布文件编辑事件
// 1. 写入文件
await Bun.write(filepath, content)

// 2. 发布事件通知文件已变更
await Bus.publish(File.Event.Edited, { file: filepath })

// src/format/index.ts - 订阅并自动格式化
// 监听文件编辑事件
Bus.subscribe(File.Event.Edited, async (payload) => {
  const file = payload.properties.file
  
  // 自动格式化文件（如 Prettier）
  await formatFile(file)
})
```

3. **实时 UI 更新**
```typescript
// src/server/server.ts - SSE 流式推送
// 订阅所有事件，推送到前端
const unsub = Bus.subscribeAll(async (event) => {
  // 通过 Server-Sent Events 推送到浏览器
  await stream.writeSSE({
    data: JSON.stringify(event)
  })
})
```

**架构图**：
```
┌─────────────────────────────────────────┐
│           EventBus 架构                  │
├─────────────────────────────────────────┤
│                                          │
│  ┌──────────┐      ┌──────────┐        │
│  │ Session  │─────>│   Bus    │        │
│  │ Module   │      │ (Local)  │        │
│  └──────────┘      └─────┬────┘        │
│                           │             │
│  ┌──────────┐            │             │
│  │  Share   │<───────────┘             │
│  │ Module   │                          │
│  └──────────┘      ┌──────────┐        │
│                    │ GlobalBus│        │
│  ┌──────────┐     │(Cross-   │        │
│  │  Format  │<────│Instance) │        │
│  │ Module   │     └──────────┘        │
│  └──────────┘                          │
│                                         │
└─────────────────────────────────────────┘
```

**Insight**：
- **本地 Bus**：实例内通信，自动清理
- **GlobalBus**：跨实例通信，基于 EventEmitter
- **类型安全**：Zod 验证事件 payload
- **通配符订阅**：`subscribeAll("*")` 监听所有事件

**练习**：
1. 定义自定义事件，实现发布订阅
2. 实现事件日志记录中间件
3. 添加事件重放功能（用于调试）

#### 2.4 数据持久化与存储抽象（Storage）⭐
**贴合文件**：`src/storage/storage.ts`

**决策还原**：
- **Why 统一存储接口**：简化数据访问、支持迁移、事务安全
- **Trade-off**：抽象层开销 vs. 灵活性

**代码伴读**：
```typescript
// src/storage/storage.ts - 统一存储 API
export namespace Storage {
  /**
   * 读取数据
   * @param key 存储路径（数组形式，如 ["session", projectID, sessionID]）
   * @returns 解析后的 JSON 数据
   */
  export async function read<T>(key: string[]): Promise<T> {
    // 获取存储目录（~/.opencode/storage/）
    const dir = await state().then((x) => x.dir)
    
    // 构造文件路径：~/.opencode/storage/session/projectID/sessionID.json
    const target = path.join(dir, ...key) + ".json"
    
    // 使用读锁保护并发访问
    // using 语法确保锁在作用域结束时自动释放
    using _ = await Lock.read(target)
    
    // 读取并解析 JSON 文件
    const result = await Bun.file(target).json()
    return result as T
  }
  
  /**
   * 写入数据
   * @param key 存储路径
   * @param content 要写入的数据（会被序列化为 JSON）
   */
  export async function write<T>(key: string[], content: T) {
    const dir = await state().then((x) => x.dir)
    const target = path.join(dir, ...key) + ".json"
    
    // 使用写锁保护并发访问
    // 确保同一时间只有一个写操作
    using _ = await Lock.write(target)
    
    // 序列化为格式化的 JSON（缩进 2 空格）
    await Bun.write(target, JSON.stringify(content, null, 2))
  }
  
  /**
   * 原子更新（读取-修改-写入）
   * @param key 存储路径
   * @param fn 修改函数，接收当前数据的草稿
   * @returns 更新后的数据
   */
  export async function update<T>(key: string[], fn: (draft: T) => void) {
    using _ = await Lock.write(target)
    
    // 读取当前数据
    const content = await Bun.file(target).json()
    
    // 修改草稿（直接修改对象）
    fn(content)
    
    // 写回文件
    await Bun.write(target, JSON.stringify(content, null, 2))
    return content as T
  }
  
  /**
   * 列出所有项
   * @param prefix 路径前缀
   * @returns 所有匹配的路径数组
   */
  export async function list(prefix: string[]): Promise<string[][]> {
    const dir = await state().then((x) => x.dir)
    
    // 使用 glob 扫描所有 .json 文件
    const results = await Array.fromAsync(
      glob.scan({ cwd: path.join(dir, ...prefix) })
    )
    
    // 转换为路径数组（去掉 .json 后缀）
    return results.map((x) => [...prefix, ...x.slice(0, -5).split(path.sep)])
  }
}
```

**数据迁移机制**：
```typescript
// 版本化迁移
const MIGRATIONS: Migration[] = [
  // Migration 0: 从旧版本项目结构迁移
  async (dir) => {
    const project = path.resolve(dir, "../project")
    // 迁移会话、消息、部分数据
    for await (const sessionFile of glob.scan("storage/session/info/*.json")) {
      const dest = path.join(dir, "session", projectID, path.basename(sessionFile))
      await Bun.write(dest, JSON.stringify(session))
    }
  },
  
  // Migration 1: 分离 session diff 数据
  async (dir) => {
    for await (const item of glob.scan("session/*/*.json")) {
      const session = await Bun.file(item).json()
      if (session.summary?.diffs) {
        // 将 diffs 移到单独文件
        await Bun.file(path.join(dir, "session_diff", session.id + ".json"))
          .write(JSON.stringify(session.summary.diffs))
      }
    }
  }
]

// 自动执行迁移
const migration = await Bun.file(path.join(dir, "migration")).json()
for (let index = migration; index < MIGRATIONS.length; index++) {
  await MIGRATIONS[index](dir)
  await Bun.write(path.join(dir, "migration"), (index + 1).toString())
}
```

**存储路径结构**：
```
~/.opencode/storage/
├── migration              # 迁移版本号
├── project/
│   └── {projectID}.json   # 项目信息
├── session/
│   └── {projectID}/
│       └── {sessionID}.json  # 会话信息
├── message/
│   └── {sessionID}/
│       └── {messageID}.json  # 消息内容
├── part/
│   └── {messageID}/
│       └── {partID}.json     # 消息部分
└── session_diff/
    └── {sessionID}.json      # 会话 diff 数据
```

**Insight**：
- **读写锁**：使用 `using` 语法自动释放锁
- **JSON 格式**：易于调试、版本控制友好
- **迁移机制**：支持无缝升级，保留用户数据
- **错误处理**：ENOENT 转换为 NotFoundError

**练习**：
1. 实现自定义存储后端（如 SQLite）
2. 添加数据压缩功能
3. 实现存储配额管理

#### 2.5 环境变量处理
**贴合文件**：`src/env/`、`sst-env.d.ts`

**代码伴读**：
```typescript
// src/env/index.ts - 类型安全的环境变量
export namespace Env {
  export function get(key: string): string | undefined {
    return process.env[key]
  }
  
  export function all(): Record<string, string> {
    return process.env as Record<string, string>
  }
}
```

**练习**：添加环境变量验证，防止配置错误。

#### 2.6 全局状态系统
**贴合文件**：`src/global/`

**代码伴读**：
```typescript
// src/global/index.ts - 全局状态
export namespace Global {
  export const Path = {
    data: path.join(os.homedir(), ".opencode"),
    config: path.join(os.homedir(), ".opencode", "config")
  }
}
```

**技术难点**：
- 配置热重载机制
- 事件订阅的内存泄漏防护
- 跨实例事件同步

**企业级价值**：
- 多环境部署（通过配置隔离）
- 配置版本控制（markdown 配置）
- 模块解耦（事件驱动）
- 实时状态同步（EventBus）
- 可观测性（事件日志）
- 数据持久化（Storage 抽象）
- 无缝升级（迁移机制）

**本章小结**：
本章实现了完整的配置系统和项目实例管理，使用 Zod 提供类型安全的配置验证，使用 Symbol 实现实例级状态隔离。EventBus 事件驱动架构支持 30+ 事件类型，实现了模块间的解耦和实时通信。Storage 抽象层提供统一的数据持久化接口，支持读写锁和版本化迁移。这些核心基础设施为后续的 AI 集成、会话处理和工具系统提供了强大的支撑。

### 第3章：AI提供商抽象与模型集成（Iteration 3: AI Provider）

**迭代目标**：构建统一AI接口，支持多提供商。输出：可切换模型的AI调用层。

#### 3.1 @ai-sdk 集成架构
**贴合文件**：`src/provider/provider.ts`

**决策还原**：
- **Why @ai-sdk**：统一接口、支持 20+ 提供商、流式响应原生支持
- **Trade-off**：抽象层开销 vs. 提供商无关性（选择灵活性）

**代码伴读**：
```typescript
// src/provider/provider.ts - 提供商抽象层
export namespace Provider {
  export const Info = z.object({
    id: z.string(),
    name: z.string(),
    models: z.array(Model)
  })
  
  export const Model = z.object({
    id: z.string(),
    name: z.string(),
    limit: z.object({
      context: z.number(),
      output: z.number()
    }),
    capabilities: z.object({
      streaming: z.boolean(),
      tools: z.boolean(),
      vision: z.boolean().optional()
    })
  })
  
  // 统一的提供商接口
  export async function create(config: {
    providerID: string
    modelID: string
    apiKey?: string
  }): Promise<LanguageModel> {
    const provider = await getProvider(config.providerID)
    
    // 根据提供商 ID 创建对应的模型实例
    switch (config.providerID) {
      case "openai":
        return openai(config.modelID, { apiKey: config.apiKey })
      
      case "anthropic":
        return anthropic(config.modelID, { apiKey: config.apiKey })
      
      case "google":
        return google(config.modelID, { apiKey: config.apiKey })
      
      case "opencode":
        // Zen 用户使用 OpenCode 提供商
        return createOpenCodeModel(config.modelID)
      
      default:
        throw new Error(`Unknown provider: ${config.providerID}`)
    }
  }
  
  // 获取提供商信息
  export async function getProvider(providerID: string): Promise<Info> {
    const providers = await list()
    const provider = providers.find((p) => p.id === providerID)
    if (!provider) {
      throw new Error(`Provider not found: ${providerID}`)
    }
    return provider
  }
  
  // 列出所有提供商
  export async function list(): Promise<Info[]> {
    return [
      {
        id: "openai",
        name: "OpenAI",
        models: [
          {
            id: "gpt-4-turbo",
            name: "GPT-4 Turbo",
            limit: { context: 128000, output: 4096 },
            capabilities: { streaming: true, tools: true, vision: true }
          },
          {
            id: "gpt-4",
            name: "GPT-4",
            limit: { context: 8192, output: 4096 },
            capabilities: { streaming: true, tools: true }
          }
        ]
      },
      {
        id: "anthropic",
        name: "Anthropic",
        models: [
          {
            id: "claude-3-5-sonnet-20241022",
            name: "Claude 3.5 Sonnet",
            limit: { context: 200000, output: 8192 },
            capabilities: { streaming: true, tools: true, vision: true }
          }
        ]
      },
      // ... 18 more providers
    ]
  }
}
```

**支持的提供商**（20+）：
- OpenAI (GPT-4, GPT-3.5)
- Anthropic (Claude 3.5)
- Google (Gemini)
- Mistral
- Cohere
- Groq
- Together AI
- Fireworks AI
- Perplexity
- DeepSeek
- ... 更多

**Insight**：使用 @ai-sdk 的 `LanguageModel` 接口，所有提供商都有统一的调用方式。

**练习**：添加新的提供商支持（如 Ollama）。

#### 3.2 模型配置管理
**贴合文件**：`src/provider/models.ts`

**决策还原**：
- **Why 动态配置**：支持运行时切换、用户自定义模型
- **Trade-off**：配置复杂度 vs. 灵活性

**代码伴读**：
```typescript
// src/provider/models.ts - 模型配置
export namespace Models {
  // 从配置文件加载模型
  export async function load(): Promise<Provider.Model[]> {
    const config = await Config.get()
    const models: Provider.Model[] = []
    
    // 1. 加载内置模型
    for (const provider of await Provider.list()) {
      models.push(...provider.models)
    }
    
    // 2. 加载用户自定义模型
    if (config.provider) {
      for (const [providerID, providerConfig] of Object.entries(config.provider)) {
        if (providerConfig.models) {
          for (const modelConfig of providerConfig.models) {
            models.push({
              id: `${providerID}/${modelConfig.id}`,
              name: modelConfig.name,
              limit: modelConfig.limit,
              capabilities: modelConfig.capabilities
            })
          }
        }
      }
    }
    
    return models
  }
  
  // 获取默认模型
  export async function getDefault(): Promise<Provider.Model> {
    const config = await Config.get()
    const defaultModelID = config.model ?? "anthropic/claude-3-5-sonnet-20241022"
    
    const models = await load()
    const model = models.find((m) => m.id === defaultModelID)
    
    if (!model) {
      throw new Error(`Default model not found: ${defaultModelID}`)
    }
    
    return model
  }
}
```

**模型能力映射**：
```typescript
// 不同模型的能力差异
const capabilities = {
  "gpt-4-turbo": {
    streaming: true,
    tools: true,
    vision: true,
    maxTokens: 128000
  },
  "claude-3-5-sonnet": {
    streaming: true,
    tools: true,
    vision: true,
    maxTokens: 200000
  },
  "gpt-3.5-turbo": {
    streaming: true,
    tools: true,
    vision: false,
    maxTokens: 16385
  }
}
```

**Insight**：模型能力映射用于工具过滤和功能降级。

**练习**：实现模型能力检测，自动禁用不支持的功能。

#### 3.3 流式响应处理
**贴合文件**：`src/session/llm.ts`

**决策还原**：
- **Why 流式响应**：实时反馈、可中断、内存友好
- **Trade-off**：实现复杂度 vs. 用户体验

**代码伴读**：
```typescript
// src/session/llm.ts - 流式响应处理
export namespace LLM {
  export async function stream(input: StreamInput) {
    const model = await Provider.create({
      providerID: input.model.providerID,
      modelID: input.model.modelID,
      apiKey: input.apiKey
    })
    
    // 使用 @ai-sdk 的 streamText
    const result = await streamText({
      model,
      messages: input.messages,
      tools: input.tools,
      maxTokens: input.model.limit.output,
      temperature: 0.7,
      abortSignal: input.abort
    })
    
    return {
      // 完整流（包含所有事件）
      fullStream: result.fullStream,
      
      // 仅文本流
      textStream: result.textStream,
      
      // 工具调用流
      toolCalls: result.toolCalls,
      
      // 完成信息
      usage: result.usage,
      finishReason: result.finishReason
    }
  }
}
```

**流式事件类型**：
```typescript
// fullStream 事件类型
type StreamEvent = 
  | { type: "text-delta"; textDelta: string }
  | { type: "tool-call"; toolCallId: string; toolName: string; args: any }
  | { type: "tool-result"; toolCallId: string; result: any }
  | { type: "finish"; finishReason: string; usage: TokenUsage }
  | { type: "error"; error: Error }
```

**使用示例**：
```typescript
// 处理流式响应
for await (const event of stream.fullStream) {
  switch (event.type) {
    case "text-delta":
      // 实时显示文本
      process.stdout.write(event.textDelta)
      break
    
    case "tool-call":
      // 执行工具调用
      const result = await executeTool(event.toolName, event.args)
      break
    
    case "finish":
      // 完成处理
      console.log(`\nTokens used: ${event.usage.totalTokens}`)
      break
  }
}
```

**Insight**：流式响应支持中断（AbortSignal），用户可随时停止生成。

**练习**：实现流式响应的暂停/恢复功能。

#### 3.4 提供商转换层
**贴合文件**：`src/provider/transform.ts`

**决策还原**：
- **Why 转换层**：统一不同提供商的输入输出格式
- **Trade-off**：性能损耗 vs. 兼容性

**代码伴读**：
```typescript
// src/provider/transform.ts - 输入输出转换
export namespace Transform {
  // 转换消息格式
  export function messages(
    messages: Message[],
    providerID: string
  ): CoreMessage[] {
    return messages.map((msg) => {
      // 统一的消息格式
      const base = {
        role: msg.role,
        content: msg.content
      }
      
      // 提供商特定转换
      switch (providerID) {
        case "openai":
          // OpenAI 支持 name 字段
          return { ...base, name: msg.name }
        
        case "anthropic":
          // Anthropic 不支持 name 字段
          return base
        
        default:
          return base
      }
    })
  }
  
  // 转换工具定义
  export function tools(
    tools: Tool[],
    providerID: string
  ): CoreTool[] {
    return tools.map((tool) => ({
      type: "function",
      function: {
        name: tool.id,
        description: tool.description,
        parameters: zodToJsonSchema(tool.parameters)
      }
    }))
  }
}
```

**Insight**：转换层隐藏提供商差异，上层代码无需关心具体提供商。

**练习**：添加新提供商的转换规则。

#### 3.5 认证与安全
**贴合文件**：`src/provider/auth.ts`

**代码伴读**：
```typescript
// src/provider/auth.ts - API 密钥管理
export namespace Auth {
  // 获取 API 密钥
  export async function getApiKey(providerID: string): Promise<string | undefined> {
    // 1. 从环境变量读取
    const envKey = process.env[`${providerID.toUpperCase()}_API_KEY`]
    if (envKey) return envKey
    
    // 2. 从配置文件读取
    const config = await Config.get()
    const providerConfig = config.provider?.[providerID]
    if (providerConfig?.apiKey) return providerConfig.apiKey
    
    // 3. 从插件获取（如 Codex Auth Plugin）
    const plugins = await Plugin.list()
    for (const plugin of plugins) {
      if (plugin.auth) {
        const result = await plugin.auth({ providerID })
        if (result.apiKey) return result.apiKey
      }
    }
    
    return undefined
  }
  
  // 验证 API 密钥
  export async function validate(
    providerID: string,
    apiKey: string
  ): Promise<boolean> {
    const model = await Provider.create({
      providerID,
      modelID: "test",
      apiKey
    })
    
    // 发送测试请求
    const result = await streamText({
      model,
      messages: [{ role: "user", content: "test" }],
      maxTokens: 1
    }).catch(() => null)
    
    return result !== null
  }
}
```

**安全最佳实践**：
- ✅ 优先使用环境变量
- ✅ 配置文件中的密钥加密存储
- ✅ 支持插件提供认证（如 OAuth）
- ✅ 定期验证密钥有效性

**练习**：实现 API 密钥加密存储。

**技术难点**：
- 流式响应的错误处理和重试
- 不同提供商的速率限制处理
- API 密钥的安全存储

**企业级价值**：
- 成本控制（多提供商比价）
- 供应商风险分散（避免单点依赖）
- 灵活切换（根据任务选择最优模型）

**本章小结**：
本章构建了统一的 AI 提供商抽象层，支持 20+ 提供商和 50+ 模型。通过 @ai-sdk 实现了提供商无关的接口，支持流式响应、工具调用和多模态输入。认证系统支持环境变量、配置文件和插件三种方式，确保 API 密钥的安全管理。

### 第4章：文件系统与版本控制（Iteration 4: File System）

**迭代目标**：构建完整的文件管理与版本控制系统。输出：安全、高效的文件操作能力。

#### 4.1 文件操作抽象层
**贴合文件**：`src/file/`、`src/util/filesystem.ts`

**决策还原**：
- **Why 抽象层**：跨平台兼容、权限控制、错误处理
- **Trade-off**：性能开销 vs. 安全性

**代码伴读**：
```typescript
// src/util/filesystem.ts - 文件系统工具
export namespace Filesystem {
  export async function isDir(target: string): Promise<boolean> {
    return fs.stat(target)
      .then((stat) => stat.isDirectory())
      .catch(() => false)
  }
  
  export async function isFile(target: string): Promise<boolean> {
    return fs.stat(target)
      .then((stat) => stat.isFile())
      .catch(() => false)
  }
  
  // 向上查找文件
  export async function* up(opts: {
    targets: string[]
    start: string
    stop: string
  }) {
    let current = opts.start
    while (current.startsWith(opts.stop)) {
      for (const target of opts.targets) {
        const candidate = path.join(current, target)
        if (await isDir(candidate)) {
          yield candidate
        }
      }
      const parent = path.dirname(current)
      if (parent === current) break
      current = parent
    }
  }
}
```

**练习**：实现文件监控去重，避免重复事件。

#### 4.2 .gitignore 规则处理
**贴合文件**：`src/file/ignore.ts`

**代码伴读**：
```typescript
// src/file/ignore.ts - 忽略规则
export namespace FileIgnore {
  export const PATTERNS = [
    "**/node_modules/**",
    "**/.git/**",
    "**/dist/**",
    "**/build/**",
    "**/.next/**",
    "**/.turbo/**"
  ]
}
```

**练习**：支持 .gitignore 文件解析。

#### 4.3 Git 集成与版本控制
**贴合文件**：`src/project/vcs.ts`

**代码伴读**：
```typescript
// src/project/vcs.ts - VCS 检测
export namespace Vcs {
  export async function detect(directory: string): Promise<"git" | "none"> {
    const gitDir = path.join(directory, ".git")
    const isGit = await Filesystem.isDir(gitDir)
    return isGit ? "git" : "none"
  }
  
  export const Event = {
    BranchUpdated: BusEvent.define(
      "vcs.branch.updated",
      z.object({
        branch: z.string(),
        projectID: z.string()
      })
    )
  }
}
```

**练习**：添加分支切换检测。

#### 4.4 文件监控与实时同步（FileWatcher）⭐
**贴合文件**：`src/file/watcher.ts`

**决策还原**：
- **Why @parcel/watcher**：高性能、跨平台、原生后端
- **Trade-off**：依赖原生模块 vs. 性能

**代码伴读**：
```typescript
// src/file/watcher.ts - 文件监控系统
export namespace FileWatcher {
  export const Event = {
    Updated: BusEvent.define(
      "file.watcher.updated",
      z.object({
        file: z.string(),
        event: z.union([
          z.literal("add"),
          z.literal("change"),
          z.literal("unlink")
        ])
      })
    )
  }
  
  const state = Instance.state(async () => {
    const w = watcher() // 加载 @parcel/watcher
    if (!w) return {}
    
    const subscribe: ParcelWatcher.SubscribeCallback = (err, evts) => {
      if (err) return
      for (const evt of evts) {
        if (evt.type === "create") 
          Bus.publish(Event.Updated, { file: evt.path, event: "add" })
        if (evt.type === "update") 
          Bus.publish(Event.Updated, { file: evt.path, event: "change" })
        if (evt.type === "delete") 
          Bus.publish(Event.Updated, { file: evt.path, event: "unlink" })
      }
    }
    
    // 订阅工作目录
    const sub = await w.subscribe(Instance.directory, subscribe, {
      ignore: [...FileIgnore.PATTERNS, ...cfgIgnores],
      backend // fs-events/inotify/windows
    })
    
    return { subs: [sub] }
  })
}
```

**平台后端**：
- macOS: `fs-events`（原生 FSEvents API）
- Linux: `inotify`（支持 glibc/musl）
- Windows: `windows`（原生 ReadDirectoryChangesW）

**Insight**：
- 订阅超时保护（10 秒）
- 忽略 .git 内部文件（除 HEAD）
- 自动清理订阅（Instance.state dispose）

**练习**：
1. 实现文件变更批处理
2. 添加变更率限制
3. 支持自定义忽略规则

#### 4.5 Git Worktree 沙盒管理（Worktree）⭐
**贴合文件**：`src/worktree/index.ts`

**决策还原**：
- **Why Git Worktree**：隔离环境、并行开发、安全测试
- **Trade-off**：磁盘空间 vs. 安全性

**代码伴读**：
```typescript
// src/worktree/index.ts - Worktree 管理
export namespace Worktree {
  export const Event = {
    Ready: BusEvent.define(
      "worktree.ready",
      z.object({
        name: z.string(),
        branch: z.string()
      })
    ),
    Failed: BusEvent.define(
      "worktree.failed",
      z.object({ message: z.string() })
    )
  }
  
  // 创建 worktree
  export const create = fn(CreateInput.optional(), async (input) => {
    const root = path.join(Global.Path.data, "worktree", Instance.project.id)
    await fs.mkdir(root, { recursive: true })
    
    // 生成唯一名称
    const info = await candidate(root, input?.name)
    // info: { name: "brave-falcon", branch: "opencode/brave-falcon", directory: "..." }
    
    // 创建 git worktree
    await $`git worktree add --no-checkout -b ${info.branch} ${info.directory}`
      .cwd(Instance.worktree)
    
    // 异步初始化
    setTimeout(async () => {
      // 1. 检出文件
      await $`git reset --hard`.cwd(info.directory)
      
      // 2. 引导实例
      await Instance.provide({
        directory: info.directory,
        init: InstanceBootstrap,
        fn: () => undefined
      })
      
      // 3. 发布就绪事件
      GlobalBus.emit("event", {
        directory: info.directory,
        payload: {
          type: Event.Ready.type,
          properties: { name: info.name, branch: info.branch }
        }
      })
      
      // 4. 执行启动脚本
      await runStartScripts(info.directory, { projectID, extra: input?.startCommand })
    }, 0)
    
    return info
  })
}
```

**名称生成算法**：
```typescript
const ADJECTIVES = ["brave", "calm", "clever", "cosmic", ...] // 30 个
const NOUNS = ["cabin", "eagle", "falcon", "nebula", ...] // 31 个

function randomName() {
  return `${pick(ADJECTIVES)}-${pick(NOUNS)}`
}

// 示例：brave-falcon, cosmic-nebula, swift-tiger
```

**Insight**：
- 930 种名称组合（30 × 31）
- 冲突时自动添加后缀
- 异步初始化不阻塞创建
- 自动执行项目启动脚本

**练习**：
1. 实现 worktree 列表查询
2. 添加 worktree 状态监控
3. 支持 worktree 快照

**技术难点**：
- 跨平台文件监控
- Git worktree 生命周期管理
- 文件变更事件去重

**企业级价值**：
- 实时协作（文件监控）
- 安全测试（worktree 隔离）
- 并行开发（多 worktree）
- 版本控制集成

**本章小结**：
本章构建了完整的文件系统抽象层，支持跨平台文件操作、.gitignore 规则处理和 Git 集成。FileWatcher 提供实时文件监控能力，支持 macOS/Linux/Windows 三大平台。Worktree 系统实现了 Git 工作树管理，支持隔离环境和并行开发。这些基础设施为后续的工具系统和会话处理提供了坚实的基础。

### 第5章：工具系统实现（Iteration 5: Tool System）

**迭代目标**：实现可扩展工具调用系统。输出：完整的文件操作、执行能力。

#### 5.1 工具定义框架
**贴合文件**：`src/tool/tool.ts`

**决策还原**：
- **Why Zod 参数验证**：类型安全、运行时验证、自动生成文档
- **Trade-off**：学习成本 vs. 类型安全

**代码伴读**：
```typescript
// src/tool/tool.ts - 工具接口
export namespace Tool {
  export interface Info {
    id: string
    init: (ctx?: { agent?: Agent.Info }) => Promise<{
      parameters: ZodType
      description: string
      execute: (args: any, ctx: Context) => Promise<Result>
    }>
  }
  
  export interface Result {
    title: string
    output: string
    metadata?: Record<string, any>
  }
}
```

**练习**：定义自定义工具，实现参数验证。

#### 5.2 文件操作工具
**贴合文件**：`src/tool/{read,write,edit,glob,grep}.ts`

**代码伴读**：
```typescript
// src/tool/read.ts - 文件读取工具
export const ReadTool: Tool.Info = {
  id: "read",
  init: async () => ({
    parameters: z.object({
      path: z.string().describe("File path to read")
    }),
    description: "Read file contents",
    execute: async (args, ctx) => {
      const filepath = path.resolve(ctx.directory, args.path)
      const content = await Bun.file(filepath).text()
      
      return {
        title: `Read: ${args.path}`,
        output: content,
        metadata: {
          size: content.length,
          lines: content.split('\n').length
        }
      }
    }
  })
}

// src/tool/write.ts - 文件写入工具
export const WriteTool: Tool.Info = {
  id: "write",
  init: async () => ({
    parameters: z.object({
      path: z.string().describe("File path to write"),
      content: z.string().describe("Content to write")
    }),
    description: "Write content to file",
    execute: async (args, ctx) => {
      const filepath = path.resolve(ctx.directory, args.path)
      await Bun.write(filepath, args.content)
      
      // 发布文件编辑事件
      await Bus.publish(File.Event.Edited, { file: filepath })
      
      return {
        title: `Wrote: ${args.path}`,
        output: `Successfully wrote ${args.content.length} bytes`,
        metadata: {
          size: args.content.length
        }
      }
    }
  })
}
```

**Insight**：文件操作工具发布事件，触发自动格式化、文件监控等后续处理。

**练习**：实现文件批量操作工具。

#### 5.3 执行环境工具
**贴合文件**：`src/tool/bash.ts`、`src/pty/`

**代码伴读**：
```typescript
// src/tool/bash.ts - 命令执行工具
export const BashTool: Tool.Info = {
  id: "bash",
  init: async () => ({
    parameters: z.object({
      command: z.string().describe("Shell command to execute")
    }),
    description: "Execute shell commands",
    execute: async (args, ctx) => {
      const result = await $`${args.command}`
        .cwd(ctx.directory)
        .nothrow()
      
      return {
        title: `$ ${args.command}`,
        output: result.text(),
        metadata: {
          exitCode: result.exitCode,
          stderr: result.stderr.toString()
        }
      }
    }
  })
}
```

**安全机制**：
- 权限预检查（Permission.ask）
- 命令参数验证（BashArity）
- 工作目录限制（ctx.directory）

**练习**：实现命令执行超时控制。

#### 5.4 网络工具集
**贴合文件**：`src/tool/{webfetch,websearch,codesearch}.ts`

**代码伴读**：
```typescript
// src/tool/webfetch.ts - 网页抓取工具
export const WebFetchTool: Tool.Info = {
  id: "webfetch",
  init: async () => ({
    parameters: z.object({
      url: z.string().url().describe("URL to fetch")
    }),
    description: "Fetch web page content",
    execute: async (args, ctx) => {
      const response = await fetch(args.url)
      const content = await response.text()
      
      return {
        title: `Fetched: ${args.url}`,
        output: content,
        metadata: {
          status: response.status,
          contentType: response.headers.get('content-type')
        }
      }
    }
  })
}
```

**练习**：添加网页内容解析（HTML → Markdown）。

**技术难点**：
- 并发工具调用管理
- 工具执行超时控制
- 工具输出截断策略

**企业级价值**：
- 自动化能力（文件操作、命令执行）
- 集成生态（网络工具、外部 API）
- 可扩展性（自定义工具）

**本章小结**：
本章实现了完整的工具系统，包括工具定义框架、文件操作工具、命令执行工具和网络工具。所有工具都使用 Zod 进行参数验证，确保类型安全。文件操作工具与 EventBus 集成，触发自动格式化等后续处理。命令执行工具实现了权限预检查和安全控制。这些工具为 AI 代理提供了强大的自动化能力。

#### 4.6 文件操作抽象层补充
**贴合文件**：`src/file/`、`src/util/filesystem.ts`

**决策还原**：
- **Why 抽象层**：跨平台兼容、权限控制、错误处理
- **Trade-off**：性能开销 vs. 安全性

**代码伴读**：
```typescript
// src/util/filesystem.ts - 文件系统工具
export namespace Filesystem {
  export async function isDir(target: string): Promise<boolean> {
    return fs.stat(target)
      .then((stat) => stat.isDirectory())
      .catch(() => false)
  }
  
  export async function isFile(target: string): Promise<boolean> {
    return fs.stat(target)
      .then((stat) => stat.isFile())
      .catch(() => false)
  }
  
  // 向上查找文件
  export async function* up(opts: {
    targets: string[]
    start: string
    stop: string
  }) {
    let current = opts.start
    while (current.startsWith(opts.stop)) {
      for (const target of opts.targets) {
        const candidate = path.join(current, target)
        if (await isDir(candidate)) {
          yield candidate
        }
      }
      const parent = path.dirname(current)
      if (parent === current) break
      current = parent
    }
  }
}
```

**练习**：实现文件监控去重，避免重复事件。

#### 4.7 .gitignore 规则处理补充
**贴合文件**：`src/file/ignore.ts`

**代码伴读**：
```typescript
// src/file/ignore.ts - 忽略规则
export namespace FileIgnore {
  export const PATTERNS = [
    "**/node_modules/**",
    "**/.git/**",
    "**/dist/**",
    "**/build/**",
    "**/.next/**",
    "**/.turbo/**"
  ]
}
```

**练习**：支持 .gitignore 文件解析。

#### 4.8 Git 集成与版本控制补充
**贴合文件**：`src/project/vcs.ts`

**代码伴读**：
```typescript
// src/project/vcs.ts - VCS 检测
export namespace Vcs {
  export async function detect(directory: string): Promise<"git" | "none"> {
    const gitDir = path.join(directory, ".git")
    const isGit = await Filesystem.isDir(gitDir)
    return isGit ? "git" : "none"
  }
  
  export const Event = {
    BranchUpdated: BusEvent.define(
      "vcs.branch.updated",
      z.object({
        branch: z.string(),
        projectID: z.string()
      })
    )
  }
}
```

**练习**：添加分支切换检测。

#### 4.9 文件监控与实时同步（FileWatcher）⭐ - 补充
**贴合文件**：`src/file/watcher.ts`

**决策还原**：
- **Why @parcel/watcher**：高性能、跨平台、原生后端
- **Trade-off**：依赖原生模块 vs. 性能

**代码伴读**：
```typescript
// src/file/watcher.ts - 文件监控系统
export namespace FileWatcher {
  export const Event = {
    Updated: BusEvent.define(
      "file.watcher.updated",
      z.object({
        file: z.string(),
        event: z.union([
          z.literal("add"),
          z.literal("change"),
          z.literal("unlink")
        ])
      })
    )
  }
  
  const state = Instance.state(async () => {
    const w = watcher() // 加载 @parcel/watcher
    if (!w) return {}
    
    const subscribe: ParcelWatcher.SubscribeCallback = (err, evts) => {
      if (err) return
      for (const evt of evts) {
        if (evt.type === "create") 
          Bus.publish(Event.Updated, { file: evt.path, event: "add" })
        if (evt.type === "update") 
          Bus.publish(Event.Updated, { file: evt.path, event: "change" })
        if (evt.type === "delete") 
          Bus.publish(Event.Updated, { file: evt.path, event: "unlink" })
      }
    }
    
    // 订阅工作目录
    const sub = await w.subscribe(Instance.directory, subscribe, {
      ignore: [...FileIgnore.PATTERNS, ...cfgIgnores],
      backend // fs-events/inotify/windows
    })
    
    return { subs: [sub] }
  })
}
```

**平台后端**：
- macOS: `fs-events`（原生 FSEvents API）
- Linux: `inotify`（支持 glibc/musl）
- Windows: `windows`（原生 ReadDirectoryChangesW）

**Insight**：
- 订阅超时保护（10 秒）
- 忽略 .git 内部文件（除 HEAD）
- 自动清理订阅（Instance.state dispose）

**练习**：
1. 实现文件变更批处理
2. 添加变更率限制
3. 支持自定义忽略规则

#### 4.10 Git Worktree 沙盒管理（Worktree）⭐ - 补充
**贴合文件**：`src/worktree/index.ts`

**决策还原**：
- **Why Git Worktree**：隔离环境、并行开发、安全测试
- **Trade-off**：磁盘空间 vs. 安全性

**代码伴读**：
```typescript
// src/worktree/index.ts - Worktree 管理
export namespace Worktree {
  export const Event = {
    Ready: BusEvent.define(
      "worktree.ready",
      z.object({
        name: z.string(),
        branch: z.string()
      })
    ),
    Failed: BusEvent.define(
      "worktree.failed",
      z.object({ message: z.string() })
    )
  }
  
  // 创建 worktree
  export const create = fn(CreateInput.optional(), async (input) => {
    const root = path.join(Global.Path.data, "worktree", Instance.project.id)
    await fs.mkdir(root, { recursive: true })
    
    // 生成唯一名称
    const info = await candidate(root, input?.name)
    // info: { name: "brave-falcon", branch: "opencode/brave-falcon", directory: "..." }
    
    // 创建 git worktree
    await $`git worktree add --no-checkout -b ${info.branch} ${info.directory}`
      .cwd(Instance.worktree)
    
    // 异步初始化
    setTimeout(async () => {
      // 1. 检出文件
      await $`git reset --hard`.cwd(info.directory)
      
      // 2. 引导实例
      await Instance.provide({
        directory: info.directory,
        init: InstanceBootstrap,
        fn: () => undefined
      })
      
      // 3. 发布就绪事件
      GlobalBus.emit("event", {
        directory: info.directory,
        payload: {
          type: Event.Ready.type,
          properties: { name: info.name, branch: info.branch }
        }
      })
      
      // 4. 执行启动脚本
      await runStartScripts(info.directory, { projectID, extra: input?.startCommand })
    }, 0)
    
    return info
  })
  
  // 删除 worktree
  export const remove = fn(RemoveInput, async (input) => {
    await $`git worktree remove --force ${input.directory}`.cwd(Instance.worktree)
    
    const branch = entry.branch?.replace(/^refs\/heads\//, "")
    if (branch) {
      await $`git branch -D ${branch}`.cwd(Instance.worktree)
    }
    
    return true
  })
  
  // 重置到默认分支
  export const reset = fn(ResetInput, async (input) => {
    // 检测默认分支（origin/main 或 origin/master）
    const target = remoteBranch ? `${remote}/${remoteBranch}` : localBranch
    
    // 重置并清理
    await $`git reset --hard ${target}`.cwd(worktreePath)
    await $`git clean -fdx`.cwd(worktreePath)
    await $`git submodule update --init --recursive --force`.cwd(worktreePath)
    
    return true
  })
}
```

**名称生成算法**：
```typescript
const ADJECTIVES = ["brave", "calm", "clever", "cosmic", ...] // 30 个
const NOUNS = ["cabin", "eagle", "falcon", "nebula", ...] // 31 个

function randomName() {
  return `${pick(ADJECTIVES)}-${pick(NOUNS)}`
}

// 示例：brave-falcon, cosmic-nebula, swift-tiger
```

**Insight**：
- 930 种名称组合（30 × 31）
- 冲突时自动添加后缀
- 异步初始化不阻塞创建
- 自动执行项目启动脚本

**练习**：
1. 实现 worktree 列表查询
2. 添加 worktree 状态监控
3. 支持 worktree 快照

**技术难点**：
- 跨平台文件监控
- Git worktree 生命周期管理
- 文件变更事件去重

**企业级价值**：
- 实时协作（文件监控）
- 安全测试（worktree 隔离）
- 并行开发（多 worktree）
- 版本控制集成

**本章小结**：
本章构建了完整的文件系统抽象层，支持跨平台文件操作、.gitignore 规则处理和 Git 集成。FileWatcher 提供实时文件监控能力，支持 macOS/Linux/Windows 三大平台。Worktree 系统实现了 Git 工作树管理，支持隔离环境和并行开发。这些基础设施为后续的工具系统和会话处理提供了坚实的基础。

## 第二部分：核心篇

### 第6章：会话处理与Agent管理（Iteration 6: Session & Agent）

**迭代目标**：实现会话处理循环、Agent 管理、定时任务调度。输出：完整的 AI 对话处理系统。

#### 6.1 Agent 定义框架
**贴合文件**：`src/agent/agent.ts`

**决策还原**：
- **Why 配置化 Agent**：灵活定义、权限隔离、可扩展
- **Trade-off**：配置复杂度 vs. 灵活性

**代码伴读**：
```typescript
// src/agent/agent.ts - 内置 Agent
const result: Record<string, Info> = {
  build: {
    name: "build",
    description: "默认 Agent，执行工具",
    permission: PermissionNext.merge(defaults, user),
    mode: "primary"
  },
  plan: {
    name: "plan",
    description: "只读模式，禁止编辑",
    permission: PermissionNext.merge(defaults, {
      edit: { "*": "deny" }
    }),
    mode: "primary"
  },
  general: {
    name: "general",
    description: "通用 Agent，多步骤任务",
    mode: "subagent"
  },
  explore: {
    name: "explore",
    description: "快速探索代码库",
    prompt: PROMPT_EXPLORE,
    mode: "subagent"
  }
}
```

**练习**：创建自定义 Agent，配置权限规则。

#### 6.2 会话处理循环
**贴合文件**：`src/session/processor.ts`、`src/session/llm.ts`

**决策还原**：
- **Why 流式处理**：实时反馈、可中断、内存友好
- **Trade-off**：复杂度 vs. 用户体验

**代码伴读**：
```typescript
// src/session/processor.ts - 核心处理循环
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  return {
    async process(streamInput: LLM.StreamInput) {
      while (true) {
        const stream = await LLM.stream(streamInput)
        
        for await (const value of stream.fullStream) {
          switch (value.type) {
            case "tool-call":
              // 执行工具调用
              break
            case "text-delta":
              // 流式输出文本
              break
            case "finish-step":
              // 完成一步，检查是否需要继续
              break
          }
        }
        
        // 检查是否需要压缩上下文
        if (needsCompaction) return "compact"
        if (blocked) return "stop"
        return "continue"
      }
    }
  }
}
```

**Insight**：处理循环支持多步推理，每步完成后决定是否继续。

#### 6.3 Doom Loop 检测
**贴合文件**：`src/session/processor.ts`

**技术难点**：防止 AI 陷入重复调用同一工具

**代码伴读**：
```typescript
// 检测最近 3 次工具调用是否相同
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

if (lastThree.length === 3 &&
    lastThree.every(p => 
      p.tool === value.toolName &&
      JSON.stringify(p.state.input) === JSON.stringify(value.input)
    )) {
  // 触发权限询问，打断循环
  await PermissionNext.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    sessionID: input.sessionID
  })
}
```

**练习**：调整 DOOM_LOOP_THRESHOLD，测试不同场景。

#### 6.4 消息压缩算法
**贴合文件**：`src/session/compaction.ts`

**决策还原**：
- **Why 压缩**：控制 token 使用、保持上下文窗口
- **Trade-off**：信息损失 vs. 成本控制

**代码伴读**：
```typescript
// src/session/compaction.ts - 压缩策略
export async function isOverflow(input: {
  tokens: { input: number; output: number }
  model: Provider.Model
}): Promise<boolean> {
  const total = input.tokens.input + input.tokens.output
  const limit = input.model.limit.context * 0.8 // 80% 阈值
  return total > limit
}
```

**Insight**：在达到 80% 上下文窗口时触发压缩，使用专门的 compaction Agent 总结历史消息。

#### 6.5 快照与回滚机制（Snapshot）⭐
**贴合文件**：`src/snapshot/index.ts`

**决策还原**：
- **Why Git-based 快照**：可靠、高效、支持 diff
- **Trade-off**：磁盘空间 vs. 安全性

**代码伴读**：
```typescript
// src/snapshot/index.ts - 快照系统
export namespace Snapshot {
  // 创建快照
  export async function track(): Promise<string> {
    const git = gitdir() // ~/.opencode/snapshot/{projectID}
    
    // 初始化独立 git 仓库
    if (await fs.mkdir(git, { recursive: true })) {
      await $`git init`
        .env({ GIT_DIR: git, GIT_WORK_TREE: Instance.worktree })
        .quiet()
      await $`git --git-dir ${git} config core.autocrlf false`.quiet()
    }
    
    // 添加所有文件并生成 tree hash
    await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet()
    const hash = await $`git --git-dir ${git} --work-tree ${Instance.worktree} write-tree`
      .quiet()
      .text()
    
    return hash.trim()
  }
  
  // 获取变更文件列表
  export async function patch(hash: string): Promise<Patch> {
    await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet()
    
    const result = await $`git --git-dir ${git} --work-tree ${Instance.worktree} diff --name-only ${hash} -- .`
      .quiet()
      .nothrow()
    
    const files = result.text()
      .trim()
      .split("\n")
      .filter(Boolean)
      .map((x) => path.join(Instance.worktree, x))
    
    return { hash, files }
  }
  
  // 回滚到快照
  export async function restore(snapshot: string) {
    await $`git --git-dir ${git} --work-tree ${Instance.worktree} read-tree ${snapshot} && git --git-dir ${git} --work-tree ${Instance.worktree} checkout-index -a -f`
      .quiet()
      .cwd(Instance.worktree)
  }
  
  // 回滚多个补丁
  export async function revert(patches: Patch[]) {
    const files = new Set<string>()
    
    for (const item of patches) {
      for (const file of item.files) {
        if (files.has(file)) continue
        
        // 检出文件到快照版本
        const result = await $`git --git-dir ${git} --work-tree ${Instance.worktree} checkout ${item.hash} -- ${file}`
          .quiet()
          .nothrow()
        
        if (result.exitCode !== 0) {
          // 文件在快照中不存在，删除它
          await fs.unlink(file).catch(() => {})
        }
        
        files.add(file)
      }
    }
  }
  
  // 获取完整 diff
  export async function diffFull(from: string, to: string): Promise<FileDiff[]> {
    const result: FileDiff[] = []
    
    for await (const line of $`git --git-dir ${git} --work-tree ${Instance.worktree} diff --numstat ${from} ${to} -- .`
      .quiet()
      .lines()) {
      const [additions, deletions, file] = line.split("\t")
      
      const before = await $`git --git-dir ${git} show ${from}:${file}`.quiet().text()
      const after = await $`git --git-dir ${git} show ${to}:${file}`.quiet().text()
      
      result.push({
        file,
        before,
        after,
        additions: parseInt(additions),
        deletions: parseInt(deletions)
      })
    }
    
    return result
  }
  
  // 定时清理
  export async function cleanup() {
    await $`git --git-dir ${git} --work-tree ${Instance.worktree} gc --prune=7.days`
      .quiet()
      .cwd(Instance.directory)
  }
}

// 注册定时清理任务
Scheduler.register({
  id: "snapshot.cleanup",
  interval: 60 * 60 * 1000, // 1 小时
  run: Snapshot.cleanup,
  scope: "instance"
})
```

**快照工作流**：
```
1. 用户发起操作
   ↓
2. track() 创建快照 → hash1
   ↓
3. 执行文件修改
   ↓
4. track() 创建新快照 → hash2
   ↓
5. patch(hash1) 获取变更列表
   ↓
6. 用户可选择：
   - 保留修改
   - revert([patch]) 回滚
   - restore(hash1) 完全恢复
```

**Insight**：
- 使用独立 git 仓库（不影响用户仓库）
- 仅存储 tree 对象（不创建 commit）
- 自动清理 7 天前的快照
- 支持 Windows 行尾符（core.autocrlf=false）

**练习**：
1. 实现快照历史查询
2. 添加快照标签功能
3. 支持部分文件回滚

#### 6.6 定时任务调度
**贴合文件**：`src/scheduler/index.ts`

**决策还原**：
- **Why 简单调度器**：满足需求、易于理解、低开销
- **Trade-off**：功能简单 vs. 复杂度低

**代码伴读**：
```typescript
// src/scheduler/index.ts - 定时任务系统（仅 70 行）
export function register(task: Task) {
  const scope = task.scope ?? "instance"
  const entry = scope === "global" ? shared : state()
  
  void run(task) // 立即执行一次
  const timer = setInterval(() => {
    void run(task)
  }, task.interval)
  timer.unref() // 不阻止进程退出
  
  entry.timers.set(task.id, timer)
}

// 使用示例
Scheduler.register({
  id: "tool.truncation.cleanup",
  interval: HOUR_MS,
  run: async () => {
    // 清理临时文件
  }
})
```

**练习**：添加自定义定时任务，实现日志清理功能。

#### 6.7 ACP 协议集成
**贴合文件**：`src/acp/README.md`、`src/acp/agent.ts`

**代码伴读**：
```typescript
// src/acp/agent.ts - ACP Agent 实现
export const ACP = {
  async initialize(params) {
    return {
      protocolVersion: 1,
      agentCapabilities: {
        streaming: false,
        tools: true
      }
    }
  },
  
  async sessionNew(params) {
    const session = await Session.create({
      agent: params.agent ?? "build",
      cwd: params.cwd
    })
    return { sessionId: session.id }
  },
  
  async sessionPrompt(params) {
    // 处理用户消息，返回 AI 响应
  }
}
```

**企业级价值**：
- 工作流自动化（多步骤任务）
- 智能决策支持（Agent 协作）
- 成本控制（消息压缩）
- 可靠性保障（Doom Loop 检测）

**本章小结**：
本章实现了完整的会话处理系统，包括 Agent 定义框架、会话处理循环、Doom Loop 检测和消息压缩算法。Snapshot 快照系统基于 Git 实现了可靠的文件版本管理和回滚机制。定时任务调度器支持实例级和全局级任务。ACP 协议集成提供了标准化的 AI 代理接口。这些核心功能为 AI 编码代理提供了强大的任务处理和状态管理能力。

### 第7章：权限引擎与安全控制（Iteration 7: Permission System）

**迭代目标**：构建完整的权限管理系统。输出：安全、可控的操作权限控制。

#### 7.1 权限系统架构
**贴合文件**：`src/permission/next.ts`

**决策还原**：
- **Why 细粒度权限**：精确控制、用户可配置、安全保障
- **Trade-off**：配置复杂度 vs. 安全性

**代码伴读**：
```typescript
// src/permission/next.ts - 权限评估
export namespace PermissionNext {
  export async function ask(input: {
    permission: string
    patterns: string[]
    sessionID: string
  }): Promise<void> {
    // 评估权限规则
    const result = await evaluate(input)
    
    if (result === "allow") return
    if (result === "deny") throw new PermissionDeniedError()
    
    // 需要用户确认
    await Question.ask({
      sessionID: input.sessionID,
      questions: [{
        question: `Allow ${input.permission} on ${input.patterns.join(", ")}?`,
        header: "Permission Request",
        options: [
          { label: "Allow", description: "Grant permission" },
          { label: "Deny", description: "Reject permission" }
        ]
      }]
    })
  }
}
```

**练习**：实现权限规则配置，支持通配符匹配。

#### 7.2 工具权限控制
**贴合文件**：`src/tool/bash.ts`

**代码伴读**：
```typescript
// 命令执行前检查权限
await ctx.ask({
  permission: "bash",
  patterns: [args.command],
  metadata: { cwd: ctx.directory }
})
```

**练习**：添加危险命令检测，自动拒绝高风险操作。

**企业级价值**：
- 安全保障（权限控制）
- 用户信任（透明操作）
- 合规要求（审计日志）

**本章小结**：
本章实现了完整的权限管理系统，支持细粒度权限控制和用户确认机制。权限系统与工具系统深度集成，确保所有操作都经过权限检查。这些功能为 AI 代理提供了安全保障，增强了用户信任。

### 第8章：TUI界面与终端交互（Iteration 8: Terminal UI）

**迭代目标**：构建功能完整的终端用户界面。输出：直观的交互体验，支持 16+ 组件。

#### 7.1 OpenTUI + SolidJS 架构
**贴合文件**：`src/cli/cmd/tui/app.tsx`、`src/cli/cmd/tui/component/`

**决策还原**：
- **Why OpenTUI**：终端原生组件、响应式更新、高性能
- **Why SolidJS**：细粒度响应式、小体积、快速渲染
- **Trade-off**：学习曲线 vs. 开发体验

**代码伴读**：
```typescript
// src/cli/cmd/tui/app.tsx - TUI 应用入口
export async function tui(input: {
  url: string
  fetch?: typeof fetch
  events?: EventSource
  args: Args
}) {
  return render(() => (
    <SdkProvider url={input.url} fetch={input.fetch} events={input.events}>
      <ThemeProvider>
        <KeybindProvider>
          <Router>
            <Routes>
              <Route path="/" component={Home} />
              <Route path="/session/:id" component={SessionView} />
            </Routes>
          </Router>
        </KeybindProvider>
      </ThemeProvider>
    </SdkProvider>
  ))
}
```

**组件列表**（16+ 个）：
- `dialog-agent.tsx` - Agent 选择对话框
- `dialog-model.tsx` - 模型选择对话框
- `dialog-session-list.tsx` - 会话列表
- `prompt/` - 提示词输入组件
- `todo-item.tsx` - 待办事项
- ... 更多组件

**练习**：创建自定义对话框组件，集成到 TUI。

#### 7.2 多线程 TUI 架构
**贴合文件**：`src/cli/cmd/tui/thread.ts`、`src/cli/cmd/tui/worker.ts`

**决策还原**：
- **Why 主从线程分离**：UI 响应不阻塞、后台任务隔离
- **Trade-off**：复杂度增加 vs. 性能提升

**代码伴读**：
```typescript
// src/cli/cmd/tui/thread.ts - 主线程（UI 线程）
const worker = new Worker(workerPath, {
  env: process.env
})

const client = Rpc.client<typeof rpc>(worker)

// 调用 Worker 方法
const server = await client.call("server", {
  port: 3000,
  hostname: "127.0.0.1"
})

// src/cli/cmd/tui/worker.ts - Worker 线程（业务逻辑）
export const rpc = {
  async fetch(input) {
    // 处理 HTTP 请求
    const request = new Request(input.url, {
      method: input.method,
      headers: input.headers,
      body: input.body
    })
    const response = await Server.App().fetch(request)
    return {
      status: response.status,
      headers: Object.fromEntries(response.headers.entries()),
      body: await response.text()
    }
  },
  
  async server(input) {
    // 启动 HTTP 服务器
    server = Server.listen(input)
    return { url: server.url.toString() }
  },
  
  async shutdown() {
    // 清理资源
    await Instance.disposeAll()
    if (server) server.stop(true)
  }
}

Rpc.listen(rpc)
```

**架构图**：
```
┌─────────────────┐         ┌─────────────────┐
│   Main Thread   │         │  Worker Thread  │
│   (UI Layer)    │         │ (Business Logic)│
├─────────────────┤         ├─────────────────┤
│  OpenTUI        │         │  Server.App()   │
│  SolidJS        │  RPC    │  Session        │
│  Keybindings    │ <-----> │  Provider       │
│  Event Handling │         │  Tool Registry  │
└─────────────────┘         └─────────────────┘
```

**Insight**：支持两种模式
1. **直接 RPC**：无 HTTP 开销，适合本地使用
2. **HTTP 服务器**：支持远程访问，适合团队协作

#### 7.3 RPC 通信机制
**贴合文件**：`src/util/rpc.ts`

**技术难点**：仅 70 行代码实现完整 RPC 系统

**代码伴读**：
```typescript
// src/util/rpc.ts - 极简 RPC 实现
export namespace Rpc {
  // Worker 端：监听请求
  export function listen(rpc: Definition) {
    onmessage = async (evt) => {
      const parsed = JSON.parse(evt.data)
      
      if (parsed.type === "rpc.request") {
        const result = await rpc[parsed.method](parsed.input)
        postMessage(JSON.stringify({
          type: "rpc.result",
          result,
          id: parsed.id
        }))
      }
    }
  }
  
  // 主线程：发送请求
  export function client<T>(target: Worker) {
    const pending = new Map<number, (result: any) => void>()
    const listeners = new Map<string, Set<(data: any) => void>>()
    let id = 0
    
    target.onmessage = (evt) => {
      const parsed = JSON.parse(evt.data)
      
      // 处理 RPC 响应
      if (parsed.type === "rpc.result") {
        const resolve = pending.get(parsed.id)
        if (resolve) {
          resolve(parsed.result)
          pending.delete(parsed.id)
        }
      }
      
      // 处理事件通知
      if (parsed.type === "rpc.event") {
        const handlers = listeners.get(parsed.event)
        if (handlers) {
          for (const handler of handlers) {
            handler(parsed.data)
          }
        }
      }
    }
    
    return {
      // 调用远程方法
      call<Method>(method: Method, input): Promise<ReturnType> {
        const requestId = id++
        return new Promise((resolve) => {
          pending.set(requestId, resolve)
          target.postMessage(JSON.stringify({
            type: "rpc.request",
            method,
            input,
            id: requestId
          }))
        })
      },
      
      // 订阅事件
      on<Data>(event: string, handler: (data: Data) => void) {
        let handlers = listeners.get(event)
        if (!handlers) {
          handlers = new Set()
          listeners.set(event, handlers)
        }
        handlers.add(handler)
        return () => handlers!.delete(handler)
      }
    }
  }
  
  // Worker 端：发送事件
  export function emit(event: string, data: unknown) {
    postMessage(JSON.stringify({
      type: "rpc.event",
      event,
      data
    }))
  }
}
```

**协议设计**：
- `rpc.request` - 调用远程方法
- `rpc.result` - 返回方法结果
- `rpc.event` - 单向事件通知

**练习**：扩展 RPC 协议，添加超时处理。

#### 7.4 键盘绑定系统
**贴合文件**：`src/cli/cmd/tui/component/textarea-keybindings.ts`

**代码伴读**：
```typescript
// Vim 风格快捷键
const keybindings = {
  "Ctrl+c": () => abort(),
  "Ctrl+d": () => scrollDown(),
  "Ctrl+u": () => scrollUp(),
  "Ctrl+p": () => openCommandPalette(),
  "Escape": () => closeDialog()
}
```

**练习**：添加自定义快捷键，实现快速导航。

#### 7.5 实时界面更新
**贴合文件**：`src/cli/cmd/tui/event.ts`

**代码伴读**：
```typescript
// SSE 事件流
const events = await sdk.event.subscribe({}, { signal })

for await (const event of events.stream) {
  switch (event.type) {
    case "session.message.part":
      // 更新消息部分
      break
    case "session.status":
      // 更新会话状态
      break
  }
}
```

**企业级价值**：
- 无 GUI 环境支持（SSH、容器）
- 远程开发体验（HTTP 模式）
- 低资源占用（终端 UI）
- 高性能响应（多线程架构）

**本章小结**：
本章构建了功能完整的终端用户界面，使用 OpenTUI 和 SolidJS 实现了 16+ 个响应式组件。多线程架构将 UI 层和业务逻辑分离，通过 RPC 通信实现高性能交互。仅 70 行代码的 RPC 实现支持方法调用和事件通知。键盘绑定系统提供 Vim 风格的快捷键支持。SSE 事件流实现了实时界面更新。这些功能为用户提供了流畅的终端交互体验。

### 第9章：工具系统扩展（Iteration 9: Advanced Tools）

**迭代目标**：实现可扩展工具调用系统。输出：完整的文件操作、执行能力。

#### 9.1 工具定义框架
**贴合文件**：`src/tool/tool.ts`

**决策还原**：
- **Why Zod 参数验证**：类型安全、运行时验证、自动生成文档
- **Trade-off**：学习成本 vs. 类型安全

**代码伴读**：
```typescript
// src/tool/tool.ts - 工具接口
export namespace Tool {
  export interface Info {
    id: string
    init: (ctx?: { agent?: Agent.Info }) => Promise<{
      parameters: ZodType
      description: string
      execute: (args: any, ctx: Context) => Promise<Result>
    }>
  }
  
  export interface Result {
    title: string
    output: string
    metadata?: Record<string, any>
  }
}
```

**练习**：定义自定义工具，实现参数验证。

#### 9.2 工具注册与动态加载（ToolRegistry）⭐
**贴合文件**：`src/tool/registry.ts`

**决策还原**：
- **Why 动态注册**：扩展性、插件支持、模型适配
- **Trade-off**：复杂度 vs. 灵活性

**代码伴读**：
```typescript
// src/tool/registry.ts - 工具注册表
export namespace ToolRegistry {
  export const state = Instance.state(async () => {
    const custom = [] as Tool.Info[]
    
    // 1. 扫描自定义工具（.opencode/tool/*.ts）
    const glob = new Bun.Glob("{tool,tools}/*.{js,ts}")
    for (const dir of await Config.directories()) {
      for await (const match of glob.scan({ cwd: dir, absolute: true })) {
        const namespace = path.basename(match, path.extname(match))
        const mod = await import(match)
        
        for (const [id, def] of Object.entries<ToolDefinition>(mod)) {
          custom.push(fromPlugin(
            id === "default" ? namespace : `${namespace}_${id}`,
            def
          ))
        }
      }
    }
    
    // 2. 加载插件工具
    const plugins = await Plugin.list()
    for (const plugin of plugins) {
      for (const [id, def] of Object.entries(plugin.tool ?? {})) {
        custom.push(fromPlugin(id, def))
      }
    }
    
    return { custom }
  })
  
  // 插件工具转换
  function fromPlugin(id: string, def: ToolDefinition): Tool.Info {
    return {
      id,
      init: async (initCtx) => ({
        parameters: z.object(def.args),
        description: def.description,
        execute: async (args, ctx) => {
          const result = await def.execute(args, {
            ...ctx,
            directory: Instance.directory,
            worktree: Instance.worktree
          })
          
          // 自动截断输出
          const out = await Truncate.output(result, {}, initCtx?.agent)
          return {
            title: "",
            output: out.truncated ? out.content : result,
            metadata: { 
              truncated: out.truncated, 
              outputPath: out.truncated ? out.outputPath : undefined 
            }
          }
        }
      })
    }
  }
  
  // 获取模型可用工具
  export async function tools(
    model: { providerID: string; modelID: string },
    agent?: Agent.Info
  ) {
    const tools = await all()
    
    return Promise.all(
      tools
        .filter((t) => {
          // Zen 用户或启用标志才能使用 websearch/codesearch
          if (t.id === "codesearch" || t.id === "websearch") {
            return model.providerID === "opencode" || Flag.OPENCODE_ENABLE_EXA
          }
          
          // GPT-4 使用 apply_patch，其他模型使用 edit/write
          const usePatch = model.modelID.includes("gpt-") && 
                          !model.modelID.includes("oss") && 
                          !model.modelID.includes("gpt-4")
          if (t.id === "apply_patch") return usePatch
          if (t.id === "edit" || t.id === "write") return !usePatch
          
          return true
        })
        .map(async (t) => ({
          id: t.id,
          ...(await t.init({ agent }))
        }))
    )
  }
  
  async function all(): Promise<Tool.Info[]> {
    const custom = await state().then((x) => x.custom)
    const config = await Config.get()
    
    return [
      InvalidTool,
      ...(["app", "cli", "desktop"].includes(Flag.OPENCODE_CLIENT) ? [QuestionTool] : []),
      BashTool,
      ReadTool,
      GlobTool,
      GrepTool,
      EditTool,
      WriteTool,
      TaskTool,
      WebFetchTool,
      TodoWriteTool,
      TodoReadTool,
      WebSearchTool,
      CodeSearchTool,
      SkillTool,
      ApplyPatchTool,
      ...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [LspTool] : []),
      ...(config.experimental?.batch_tool === true ? [BatchTool] : []),
      ...(Flag.OPENCODE_EXPERIMENTAL_PLAN_MODE && Flag.OPENCODE_CLIENT === "cli" 
        ? [PlanExitTool, PlanEnterTool] 
        : []),
      ...custom
    ]
  }
}
```

**工具来源**：
1. **内置工具**（15+ 个）
   - `bash` - 命令执行
   - `read` - 文件读取
   - `write` - 文件写入
   - `edit` - 文件编辑
   - `glob` - 文件搜索
   - `grep` - 代码搜索
   - `task` - 子任务
   - `webfetch` - 网页抓取
   - `websearch` - 网页搜索
   - `codesearch` - 代码搜索
   - `skill` - 技能调用
   - `apply_patch` - 补丁应用

2. **插件工具**（Plugin.tool）
   - 通过插件系统注册
   - 支持 MCP 工具

3. **自定义工具**（.opencode/tool/*.ts）
   - 项目级工具
   - 用户级工具

**模型适配策略**：
```typescript
// GPT-4 系列
if (model.modelID.includes("gpt-4")) {
  tools = [...tools, ApplyPatchTool] // 使用 apply_patch
} else {
  tools = [...tools, EditTool, WriteTool] // 使用 edit + write
}

// Zen 用户
if (model.providerID === "opencode") {
  tools = [...tools, WebSearchTool, CodeSearchTool]
}
```

**Insight**：
- 工具动态加载（支持热更新）
- 模型特定工具过滤
- 自动输出截断
- 插件工具无缝集成

**练习**：
1. 开发自定义工具
2. 实现工具权限控制
3. 添加工具使用统计

#### 9.3 内置工具实现
**贴合文件**：`src/tool/{bash,read,write,edit,grep,glob}.ts`

**代码伴读**：
```typescript
// src/tool/bash.ts - 命令执行工具
export const BashTool: Tool.Info = {
  id: "bash",
  init: async () => ({
    parameters: z.object({
      command: z.string().describe("Shell command to execute")
    }),
    description: "Execute shell commands",
    execute: async (args, ctx) => {
      const result = await $`${args.command}`
        .cwd(ctx.directory)
        .nothrow()
      
      return {
        title: `$ ${args.command}`,
        output: result.text(),
        metadata: {
          exitCode: result.exitCode,
          stderr: result.stderr.toString()
        }
      }
    }
  })
}
```

**练习**：实现文件批量操作工具。

#### 9.4 交互式问答与用户确认（Question）⭐
**贴合文件**：`src/question/index.ts`

**决策还原**：
- **Why Promise-based**：异步等待、类型安全、易于集成
- **Trade-off**：阻塞执行 vs. 用户控制

**代码伴读**：
```typescript
// src/question/index.ts - 交互式问答系统
export namespace Question {
  export const Info = z.object({
    question: z.string().describe("Complete question"),
    header: z.string().describe("Very short label (max 30 chars)"),
    options: z.array(Option).describe("Available choices"),
    multiple: z.boolean().optional().describe("Allow multiple selections"),
    custom: z.boolean().optional().describe("Allow custom answer (default: true)")
  })
  
  export const Option = z.object({
    label: z.string().describe("Display text (1-5 words)"),
    description: z.string().describe("Explanation of choice")
  })
  
  const state = Instance.state(async () => {
    const pending: Record<string, {
      info: Request
      resolve: (answers: Answer[]) => void
      reject: (e: any) => void
    }> = {}
    
    return { pending }
  })
  
  // 发起问答
  export async function ask(input: {
    sessionID: string
    questions: Info[]
    tool?: { messageID: string; callID: string }
  }): Promise<Answer[]> {
    const s = await state()
    const id = Identifier.ascending("question")
    
    return new Promise<Answer[]>((resolve, reject) => {
      const info: Request = {
        id,
        sessionID: input.sessionID,
        questions: input.questions,
        tool: input.tool
      }
      
      s.pending[id] = { info, resolve, reject }
      Bus.publish(Event.Asked, info)
    })
  }
  
  // 用户回复
  export async function reply(input: { 
    requestID: string
    answers: Answer[] 
  }): Promise<void> {
    const s = await state()
    const existing = s.pending[input.requestID]
    if (!existing) return
    
    delete s.pending[input.requestID]
    
    Bus.publish(Event.Replied, {
      sessionID: existing.info.sessionID,
      requestID: existing.info.id,
      answers: input.answers
    })
    
    existing.resolve(input.answers)
  }
  
  // 用户拒绝
  export async function reject(requestID: string): Promise<void> {
    const s = await state()
    const existing = s.pending[requestID]
    if (!existing) return
    
    delete s.pending[requestID]
    
    Bus.publish(Event.Rejected, {
      sessionID: existing.info.sessionID,
      requestID: existing.info.id
    })
    
    existing.reject(new RejectedError())
  }
  
  export class RejectedError extends Error {
    constructor() {
      super("The user dismissed this question")
    }
  }
}
```

**使用示例**：
```typescript
// 在工具中使用
const answers = await Question.ask({
  sessionID,
  questions: [{
    question: "Which files should I modify?",
    header: "File Selection",
    options: [
      { label: "src/index.ts", description: "Main entry point" },
      { label: "src/config.ts", description: "Configuration file" },
      { label: "test/index.test.ts", description: "Test file" }
    ],
    multiple: true,
    custom: true
  }],
  tool: { messageID, callID }
})

// answers: [["src/index.ts", "src/config.ts"]]
```

**Insight**：
- Promise-based 异步等待
- 支持多选/单选/自定义输入
- 自动清理待处理问题
- 用户拒绝抛出异常

**练习**：
1. 实现问题超时机制
2. 添加问题历史记录
3. 支持条件问题（基于前一个答案）

**技术难点**：
- 工具动态加载与热更新
- 模型特定工具过滤
- 异步问答流程管理

**企业级价值**：
- 工具扩展性（插件系统）
- 用户交互控制（问答系统）
- 模型适配（工具过滤）
- 自动化能力（内置工具）

**本章小结**：
本章实现了可扩展的工具系统，包括工具注册表、动态加载机制和模型适配策略。ToolRegistry 支持内置工具、插件工具和自定义工具三种来源，自动根据模型能力过滤可用工具。Question 交互式问答系统基于 Promise 实现异步等待，支持多选/单选/自定义输入。这些功能为 AI 代理提供了强大的工具调用能力和用户交互控制。

### 第10章：LSP集成与代码智能（Iteration 10: Code Intelligence）

**迭代目标**：集成语言服务器，提供代码理解能力。输出：智能代码分析。

#### 10.1 LSP 架构设计
**贴合文件**：`src/lsp/index.ts`、`src/lsp/client.ts`、`src/lsp/server.ts`

**决策还原**：
- **Why LSP**：标准化协议、多语言支持、丰富的代码智能功能
- **Why 多服务器架构**：不同语言使用不同服务器，按需启动
- **Trade-off**：服务器启动开销 vs. 代码智能能力

**代码伴读**：
```typescript
// src/lsp/index.ts - LSP 核心管理
export namespace LSP {
  // 状态管理
  const state = Instance.state(
    async () => {
      const clients: LSPClient.Info[] = []
      const servers: Record<string, LSPServer.Info> = {}
      const cfg = await Config.get()

      // 加载所有可用的 LSP 服务器
      for (const server of Object.values(LSPServer)) {
        servers[server.id] = server
      }

      // 过滤实验性服务器
      filterExperimentalServers(servers)

      // 加载用户自定义服务器
      for (const [name, item] of Object.entries(cfg.lsp ?? {})) {
        if (item.disabled) {
          delete servers[name]
          continue
        }
        servers[name] = {
          ...existing,
          id: name,
          root: existing?.root ?? (async () => Instance.directory),
          extensions: item.extensions ?? existing?.extensions ?? [],
          spawn: async (root) => {
            return {
              process: spawn(item.command[0], item.command.slice(1), {
                cwd: root,
                env: { ...process.env, ...item.env }
              }),
              initialization: item.initialization
            }
          }
        }
      }

      return {
        broken: new Set<string>(),
        servers,
        clients,
        spawning: new Map<string, Promise<LSPClient.Info | undefined>>()
      }
    },
    async (state) => {
      // 清理所有客户端
      await Promise.all(state.clients.map((client) => client.shutdown()))
    }
  )

  // 获取文件对应的 LSP 客户端
  async function getClients(file: string) {
    const s = await state()
    const extension = path.parse(file).ext || file
    const result: LSPClient.Info[] = []

    for (const server of Object.values(s.servers)) {
      // 检查文件扩展名是否匹配
      if (server.extensions.length && !server.extensions.includes(extension)) continue

      // 获取项目根目录
      const root = await server.root(file)
      if (!root) continue
      if (s.broken.has(root + server.id)) continue

      // 查找已存在的客户端
      const match = s.clients.find((x) => x.root === root && x.serverID === server.id)
      if (match) {
        result.push(match)
        continue
      }

      // 启动新的 LSP 服务器
      const client = await schedule(server, root, root + server.id)
      if (!client) continue

      result.push(client)
      Bus.publish(Event.Updated, {})
    }

    return result
  }

  // LSP 操作
  export async function definition(input: { file: string; line: number; character: number }) {
    return run(input.file, (client) =>
      client.connection
        .sendRequest("textDocument/definition", {
          textDocument: { uri: pathToFileURL(input.file).href },
          position: { line: input.line, character: input.character }
        })
        .catch(() => null)
    ).then((result) => result.flat().filter(Boolean))
  }

  export async function references(input: { file: string; line: number; character: number }) {
    return run(input.file, (client) =>
      client.connection
        .sendRequest("textDocument/references", {
          textDocument: { uri: pathToFileURL(input.file).href },
          position: { line: input.line, character: input.character },
          context: { includeDeclaration: true }
        })
        .catch(() => [])
    ).then((result) => result.flat().filter(Boolean))
  }

  export async function hover(input: { file: string; line: number; character: number }) {
    return run(input.file, (client) =>
      client.connection
        .sendRequest("textDocument/hover", {
          textDocument: { uri: pathToFileURL(input.file).href },
          position: { line: input.line, character: input.character }
        })
        .catch(() => null)
    )
  }

  export async function workspaceSymbol(query: string) {
    return runAll((client) =>
      client.connection
        .sendRequest("workspace/symbol", { query })
        .then((result: any) => result.filter((x: LSP.Symbol) => kinds.includes(x.kind)))
        .then((result: any) => result.slice(0, 10))
        .catch(() => [])
    ).then((result) => result.flat() as LSP.Symbol[])
  }
}
```

**支持的 LSP 服务器**（15+ 个）：
- TypeScript/JavaScript - `typescript-language-server`
- Python - `pyright` / `ty`（实验性）
- Go - `gopls`
- Rust - `rust-analyzer`
- C/C++ - `clangd`
- C# - `csharp-ls`
- F# - `fsautocomplete`
- Ruby - `rubocop`
- Elixir - `elixir-ls`
- Zig - `zls`
- Swift - `sourcekit-lsp`
- Vue - `vue-language-server`
- ESLint - `vscode-eslint`
- Biome - `biome`
- Oxlint - `oxlint`
- Deno - `deno lsp`

**Insight**：
- 按需启动服务器（延迟加载）
- 自动检测项目根目录（package.json、go.mod 等）
- 支持多个服务器同时运行（如 TypeScript + ESLint）
- 自动下载缺失的服务器（可通过 Flag 禁用）

**练习**：
1. 添加新的 LSP 服务器支持
2. 实现服务器健康检查
3. 添加服务器重启机制

#### 10.2 LSP 客户端实现
**贴合文件**：`src/lsp/client.ts`

**决策还原**：
- **Why vscode-jsonrpc**：成熟的 JSON-RPC 实现、类型安全
- **Why 诊断去抖**：避免频繁更新、提升性能
- **Trade-off**：150ms 延迟 vs. 性能优化

**代码伴读**：
```typescript
// src/lsp/client.ts - LSP 客户端
export namespace LSPClient {
  const DIAGNOSTICS_DEBOUNCE_MS = 150

  export async function create(input: { 
    serverID: string
    server: LSPServer.Handle
    root: string 
  }) {
    // 创建 JSON-RPC 连接
    const connection = createMessageConnection(
      new StreamMessageReader(input.server.process.stdout as any),
      new StreamMessageWriter(input.server.process.stdin as any)
    )

    // 诊断信息缓存
    const diagnostics = new Map<string, Diagnostic[]>()
    
    // 监听诊断通知
    connection.onNotification("textDocument/publishDiagnostics", (params) => {
      const filePath = Filesystem.normalizePath(fileURLToPath(params.uri))
      diagnostics.set(filePath, params.diagnostics)
      
      // 发布诊断事件（去抖处理）
      Bus.publish(Event.Diagnostics, { 
        path: filePath, 
        serverID: input.serverID 
      })
    })

    // 处理服务器请求
    connection.onRequest("workspace/configuration", async () => {
      return [input.server.initialization ?? {}]
    })
    
    connection.onRequest("workspace/workspaceFolders", async () => [{
      name: "workspace",
      uri: pathToFileURL(input.root).href
    }])

    connection.listen()

    // 发送初始化请求
    await withTimeout(
      connection.sendRequest("initialize", {
        rootUri: pathToFileURL(input.root).href,
        processId: input.server.process.pid,
        workspaceFolders: [{
          name: "workspace",
          uri: pathToFileURL(input.root).href
        }],
        initializationOptions: input.server.initialization,
        capabilities: {
          textDocument: {
            synchronization: {
              didOpen: true,
              didChange: true
            },
            publishDiagnostics: {
              versionSupport: true
            }
          }
        }
      }),
      45_000 // 45 秒超时
    )

    await connection.sendNotification("initialized", {})

    // 文件版本管理
    const files: { [path: string]: number } = {}

    return {
      root: input.root,
      serverID: input.serverID,
      connection,
      notify: {
        async open(input: { path: string }) {
          const file = Bun.file(input.path)
          const text = await file.text()
          const extension = path.extname(input.path)
          const languageId = LANGUAGE_EXTENSIONS[extension] ?? "plaintext"

          const version = files[input.path]
          if (version !== undefined) {
            // 文件已打开，发送变更通知
            const next = version + 1
            files[input.path] = next
            await connection.sendNotification("textDocument/didChange", {
              textDocument: {
                uri: pathToFileURL(input.path).href,
                version: next
              },
              contentChanges: [{ text }]
            })
            return
          }

          // 首次打开文件
          await connection.sendNotification("textDocument/didOpen", {
            textDocument: {
              uri: pathToFileURL(input.path).href,
              languageId,
              version: 0,
              text
            }
          })
          files[input.path] = 0
        }
      },
      diagnostics,
      async waitForDiagnostics(input: { path: string }) {
        // 等待诊断结果（带去抖）
        return await withTimeout(
          new Promise<void>((resolve) => {
            let debounceTimer: ReturnType<typeof setTimeout> | undefined
            const unsub = Bus.subscribe(Event.Diagnostics, (event) => {
              if (event.properties.path === normalizedPath) {
                if (debounceTimer) clearTimeout(debounceTimer)
                debounceTimer = setTimeout(() => {
                  unsub()
                  resolve()
                }, DIAGNOSTICS_DEBOUNCE_MS)
              }
            })
          }),
          3000
        )
      },
      async shutdown() {
        connection.end()
        connection.dispose()
        input.server.process.kill()
      }
    }
  }
}
```

**Insight**：
- 使用 JSON-RPC 2.0 协议通信
- 文件版本管理（避免冲突）
- 诊断去抖（150ms）减少事件风暴
- 45 秒初始化超时（处理慢速服务器）
- 自动清理资源（shutdown）

**练习**：
1. 实现增量文件同步
2. 添加连接重试机制
3. 支持服务器能力协商

#### 10.3 LSP 服务器管理
**贴合文件**：`src/lsp/server.ts`

**决策还原**：
- **Why 自动下载**：简化用户配置、开箱即用
- **Why 项目根检测**：正确的工作区配置、准确的代码分析
- **Trade-off**：首次启动慢 vs. 用户体验

**代码伴读**：
```typescript
// src/lsp/server.ts - LSP 服务器定义
export namespace LSPServer {
  export interface Info {
    id: string
    extensions: string[]
    root: RootFunction
    spawn(root: string): Promise<Handle | undefined>
  }

  // 项目根检测函数
  const NearestRoot = (includePatterns: string[], excludePatterns?: string[]): RootFunction => {
    return async (file) => {
      // 排除特定模式（如 deno.json 排除 TypeScript 服务器）
      if (excludePatterns) {
        const excluded = await Filesystem.up({
          targets: excludePatterns,
          start: path.dirname(file),
          stop: Instance.directory
        }).next()
        if (excluded.value) return undefined
      }
      
      // 向上查找项目根标记文件
      const files = Filesystem.up({
        targets: includePatterns,
        start: path.dirname(file),
        stop: Instance.directory
      })
      const first = await files.next()
      if (!first.value) return Instance.directory
      return path.dirname(first.value)
    }
  }

  // TypeScript 服务器
  export const Typescript: Info = {
    id: "typescript",
    root: NearestRoot(
      ["package-lock.json", "bun.lockb", "bun.lock", "pnpm-lock.yaml", "yarn.lock"],
      ["deno.json", "deno.jsonc"] // 排除 Deno 项目
    ),
    extensions: [".ts", ".tsx", ".js", ".jsx", ".mjs", ".cjs", ".mts", ".cts"],
    async spawn(root) {
      // 查找本地 tsserver
      const tsserver = await Bun.resolve("typescript/lib/tsserver.js", Instance.directory)
        .catch(() => {})
      if (!tsserver) return

      const proc = spawn(BunProc.which(), ["x", "typescript-language-server", "--stdio"], {
        cwd: root,
        env: { ...process.env, BUN_BE_BUN: "1" }
      })

      return {
        process: proc,
        initialization: {
          tsserver: { path: tsserver }
        }
      }
    }
  }

  // Python 服务器（Pyright）
  export const Pyright: Info = {
    id: "pyright",
    extensions: [".py", ".pyi"],
    root: NearestRoot([
      "pyproject.toml", 
      "setup.py", 
      "requirements.txt", 
      "Pipfile", 
      "pyrightconfig.json"
    ]),
    async spawn(root) {
      let binary = Bun.which("pyright-langserver")
      const args = []
      
      if (!binary) {
        // 自动安装 Pyright
        const js = path.join(Global.Path.bin, "node_modules", "pyright", "dist", "pyright-langserver.js")
        if (!(await Bun.file(js).exists())) {
          if (Flag.OPENCODE_DISABLE_LSP_DOWNLOAD) return
          await Bun.spawn([BunProc.which(), "install", "pyright"], {
            cwd: Global.Path.bin,
            env: { ...process.env, BUN_BE_BUN: "1" }
          }).exited
        }
        binary = BunProc.which()
        args.push("run", js)
      }
      args.push("--stdio")

      // 检测虚拟环境
      const initialization: Record<string, string> = {}
      const potentialVenvPaths = [
        process.env["VIRTUAL_ENV"],
        path.join(root, ".venv"),
        path.join(root, "venv")
      ].filter((p): p is string => p !== undefined)

      for (const venvPath of potentialVenvPaths) {
        const pythonPath = process.platform === "win32"
          ? path.join(venvPath, "Scripts", "python.exe")
          : path.join(venvPath, "bin", "python")
        if (await Bun.file(pythonPath).exists()) {
          initialization["pythonPath"] = pythonPath
          break
        }
      }

      const proc = spawn(binary, args, {
        cwd: root,
        env: { ...process.env, BUN_BE_BUN: "1" }
      })

      return {
        process: proc,
        initialization
      }
    }
  }

  // Rust 服务器
  export const RustAnalyzer: Info = {
    id: "rust",
    root: async (root) => {
      // 查找 Cargo.toml
      const crateRoot = await NearestRoot(["Cargo.toml", "Cargo.lock"])(root)
      if (crateRoot === undefined) return undefined

      // 向上查找 workspace 根目录
      let currentDir = crateRoot
      while (currentDir !== path.dirname(currentDir)) {
        const cargoTomlPath = path.join(currentDir, "Cargo.toml")
        const cargoTomlContent = await Bun.file(cargoTomlPath).text().catch(() => "")
        if (cargoTomlContent.includes("[workspace]")) {
          return currentDir
        }
        const parentDir = path.dirname(currentDir)
        if (parentDir === currentDir) break
        currentDir = parentDir
        if (!currentDir.startsWith(Instance.worktree)) break
      }

      return crateRoot
    },
    extensions: [".rs"],
    async spawn(root) {
      const bin = Bun.which("rust-analyzer")
      if (!bin) {
        log.info("rust-analyzer not found in path, please install it")
        return
      }
      return {
        process: spawn(bin, { cwd: root })
      }
    }
  }
}
```

**服务器特性**：
1. **自动下载**：首次使用时自动安装（可禁用）
2. **项目根检测**：智能查找 package.json、Cargo.toml 等
3. **虚拟环境检测**：Python 自动检测 .venv、venv
4. **Workspace 支持**：Rust 自动检测 workspace 根目录
5. **跨平台支持**：Windows/macOS/Linux

**Insight**：
- 服务器二进制存储在 `~/.opencode/bin/`
- 使用 Bun 的 `spawn` 启动服务器进程
- 支持自定义初始化选项
- 自动处理平台差异（.exe 后缀等）

**练习**：
1. 添加服务器版本管理
2. 实现服务器更新检查
3. 支持服务器配置文件

#### 10.4 LSP 工具集成
**贴合文件**：`src/tool/lsp.ts`

**代码伴读**：
```typescript
// src/tool/lsp.ts - LSP 工具
export const LspTool = Tool.define("lsp", {
  description: "Perform LSP operations like go-to-definition, find-references, hover, etc.",
  parameters: z.object({
    operation: z.enum([
      "goToDefinition",
      "findReferences",
      "hover",
      "documentSymbol",
      "workspaceSymbol",
      "goToImplementation",
      "prepareCallHierarchy",
      "incomingCalls",
      "outgoingCalls"
    ]).describe("The LSP operation to perform"),
    filePath: z.string().describe("The absolute or relative path to the file"),
    line: z.number().int().min(1).describe("The line number (1-based)"),
    character: z.number().int().min(1).describe("The character offset (1-based)")
  }),
  execute: async (args, ctx) => {
    const file = path.isAbsolute(args.filePath) 
      ? args.filePath 
      : path.join(Instance.directory, args.filePath)

    // 权限检查
    await ctx.ask({
      permission: "lsp",
      patterns: ["*"],
      always: ["*"],
      metadata: {}
    })

    // 检查文件是否存在
    const exists = await Bun.file(file).exists()
    if (!exists) {
      throw new Error(`File not found: ${file}`)
    }

    // 检查是否有可用的 LSP 服务器
    const available = await LSP.hasClients(file)
    if (!available) {
      throw new Error("No LSP server available for this file type.")
    }

    // 打开文件并等待诊断
    await LSP.touchFile(file, true)

    // 执行 LSP 操作
    const position = {
      file,
      line: args.line - 1, // 转换为 0-based
      character: args.character - 1
    }

    const result: unknown[] = await (async () => {
      switch (args.operation) {
        case "goToDefinition":
          return LSP.definition(position)
        case "findReferences":
          return LSP.references(position)
        case "hover":
          return LSP.hover(position)
        case "documentSymbol":
          return LSP.documentSymbol(pathToFileURL(file).href)
        case "workspaceSymbol":
          return LSP.workspaceSymbol("")
        case "goToImplementation":
          return LSP.implementation(position)
        case "prepareCallHierarchy":
          return LSP.prepareCallHierarchy(position)
        case "incomingCalls":
          return LSP.incomingCalls(position)
        case "outgoingCalls":
          return LSP.outgoingCalls(position)
      }
    })()

    const output = result.length === 0
      ? `No results found for ${args.operation}`
      : JSON.stringify(result, null, 2)

    return {
      title: `${args.operation} ${path.relative(Instance.worktree, file)}:${args.line}:${args.character}`,
      metadata: { result },
      output
    }
  }
})
```

**支持的操作**：
- `goToDefinition` - 跳转到定义
- `findReferences` - 查找引用
- `hover` - 悬停信息
- `documentSymbol` - 文档符号
- `workspaceSymbol` - 工作区符号
- `goToImplementation` - 跳转到实现
- `prepareCallHierarchy` - 准备调用层次
- `incomingCalls` - 传入调用
- `outgoingCalls` - 传出调用

**Insight**：
- 1-based 行号转换为 0-based（LSP 协议）
- 自动打开文件并等待诊断
- 权限检查确保安全
- 结果格式化为 JSON

**练习**：
1. 添加代码补全支持
2. 实现代码格式化
3. 支持代码重构操作

**技术难点**：
- 多服务器并发管理
- 诊断事件去抖
- 项目根自动检测
- 服务器自动下载与安装

**企业级价值**：
- 代码质量提升（实时诊断）
- 开发效率优化（智能跳转）
- 多语言支持（15+ 语言）
- 开箱即用（自动安装）

**本章小结**：
本章集成了语言服务器协议（LSP），实现了完整的代码智能功能。LSP 架构支持 15+ 种编程语言，包括 TypeScript、Python、Go、Rust、C/C++、C#、Ruby 等。LSP 客户端使用 JSON-RPC 2.0 协议与服务器通信，实现了诊断去抖、文件版本管理和自动资源清理。LSP 服务器管理支持自动下载、项目根检测和虚拟环境检测。LSP 工具集成提供了 9 种代码智能操作，包括跳转定义、查找引用、悬停信息等。这些功能显著提升了代码质量和开发效率。

### 第11章：MCP协议与插件生态（Iteration 11: Extensibility）

**迭代目标**：实现标准化插件系统。输出：可扩展的AI代理平台。

#### 11.1 MCP协议集成
**贴合文件**：`@modelcontextprotocol/sdk`、`src/mcp/`

**决策还原**：
- **Why MCP**：标准化协议、生态互通、工具共享
- **Trade-off**：协议限制 vs. 标准化收益

**代码伴读**：
```typescript
// src/mcp/index.ts - MCP 服务器管理
export namespace MCP {
  export async function connect(config: McpConfig) {
    const transport = config.command 
      ? new StdioClientTransport({
          command: config.command,
          args: config.args,
          env: config.env
        })
      : new SSEClientTransport(new URL(config.url!))
    
    const client = new Client({
      name: "opencode",
      version: "1.0.0"
    }, {
      capabilities: {
        tools: {},
        prompts: {},
        resources: {}
      }
    })
    
    await client.connect(transport)
    return client
  }
}
```

**练习**：开发自定义 MCP 服务器。

#### 11.2 插件系统架构（Plugin）⭐
**贴合文件**：`src/plugin/index.ts`

**决策还原**：
- **Why 插件系统**：扩展性、隔离性、生态建设
- **Trade-off**：安全风险 vs. 灵活性

**代码伴读**：
```typescript
// src/plugin/index.ts - 插件系统
export namespace Plugin {
  const BUILTIN = [
    "opencode-anthropic-auth@0.0.13",
    "@gitlab/opencode-gitlab-auth@1.3.2"
  ]
  
  const INTERNAL_PLUGINS: PluginInstance[] = [
    CodexAuthPlugin,
    CopilotAuthPlugin
  ]
  
  const state = Instance.state(async () => {
    const hooks: Hooks[] = []
    const input: PluginInput = {
      client: createOpencodeClient({ baseUrl: "http://localhost:4096" }),
      project: Instance.project,
      worktree: Instance.worktree,
      directory: Instance.directory,
      serverUrl: Server.url(),
      $: Bun.$
    }
    
    // 1. 加载内置插件
    for (const plugin of INTERNAL_PLUGINS) {
      const init = await plugin(input)
      hooks.push(init)
    }
    
    // 2. 加载配置的插件
    const plugins = [...(config.plugin ?? []), ...BUILTIN]
    
    for (let plugin of plugins) {
      // 跳过已废弃的插件
      if (plugin.includes("opencode-openai-codex-auth") || 
          plugin.includes("opencode-copilot-auth")) continue
      
      // 安装 NPM 包
      if (!plugin.startsWith("file://")) {
        const lastAtIndex = plugin.lastIndexOf("@")
        const pkg = lastAtIndex > 0 ? plugin.substring(0, lastAtIndex) : plugin
        const version = lastAtIndex > 0 ? plugin.substring(lastAtIndex + 1) : "latest"
        
        plugin = await BunProc.install(pkg, version).catch((err) => {
          Bus.publish(Session.Event.Error, {
            error: new NamedError.Unknown({
              message: `Failed to install plugin ${pkg}@${version}: ${err.message}`
            }).toObject()
          })
          return ""
        })
        
        if (!plugin) continue
      }
      
      // 加载插件模块
      const mod = await import(plugin)
      
      // 防止重复初始化（named export + default export）
      const seen = new Set<PluginInstance>()
      for (const [_name, fn] of Object.entries<PluginInstance>(mod)) {
        if (seen.has(fn)) continue
        seen.add(fn)
        
        const init = await fn(input)
        hooks.push(init)
      }
    }
    
    return { hooks, input }
  })
  
  // 触发插件 Hook
  export async function trigger<Name extends keyof Hooks>(
    name: Name,
    input: Parameters<Hooks[Name]>[0],
    output: Parameters<Hooks[Name]>[1]
  ): Promise<typeof output> {
    if (!name) return output
    
    for (const hook of await state().then((x) => x.hooks)) {
      const fn = hook[name]
      if (!fn) continue
      await fn(input, output)
    }
    
    return output
  }
  
  // 初始化插件
  export async function init() {
    const hooks = await state().then((x) => x.hooks)
    const config = await Config.get()
    
    // 传递配置
    for (const hook of hooks) {
      await hook.config?.(config)
    }
    
    // 订阅事件
    Bus.subscribeAll(async (input) => {
      const hooks = await state().then((x) => x.hooks)
      for (const hook of hooks) {
        hook["event"]?.({ event: input })
      }
    })
  }
}
```

**插件 Hook 类型**：
```typescript
export interface Hooks {
  // 认证 Hook
  auth?: (ctx: AuthContext) => Promise<AuthResult>
  
  // 事件 Hook
  event?: (evt: { event: any }) => void
  
  // 工具 Hook
  tool?: Record<string, ToolDefinition>
  
  // 配置 Hook
  config?: (config: Config) => Promise<void>
  
  // 提示词 Hook
  beforePrompt?: (input, output) => Promise<void>
  afterResponse?: (input, output) => Promise<void>
}
```

**内置插件**：
1. **CodexAuthPlugin** - OpenAI Codex 认证
2. **CopilotAuthPlugin** - GitHub Copilot 认证

**Insight**：
- 自动安装缺失依赖
- 防止重复初始化
- 事件自动转发
- 工具无缝集成

**练习**：
1. 开发认证插件
2. 实现工具插件
3. 添加插件沙箱隔离

#### 11.3 技能系统与知识库（Skill）⭐
**贴合文件**：`src/skill/skill.ts`

**决策还原**：
- **Why Markdown 格式**：易编辑、版本控制友好、支持富文本
- **Trade-off**：解析开销 vs. 易用性

**代码伴读**：
```typescript
// src/skill/skill.ts - 技能系统
export namespace Skill {
  export const Info = z.object({
    name: z.string(),
    description: z.string(),
    location: z.string()
  })
  
  const OPENCODE_SKILL_GLOB = new Bun.Glob("{skill,skills}/**/SKILL.md")
  const CLAUDE_SKILL_GLOB = new Bun.Glob("skills/**/SKILL.md")
  
  export const state = Instance.state(async () => {
    const skills: Record<string, Info> = {}
    
    const addSkill = async (match: string) => {
      // 解析 Markdown frontmatter
      const md = await ConfigMarkdown.parse(match).catch((err) => {
        log.error("failed to load skill", { skill: match, err })
        return undefined
      })
      
      if (!md) return
      
      const parsed = Info.pick({ name: true, description: true }).safeParse(md.data)
      if (!parsed.success) return
      
      // 警告重复技能
      if (skills[parsed.data.name]) {
        log.warn("duplicate skill name", {
          name: parsed.data.name,
          existing: skills[parsed.data.name].location,
          duplicate: match
        })
      }
      
      skills[parsed.data.name] = {
        name: parsed.data.name,
        description: parsed.data.description,
        location: match
      }
    }
    
    // 1. 扫描 .claude/skills/（兼容 Claude Code）
    const claudeDirs = await Array.fromAsync(
      Filesystem.up({
        targets: [".claude"],
        start: Instance.directory,
        stop: Instance.worktree
      })
    )
    
    // 添加全局 ~/.claude/skills/
    const globalClaude = `${Global.Path.home}/.claude`
    if (await Filesystem.isDir(globalClaude)) {
      claudeDirs.push(globalClaude)
    }
    
    if (!Flag.OPENCODE_DISABLE_CLAUDE_CODE_SKILLS) {
      for (const dir of claudeDirs) {
        const matches = await Array.fromAsync(
          CLAUDE_SKILL_GLOB.scan({ cwd: dir, absolute: true })
        )
        for (const match of matches) {
          await addSkill(match)
        }
      }
    }
    
    // 2. 扫描 .opencode/skill/
    for (const dir of await Config.directories()) {
      for await (const match of OPENCODE_SKILL_GLOB.scan({ cwd: dir, absolute: true })) {
        await addSkill(match)
      }
    }
    
    return skills
  })
  
  export async function get(name: string) {
    return state().then((x) => x[name])
  }
  
  export async function all() {
    return state().then((x) => Object.values(x))
  }
}
```

**技能定义示例**：
```markdown
---
name: "typescript-expert"
description: "TypeScript best practices and patterns"
---

# TypeScript Expert Skill

## Guidelines

1. Always use strict mode
2. Prefer type inference over explicit types
3. Use const assertions for literal types
4. Avoid `any` type

## Examples

\`\`\`typescript
// Good
const config = {
  port: 3000,
  host: "localhost"
} as const

// Bad
const config: any = {
  port: 3000,
  host: "localhost"
}
\`\`\`
```

**扫描路径**：
- `.opencode/skill/**/SKILL.md`
- `.opencode/skills/**/SKILL.md`
- `.claude/skills/**/SKILL.md`（兼容 Claude Code）
- `~/.claude/skills/**/SKILL.md`（全局技能）

**Insight**：
- 兼容 Claude Code 技能格式
- 支持项目级和全局技能
- 重复技能警告
- Markdown frontmatter 元数据

**练习**：
1. 创建自定义技能
2. 实现技能搜索功能
3. 添加技能版本管理

#### 11.4 命令系统与自定义工作流（Command）⭐
**贴合文件**：`src/command/index.ts`

**代码伴读**：
```typescript
// src/command/index.ts - 命令系统
export namespace Command {
  export const Event = {
    Executed: BusEvent.define(
      "command.executed",
      z.object({
        commandID: z.string(),
        sessionID: z.string()
      })
    )
  }
  
  export async function list() {
    const commands: Info[] = []
    
    // 扫描 .opencode/command/*.md
    for (const dir of await Config.directories()) {
      const glob = new Bun.Glob("command/*.md")
      for await (const match of glob.scan({ cwd: dir, absolute: true })) {
        const md = await ConfigMarkdown.parse(match)
        const info = Info.parse({
          id: path.basename(match, ".md"),
          ...md.data,
          content: md.content
        })
        commands.push(info)
      }
    }
    
    return commands
  }
  
  export async function execute(commandID: string, context: Context) {
    const commands = await list()
    const command = commands.find((c) => c.id === commandID)
    if (!command) throw new Error(`Command not found: ${commandID}`)
    
    // 执行命令（发送到 AI）
    const session = await Session.create({
      agent: command.agent ?? "build",
      cwd: context.directory
    })
    
    await Session.prompt({
      sessionID: session.id,
      content: command.content
    })
    
    Bus.publish(Event.Executed, {
      commandID,
      sessionID: session.id
    })
    
    return session
  }
}
```

**命令定义示例**：
```markdown
---
name: "refactor-code"
description: "Refactor code following best practices"
agent: "build"
---

Please refactor the selected code following these guidelines:

1. Extract reusable functions
2. Improve variable naming
3. Add type annotations
4. Remove code duplication
```

**练习**：
1. 创建自定义命令
2. 实现命令参数化
3. 添加命令历史记录

#### 11.5 多端支持与生产部署
**贴合文件**：`packages/desktop/`、`packages/app/`、`sdks/vscode/`

**代码伴读**：
```typescript
// packages/desktop/ - Tauri 桌面应用
// packages/app/ - Web 控制台
// sdks/vscode/ - VSCode 扩展
```

**练习**：配置生产环境部署。

**技术难点**：
- 插件安全隔离
- 技能冲突解决
- 命令参数化

**企业级价值**：
- 生态建设（插件系统）
- 知识复用（技能系统）
- 工作流自动化（命令系统）
- 多端支持（桌面/Web/VSCode）

**本章小结**：
本章实现了标准化的插件系统，集成 MCP 协议支持工具共享和生态互通。Plugin 系统支持内置插件、NPM 包插件和本地插件三种来源，自动安装缺失依赖并防止重复初始化。Skill 技能系统使用 Markdown 格式定义知识库，兼容 Claude Code 格式。Command 命令系统支持自定义工作流，实现工作流自动化。多端支持（桌面/Web/VSCode）为用户提供了灵活的使用方式。

## 第三部分：工程篇

### 第12章：测试驱动开发（Iteration 12: Testing）

**迭代目标**：构建完整测试体系。输出：40+ 测试文件，覆盖核心功能。

#### 12.1 Bun Test 框架基础
**贴合文件**：`packages/opencode/test/`、`packages/app/e2e/`

**决策还原**：
- **Why Bun Test**：内置框架、快速执行、TypeScript 原生
- **Why 无 Mock**：测试真实实现、避免测试与实现脱节
- **Trade-off**：测试速度 vs. 真实性（选择真实性）

**代码伴读**：
```typescript
// test/tool/bash.test.ts - 真实测试示例
import { describe, expect, test } from "bun:test"
import { BashTool } from "../../src/tool/bash"

describe("BashTool", () => {
  test("should execute simple commands", async () => {
    await using tmp = await tmpdir()
    
    const tool = await BashTool.init()
    const result = await tool.execute(
      { command: "echo 'hello'" },
      createContext({ directory: tmp })
    )
    
    expect(result.output).toContain("hello")
    expect(result.metadata.exitCode).toBe(0)
  })
  
  test("should handle command errors", async () => {
    const tool = await BashTool.init()
    const result = await tool.execute(
      { command: "exit 1" },
      createContext()
    )
    
    expect(result.metadata.exitCode).toBe(1)
  })
})
```

**Insight**：使用 `await using` 自动清理临时目录，无需手动 cleanup。

**练习**：为自定义工具编写测试，覆盖正常和异常场景。

#### 12.2 单元测试实战
**贴合文件**：`test/tool/`、`test/session/`、`test/permission/`

**测试分类**：
1. **工具测试**（10 个文件）
   - `bash.test.ts` - 命令执行
   - `read.test.ts` - 文件读取
   - `edit.test.ts` - 文件编辑
   - `grep.test.ts` - 代码搜索
   - `truncation.test.ts` - 输出截断

2. **会话测试**（7 个文件）
   - `session.test.ts` - 会话管理
   - `llm.test.ts` - LLM 调用
   - `compaction.test.ts` - 消息压缩
   - `retry.test.ts` - 重试机制

3. **权限测试**（2 个文件）
   - `next.test.ts` - 权限评估
   - `arity.test.ts` - 命令参数检查

**代码伴读**：
```typescript
// test/session/compaction.test.ts - 压缩测试
describe("SessionCompaction", () => {
  test("should trigger compaction at 80% context", async () => {
    const model = {
      limit: { context: 100000 }
    }
    
    const overflow = await SessionCompaction.isOverflow({
      tokens: { input: 70000, output: 15000 },
      model
    })
    
    expect(overflow).toBe(true) // 85000 > 80000 (80%)
  })
})
```

**练习**：添加边界条件测试，验证极端场景。

#### 12.3 E2E 测试（Playwright）
**贴合文件**：`packages/app/e2e/`

**测试场景**（20+ 个）：
- `home.spec.ts` - 首页加载
- `prompt.spec.ts` - 提示词输入
- `session.spec.ts` - 会话管理
- `file-tree.spec.ts` - 文件树操作
- `terminal.spec.ts` - 终端交互

**代码伴读**：
```typescript
// e2e/prompt.spec.ts - E2E 测试
import { test, expect } from "@playwright/test"

test("should send prompt and receive response", async ({ page }) => {
  await page.goto("/")
  
  // 输入提示词
  const input = page.locator('[data-testid="prompt-input"]')
  await input.fill("Hello, OpenCode!")
  await input.press("Enter")
  
  // 等待响应
  await page.waitForSelector('[data-testid="message-assistant"]')
  
  // 验证响应
  const response = page.locator('[data-testid="message-assistant"]')
  await expect(response).toBeVisible()
})
```

**练习**：编写完整用户流程测试，从登录到完成任务。

#### 12.4 测试覆盖率与质量保证
**贴合文件**：`package.json`

**代码伴读**：
```json
{
  "scripts": {
    "test": "bun test",
    "test:coverage": "bun test --coverage",
    "test:e2e": "playwright test"
  }
}
```

**测试策略**：
- 单元测试：覆盖核心逻辑
- 集成测试：验证模块协作
- E2E 测试：保证用户体验

**企业级价值**：
- 代码质量保障
- 重构信心
- 回归测试自动化
- 文档化行为

**本章小结**：
本章构建了完整的测试体系，使用 Bun Test 框架实现 40+ 单元测试文件，覆盖工具、会话、权限等核心功能。坚持"无 Mock 真实测试"的最佳实践，确保测试与实现不脱节。使用 Playwright 实现 20+ E2E 测试场景，保证用户体验。测试覆盖率和质量保证机制为代码重构提供了信心，为持续集成提供了基础。

### 第13章：国际化与本地化（Iteration 13: i18n）

**迭代目标**：实现多语言支持。输出：14 种语言的完整翻译系统。

#### 13.1 i18n 系统设计
**贴合文件**：`packages/app/src/i18n/`

**决策还原**：
- **Why @solid-primitives/i18n**：SolidJS 原生、响应式、类型安全
- **Trade-off**：框架绑定 vs. 开发体验

**支持的语言**（14 种）：
```
ar  - 阿拉伯语      br  - 葡萄牙语（巴西）
da  - 丹麦语        de  - 德语
en  - 英语          es  - 西班牙语
fr  - 法语          ja  - 日语
ko  - 韩语          no  - 挪威语
pl  - 波兰语        ru  - 俄语
zh  - 简体中文      zht - 繁体中文
```

**代码伴读**：
```typescript
// src/i18n/en.ts - 英语翻译
export const dict = {
  "command.category.suggested": "Suggested",
  "command.category.view": "View",
  "command.category.session": "Session",
  "command.category.agent": "Agent",
  "command.category.model": "Model",
  "session.status.busy": "Busy",
  "session.status.idle": "Idle",
  "prompt.placeholder": "Ask OpenCode anything...",
  // ... 100+ 翻译键
}

// src/i18n/zh.ts - 中文翻译
import * as en from "./en"
type Keys = keyof typeof en

export const dict: Record<Keys, string> = {
  "command.category.suggested": "建议",
  "command.category.view": "视图",
  "command.category.session": "会话",
  // ... 对应翻译
}
```

**Insight**：使用 TypeScript 类型确保所有语言的翻译键一致。

#### 13.2 类型安全的翻译
**贴合文件**：`packages/app/src/context/`

**代码伴读**：
```typescript
// 使用翻译
import { useI18n } from "@solid-primitives/i18n"

function CommandPalette() {
  const [t] = useI18n()
  
  return (
    <div>
      <h2>{t("command.category.suggested")}</h2>
      {/* TypeScript 会检查键是否存在 */}
    </div>
  )
}
```

**练习**：添加新的翻译键，更新所有语言文件。

#### 13.3 动态语言切换
**代码伴读**：
```typescript
// 语言切换
import { useI18n } from "@solid-primitives/i18n"

function LanguageSelector() {
  const [, { locale, setLocale }] = useI18n()
  
  return (
    <select 
      value={locale()} 
      onChange={(e) => setLocale(e.target.value)}
    >
      <option value="en">English</option>
      <option value="zh">简体中文</option>
      <option value="ja">日本語</option>
      {/* ... 更多语言 */}
    </select>
  )
}
```

**练习**：实现语言偏好持久化，记住用户选择。

#### 13.4 翻译文件管理
**最佳实践**：
1. 使用命名空间组织翻译键（`command.*`、`session.*`）
2. 保持翻译简洁，避免长句
3. 使用占位符支持动态内容
4. 定期审查未使用的翻译键

**企业级价值**：
- 全球化产品
- 用户体验本地化
- 市场扩展能力

**本章小结**：
本章实现了完整的国际化系统，支持 14 种语言（阿拉伯语、葡萄牙语、丹麦语、德语、英语、西班牙语、法语、日语、韩语、挪威语、波兰语、俄语、简体中文、繁体中文）。使用 @solid-primitives/i18n 提供类型安全的翻译，确保所有语言的翻译键一致。动态语言切换支持实时更新界面。翻译文件管理使用命名空间组织，保持代码整洁。这些功能为产品全球化提供了坚实的基础。

### 第14章：CI/CD与自动化发布（Iteration 14: Automation）

**迭代目标**：构建完整的自动化流程。输出：6 个 GitHub Actions 工作流。

#### 14.1 GitHub Actions 工作流
**贴合文件**：`.github/workflows/`

**工作流列表**：
1. `publish.yml` - 发布到 npm
2. `deploy.yml` - 部署到 Cloudflare
3. `generate.yml` - 代码生成
4. `nix-desktop.yml` - Nix 桌面构建
5. `update-nix-hashes.yml` - Nix 哈希更新
6. `review.yml` - PR 审查

**代码伴读**：
```yaml
# .github/workflows/publish.yml
name: publish

on:
  push:
    branches: [dev]
  workflow_dispatch:

jobs:
  publish:
    runs-on: blacksmith-4vcpu-ubuntu-2404
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Bun
        uses: ./.github/actions/setup-bun
      
      - name: Build
        run: bun run build
      
      - name: Publish
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**练习**：添加自动化测试步骤，失败时阻止发布。

#### 14.2 自动化测试流程
**代码伴读**：
```yaml
# 添加测试步骤
- name: Run tests
  run: bun test

- name: Run E2E tests
  run: bun run test:e2e
```

**练习**：配置测试覆盖率报告，上传到 Codecov。

#### 14.3 版本管理与 Changelog
**贴合文件**：`script/changelog.ts`

**代码伴读**：
```typescript
// script/changelog.ts - 自动生成 Changelog
const commits = await getCommits(from, to)
const grouped = groupByType(commits)

const changelog = `
## ${version}

### Features
${grouped.feat.map(c => `- ${c.message}`).join('\n')}

### Bug Fixes
${grouped.fix.map(c => `- ${c.message}`).join('\n')}
`
```

**练习**：实现语义化版本自动升级。

#### 14.4 多平台打包发布
**贴合文件**：`.github/workflows/nix-desktop.yml`

**代码伴读**：
```yaml
# 多平台构建
strategy:
  matrix:
    os:
      - ubuntu-latest
      - macos-15-intel
      - macos-latest

runs-on: ${{ matrix.os }}

steps:
  - name: Build desktop
    run: nix build .#desktop
```

**企业级价值**：
- 发布流程自动化
- 质量门禁
- 快速迭代
- 多平台支持

**本章小结**：
本章构建了完整的 CI/CD 自动化流程，使用 GitHub Actions 实现 6 个工作流（发布、部署、代码生成、Nix 构建、哈希更新、PR 审查）。自动化测试流程确保代码质量，版本管理和 Changelog 生成实现语义化版本控制。多平台打包发布支持 Linux、macOS（Intel/ARM）三大平台。这些自动化流程显著提升了发布效率和代码质量。

## 第四部分：优化篇

### 第15章：性能优化实战（Iteration 15: Performance）

**迭代目标**：实施 5 阶段性能优化。输出：高性能、可扩展的生产系统。

#### 15.1 性能基线与监控
**贴合文件**：`specs/perf-roadmap.md`

**优化路线图**：
```
Phase 0: 基线 + 特性标志
Phase 1: 停止最严重的卡顿（存储 + 请求风暴）
Phase 2: 限制内存增长（内存驱逐）
Phase 3: 大会话滚动可扩展性（scroll spy）
Phase 4: 易于保持快速（模块化 + 去重）
```

**代码伴读**：
```typescript
// packages/app/src/utils/perf.ts - 性能监控
export function measurePerf(name: string, fn: () => void) {
  const start = performance.now()
  fn()
  const duration = performance.now() - start
  console.log(`[Perf] ${name}: ${duration.toFixed(2)}ms`)
}
```

**练习**：添加性能监控点，识别瓶颈。

#### 15.2 存储优化（Payload Limits）
**贴合文件**：`specs/01-persist-payload-limits.md`

**问题**：大型会话历史导致存储阻塞

**解决方案**：
```typescript
// 检测超大负载
const MAX_PAYLOAD_SIZE = 5 * 1024 * 1024 // 5MB

function persist(key: string, value: any) {
  const serialized = JSON.stringify(value)
  
  if (serialized.length > MAX_PAYLOAD_SIZE) {
    console.warn(`Payload too large: ${key}`)
    // 压缩或分片存储
    return persistLarge(key, value)
  }
  
  localStorage.setItem(key, serialized)
}
```

**练习**：实现图片 base64 剥离，减少存储大小。

#### 15.3 缓存策略（LRU/TTL）
**贴合文件**：`specs/02-cache-eviction.md`

**代码伴读**：
```typescript
// LRU 缓存实现
class LRUCache<K, V> {
  private cache = new Map<K, V>()
  private maxSize: number
  
  constructor(maxSize: number) {
    this.maxSize = maxSize
  }
  
  get(key: K): V | undefined {
    const value = this.cache.get(key)
    if (value !== undefined) {
      // 移到最前面（最近使用）
      this.cache.delete(key)
      this.cache.set(key, value)
    }
    return value
  }
  
  set(key: K, value: V) {
    if (this.cache.size >= this.maxSize) {
      // 删除最久未使用的项
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }
    this.cache.set(key, value)
  }
}

// 应用到文件内容缓存
const fileCache = new LRUCache<string, string>(100)
```

**练习**：添加 TTL 支持，自动过期旧缓存。

#### 15.4 请求节流与防抖
**贴合文件**：`specs/03-request-throttling.md`

**代码伴读**：
```typescript
// 防抖实现
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): T {
  let timer: ReturnType<typeof setTimeout>
  
  return ((...args) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }) as T
}

// 应用到文件搜索
const searchFiles = debounce(async (query: string) => {
  const results = await sdk.file.search({ query })
  setResults(results)
}, 300) // 300ms 防抖
```

**练习**：实现节流（throttle），限制请求频率。

#### 15.5 滚动性能优化
**贴合文件**：`specs/04-scroll-spy-optimization.md`

**问题**：大会话滚动时 `querySelectorAll` 性能差

**解决方案**：
```typescript
// 使用 IntersectionObserver
const observer = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      if (entry.isIntersecting) {
        const messageId = entry.target.getAttribute('data-message-id')
        setActiveMessage(messageId)
      }
    }
  },
  { threshold: 0.5 }
)

// 观察所有消息元素
for (const el of messageElements) {
  observer.observe(el)
}
```

**练习**：添加虚拟滚动，只渲染可见消息。

**企业级价值**：
- 用户体验提升
- 资源成本降低
- 系统可扩展性
- 长期可维护性

**本章小结**：
本章实施了 5 阶段性能优化路线图，从基线监控到模块化重构。存储优化通过 Payload Limits 解决大型会话历史阻塞问题。LRU/TTL 缓存策略减少重复计算和网络请求。请求节流与防抖优化用户交互响应。滚动性能优化使用 IntersectionObserver 替代 querySelectorAll，支持大会话流畅滚动。这些优化显著提升了用户体验和系统可扩展性，为生产环境部署提供了坚实的基础。

## 附录

### A. 配置参考手册
- OpenCode 配置文件完整说明
- MCP 服务器配置示例
- Agent 自定义配置

### B. API 文档索引
- SDK API 参考
- 工具 API 文档
- 插件开发 API

### C. 部署指南
- Nix 部署完整教程
- Docker 容器化部署
- Cloudflare Workers 部署

### D. 故障排除
- 常见问题解答
- 调试技巧
- 日志分析

### E. 社区贡献指南
- 代码风格指南（AGENTS.md）
- PR 提交流程
- Issue 报告模板

---

## 📚 学习资源

### 推荐阅读顺序

**初学者**（第 1-5 章）：
1. 第 1 章：环境搭建
2. 第 2 章：配置系统
3. 第 3 章：AI 集成
4. 第 4 章：工具系统
5. 第 5 章：权限控制

**进阶开发者**（第 6-10 章）：
6. 第 6 章：会话处理
7. 第 7 章：TUI 界面
8. 第 8 章：LSP 集成
9. 第 9 章：MCP 协议
10. 第 10 章：多端部署

**工程实践**（第 11-14 章）：
11. 第 11 章：测试体系
12. 第 12 章：国际化
13. 第 13 章：CI/CD
14. 第 14 章：性能优化

### 配套资源

- **代码仓库**：https://github.com/anomalyco/opencode
- **在线文档**：https://opencode.ai/docs
- **社区讨论**：https://discord.gg/opencode
- **视频教程**：（待补充）

### 实践项目建议

1. **基础项目**：实现自定义 CLI 命令
2. **进阶项目**：开发自定义 Agent
3. **高级项目**：创建 MCP 服务器
4. **综合项目**：构建完整的 AI 编码助手

---

**本书完成后，读者将能够**：
- ✅ 从零搭建完整的 OpenCode 项目（90-95% 功能）
- ✅ 理解 AI 编码代理的核心架构
- ✅ 掌握企业级开发最佳实践
- ✅ 具备性能优化和问题排查能力
- ✅ 能够贡献代码到 OpenCode 社区

**预计学习时间**：
- 快速通读：2-3 周
- 深入学习：6-8 周
- 完整实践：3-4 个月
