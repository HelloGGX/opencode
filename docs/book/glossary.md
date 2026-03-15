# OpenCode 书籍术语表

> **用途**: 统一全书术语翻译，确保一致性
> **更新日期**: 2026-03-15
> **使用方法**: 编写章节时查阅，首次出现需标注英文原文

---

## 一、核心概念术语

### 1.1 Agent 相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Agent | 智能体 | 第1章 | 不翻译，全书统一使用"智能体" |
| Agent-first | 智能体优先 | 第1章 | 产品哲学理念 |
| IDE-first | 编辑器优先 | 第1章 | 产品哲学理念 |
| PDA Paradigm | PDA 范式 | 第1章 | 感知-决策-行动，保留缩写 |
| Perception | 感知 | 第1章 | PDA 中的 P |
| Decision | 决策 | 第1章 | PDA 中的 D |
| Action | 行动 | 第1章 | PDA 中的 A |
| State Machine | 状态机 | 第1章 | |
| Context Window | 上下文窗口 | 第1章 | LLM 的核心约束 |
| Token | Token | 第1章 | 不翻译，技术术语 |

### 1.2 架构相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Monorepo | Monorepo | 第2章 | 不翻译，业界通用 |
| Workspace | 工作区 | 第2章 | |
| Package | 包 | 第2章 | |
| Dependency | 依赖 | 第2章 | |
| CLI (Command Line Interface) | 命令行界面 / CLI | 第1章 | 首次出现写全称，后续可用 CLI |
| TUI (Terminal User Interface) | 终端用户界面 / TUI | 第16章 | 首次出现写全称 |
| SSE (Server-Sent Events) | 服务器推送事件 / SSE | 第17章 | 首次出现写全称 |
| LSP (Language Server Protocol) | 语言服务器协议 / LSP | 第18章 | 首次出现写全称 |
| MCP (Model Context Protocol) | 模型上下文协议 / MCP | 第19章 | 首次出现写全称 |

### 1.3 会话与消息

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Session | 会话 | 第7章 | |
| Message | 消息 | 第7章 | |
| Prompt | 提示词 | 第6章 | |
| System Prompt | 系统提示词 | 第1章 | |
| Context | 上下文 | 第1章 | |
| Conversation | 对话 | 第7章 | |
| Turn | 轮次 | 第7章 | 多轮对话中的单次交互 |

### 1.4 工具与权限

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Tool | 工具 | 第11章 | AI 可调用的能力 |
| Tool Call | 工具调用 | 第11章 | |
| Tool Result | 工具结果 | 第11章 | |
| Permission | 权限 | 第12章 | |
| Rule | 规则 | 第12章 | 权限规则 |
| Ruleset | 规则集 | 第12章 | |
| Pattern | 模式 | 第12章 | 通配符匹配模式 |

---

## 二、系统底层术语

### 2.1 操作系统相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| PATH | PATH 环境变量 | 第3章 | 不翻译 |
| Environment Variable | 环境变量 | 第3章 | |
| Symbolic Link / Symlink | 符号链接 | 第3章 | |
| Shebang / Hashbang | Shebang | 第3章 | 不翻译，技术术语 |
| Process | 进程 | 第3章 | |
| Spawn | 衍生 | 第3章 | 创建子进程 |
| execve | execve | 第3章 | 系统调用，不翻译 |
| stdin / stdout / stderr | 标准输入/输出/错误 | 第3章 | 可用缩写 |

### 2.2 包管理相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Package Manager | 包管理器 | 第3章 | |
| Registry | 注册表 / 源 | 第2章 | npm registry |
| Lock File | 锁文件 | 第2章 | |
| Hoisting | 提升 | 第3章 | 依赖提升机制 |
| Transitive Dependency | 传递依赖 | 第2章 | |
| Peer Dependency | 同级依赖 | 第2章 | |

### 2.3 编译与运行时

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Runtime | 运行时 | 第2章 | |
| Compile | 编译 | 第2章 | |
| Transpile | 转译 | 第2章 | |
| Bundle | 打包 | 第2章 | |
| JIT (Just-In-Time) | 即时编译 | 第3章 | |
| Cold Start | 冷启动 | 第3章 | |

---

## 三、AI 与 LLM 术语

### 3.1 模型相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| LLM (Large Language Model) | 大语言模型 / LLM | 第1章 | 首次出现写全称 |
| Model | 模型 | 第6章 | |
| Provider | 提供商 | 第6章 | AI 服务提供商 |
| API Key | API Key | 第6章 | 不翻译 |
| Embedding | 嵌入 | - | 向量表示 |
| Temperature | 温度 | 第6章 | 生成随机性参数 |

### 3.2 生成相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Generate | 生成 | 第6章 | |
| Stream | 流式 | 第6章 | |
| Streaming | 流式输出 | 第10章 | |
| Chunk | 分块 / 片段 | 第10章 | 流式输出的数据块 |
| Token | Token | 第1章 | 不翻译 |
| Completion | 补全 | 第6章 | |
| Response | 响应 | 第6章 | |

### 3.3 AI SDK 相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| generateText | generateText | 第6章 | 函数名，不翻译 |
| streamText | streamText | 第10章 | 函数名，不翻译 |
| AbortController | AbortController | 第10章 | API 名，不翻译 |
| AbortSignal | AbortSignal | 第10章 | API 名，不翻译 |

---

## 四、配置与存储术语

### 4.1 配置相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Config / Configuration | 配置 | 第5章 | |
| Schema | Schema / 模式 | 第5章 | Zod schema 保留英文 |
| JSONC | JSONC | 第5章 | JSON with Comments，不翻译 |
| Override | 覆盖 | 第5章 | |
| Merge | 合并 | 第5章 | |
| Priority | 优先级 | 第5章 | |
| Managed Config | 托管配置 | 第5章 | 企业部署场景 |

### 4.2 存储相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Storage | 存储 | 第9章 | |
| Persist / Persistence | 持久化 | 第7章 | |
| Database | 数据库 | 第9章 | |
| Schema Migration | Schema 迁移 | 第9章 | |
| Query | 查询 | 第9章 | |
| Transaction | 事务 | 第9章 | |

---

## 五、代码与工程术语

### 5.1 代码相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Source Code | 源码 | 第2章 | |
| Entry Point | 入口 | 第3章 | |
| Module | 模块 | 第2章 | |
| Import | 导入 | 第2章 | |
| Export | 导出 | 第2章 | |
| Type Definition | 类型定义 | 第3章 | |
| Type Inference | 类型推断 | 第3章 | |

### 5.2 测试相关

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Test | 测试 | 第8章 | |
| Unit Test | 单元测试 | 第8章 | |
| Integration Test | 集成测试 | 第21章 | |
| Mock | Mock | 第8章 | 不翻译 |
| Fixture | Fixture | 第8章 | 测试固件，不翻译 |
| AAA Pattern | AAA 模式 | 第8章 | Arrange-Act-Assert |
| TDD (Test-Driven Development) | 测试驱动开发 | 第8章 | |

### 5.3 工程实践

| 英文 | 中文翻译 | 首次定义章节 | 备注 |
|------|---------|-------------|------|
| Refactor | 重构 | - | |
| Debug | 调试 | - | |
| Lint | Lint | 第2章 | 不翻译 |
| Format | 格式化 | - | |
| CI/CD | CI/CD | 第22章 | 不翻译 |

---

## 六、保持英文不翻译的术语

以下术语在全书保持英文，不进行翻译：

### 6.1 技术术语

- Agent
- Token
- Shebang
- PATH
- Monorepo
- JSONC
- Mock
- Fixture
- Lint
- CI/CD
- API Key

### 6.2 API / 函数名

- generateText
- streamText
- AbortController
- AbortSignal
- process.argv
- process.env
- child_process.spawn

### 6.3 文件名 / 路径

- package.json
- tsconfig.json
- bunfig.toml
- .env
- .gitignore

### 6.4 命令名

- bun
- npm
- turbo
- yargs

---

## 七、易混淆术语辨析

### 7.1 会话相关

| 术语 | 含义 | 使用场景 |
|------|------|---------|
| Session | 会话 | 一次完整的交互过程，包含多条消息 |
| Conversation | 对话 | 强调交互行为本身 |
| Message | 消息 | 单条用户输入或 AI 输出 |
| Turn | 轮次 | 一问一答构成一个轮次 |

### 7.2 配置相关

| 术语 | 含义 | 使用场景 |
|------|------|---------|
| Global Config | 全局配置 | 用户级默认配置 |
| Project Config | 项目配置 | 项目级配置 |
| Managed Config | 托管配置 | 企业强制配置 |
| Environment Variable | 环境变量 | 进程级配置 |

### 7.3 进程相关

| 术语 | 含义 | 使用场景 |
|------|------|---------|
| Process | 进程 | 操作系统层面的执行单元 |
| Thread | 线程 | 进程内的执行单元 |
| Child Process | 子进程 | 由父进程创建的进程 |
| Spawn | 衍生 | 创建子进程的动作 |

---

## 八、首次出现标注规范

### 8.1 标注格式

当术语首次出现时，采用以下格式：

```markdown
智能体（Agent）是...
```

或

```markdown
Agent（智能体）是...
```

### 8.2 选择原则

- **中文优先**：如果中文翻译是主要用法，先写中文，括号标注英文
- **英文优先**：如果英文是主要用法（如 Agent、Token），先写英文，括号标注中文
- **仅英文**：对于不翻译的术语，直接使用英文

### 8.3 示例

```markdown
# 中文优先示例
会话（Session）是 OpenCode 管理对话状态的核心单元。

# 英文优先示例
Agent（智能体）的核心能力在于自主决策与执行。

# 仅英文示例
Token 消耗是成本控制的关键指标。
```

---

## 九、术语使用检查清单

编写章节时，请对照以下清单检查：

- [ ] 新术语首次出现时是否标注了英文原文
- [ ] 同一术语在全文中的翻译是否一致
- [ ] 是否使用了本表规定的不翻译术语
- [ ] 易混淆术语是否使用正确
- [ ] 技术术语是否使用了业界通用翻译

---

## 十、现有章节术语一致性检查

### 10.1 已发现的问题

| 章节 | 术语 | 当前用法 | 应修正为 | 状态 |
|------|------|---------|---------|------|
| 第1章 | Agent | 混用"Agent"和"智能体" | 统一使用"Agent"，首次出现标注"Agent（智能体）" | ✅ 已修正 |
| 第1章 | System Prompt | 直接使用英文 | "系统提示词（System Prompt）" | ✅ 已修正 |
| 第1章 | Context Window | 直接使用英文 | "上下文窗口（Context Window）" | ✅ 已修正 |
| 第3章 | 符号链接 | 使用"符号链接（Symbolic Link）" | ✅ 正确 | ✅ 已符合 |
| 第6章 | 会话 | 使用"会话（Session）" | ✅ 正确 | ✅ 已符合 |

### 10.2 修正优先级

**高优先级**（影响读者理解）：
1. Agent / 智能体 的统一
2. Prompt / 提示词 的统一

**中优先级**（影响专业性）：
1. Context Window / 上下文窗口 的统一
2. System Prompt / 系统提示词 的统一

---

## 十一、术语更新日志

| 日期 | 术语 | 变更内容 | 原因 |
|------|------|---------|------|
| 2026-03-15 | 初始版本 | 建立术语表 | 统一全书术语 |
| 2026-03-15 | Agent | 明确不翻译，首次标注中文 | 发现第1章混用 |
| 2026-03-15 | Prompt | 明确翻译为"提示词" | 发现第1章混用 |

---

**维护说明**：
1. 新增术语时请更新本表
2. 发现不一致时请在"现有章节术语一致性检查"中记录
3. 每章完成后检查术语一致性
4. 修正完成后更新状态为 ✅
