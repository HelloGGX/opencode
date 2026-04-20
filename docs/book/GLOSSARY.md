# OpenCode 术语表 (Glossary)

> **使用说明**: 本术语表统一了全书的技术术语翻译，确保中英文对照的一致性。所有章节应遵循此术语表的翻译规范。

---

## 核心概念 (Core Concepts)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **Agent** | 代理 / 智能体 | 系统组件用"代理"，概念讨论用"智能体" | 代理系统 (Agent System)<br/>智能体模型 (Agent Model) |
| **Session** | 会话 | 技术层面的会话管理 | 会话管理 (Session Management)<br/>会话状态 (Session State) |
| **Conversation** | 对话 | 用户层面的交互 | 多轮对话 (Multi-turn Conversation)<br/>对话历史 (Conversation History) |
| **Provider** | 提供商 | AI 模型提供商 | OpenAI 提供商<br/>提供商抽象层 |
| **Tool** | 工具 | 系统工具和工具调用 | 工具系统 (Tool System)<br/>工具调用 (Tool Call) |
| **Command** | 命令 | CLI 命令 | 命令行 (Command Line)<br/>命令系统 (Command System) |
| **Event** | 事件 | 事件驱动架构 | 事件总线 (Event Bus)<br/>事件驱动 (Event-Driven) |
| **Permission** | 权限 | 权限控制系统 | 权限系统 (Permission System)<br/>权限控制 (Permission Control) |
| **Instance** | 实例 | 实例管理 | 实例隔离 (Instance Isolation)<br/>实例管理 (Instance Management) |

---

## 架构与设计 (Architecture & Design)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **Layer** | 层 | 分层架构 | 用户界面层 (UI Layer)<br/>核心业务层 (Core Layer) |
| **Architecture** | 架构 | 系统架构 | 分层架构 (Layered Architecture)<br/>全景架构 (Overall Architecture) |
| **Pattern** | 模式 | 设计模式 | 启动器模式 (Launcher Pattern)<br/>适配器模式 (Adapter Pattern) |
| **Interface** | 接口 | API 接口 | 统一接口 (Unified Interface)<br/>API 接口 (API Interface) |
| **Adapter** | 适配器 | 适配器模式 | 适配器模式 (Adapter Pattern)<br/>提供商适配器 (Provider Adapter) |
| **Context** | 上下文 | 上下文管理 | 上下文管理 (Context Management)<br/>上下文窗口 (Context Window) |
| **Middleware** | 中间件 | 中间件层 | 中间件注册 (Middleware Registration) |
| **Hook** | 钩子 | Git 钩子 | Git 钩子 (Git Hook)<br/>pre-push 钩子 |

---

## 数据与状态 (Data & State)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **State** | 状态 | 状态管理 | 状态机 (State Machine)<br/>状态管理 (State Management)<br/>状态持久化 (State Persistence) |
| **Storage** | 存储 | 存储层 | 存储抽象 (Storage Abstraction)<br/>持久化存储 (Persistent Storage) |
| **Message** | 消息 | 消息处理 | 消息处理 (Message Processing)<br/>消息历史 (Message History) |
| **Config/Configuration** | 配置 | 配置系统 | 配置系统 (Configuration System)<br/>配置文件 (Config File) |
| **Cache** | 缓存 | 缓存机制 | 缓存挂载 (Cache Mounting)<br/>缓存策略 (Cache Strategy) |
| **Snapshot** | 快照 | 快照系统 | 快照系统 (Snapshot System) |

---

## API 与通信 (API & Communication)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **Request** | 请求 | HTTP/API 请求 | HTTP 请求 (HTTP Request)<br/>API 请求 (API Request) |
| **Response** | 响应 | HTTP/API 响应 | API 响应 (API Response)<br/>响应体 (Response Body) |
| **Stream/Streaming** | 流式 | 流式输出 | 流式输出 (Streaming Output)<br/>流式调用 (Streaming Call) |
| **Authentication** | 鉴权 | 系统级认证 | 鉴权机制 (Authentication Mechanism)<br/>鉴权凭证 (Auth Credentials) |
| **Authorization** | 授权 | 权限授权 | 授权控制 (Authorization Control) |
| **Token** | 令牌 / Token | API 令牌 | API 令牌 (API Token)<br/>Bearer Token |
| **Endpoint** | 端点 | API 端点 | API 端点 (API Endpoint) |
| **Protocol** | 协议 | 通信协议 | HTTP 协议<br/>MCP 协议 (Model Context Protocol) |

---

## 开发工具 (Development Tools)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **CLI** | 命令行工具 / CLI | 命令行界面 | CLI 工具 (CLI Tool)<br/>命令行界面 (Command Line Interface) |
| **Monorepo** | Monorepo / 单仓库 | 项目结构 | Monorepo 架构<br/>单仓库管理 |
| **Workspace** | 工作区 | 项目工作区 | 工作区配置 (Workspace Configuration) |
| **Package** | 包 | npm 包 | 包管理器 (Package Manager)<br/>子包 (Sub-package) |
| **Dependency** | 依赖 | 依赖管理 | 依赖安装 (Dependency Installation)<br/>依赖版本 (Dependency Version) |
| **Build** | 构建 | 构建系统 | 构建系统 (Build System)<br/>构建产物 (Build Artifact) |
| **Test** | 测试 | 测试系统 | 测试框架 (Test Framework)<br/>单元测试 (Unit Test) |

---

## 运行时与环境 (Runtime & Environment)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **Runtime** | 运行时 | 运行时环境 | 运行时环境 (Runtime Environment)<br/>Bun 运行时 |
| **Environment** | 环境 | 开发环境 | 开发环境 (Development Environment)<br/>环境变量 (Environment Variable) |
| **Process** | 进程 | 系统进程 | 子进程 (Child Process)<br/>进程管理 (Process Management) |
| **Binary** | 二进制文件 | 可执行文件 | 二进制文件 (Binary File)<br/>二进制产物 (Binary Artifact) |
| **Launcher** | 启动器 | 启动器模式 | 启动器模式 (Launcher Pattern)<br/>启动器脚本 (Launcher Script) |
| **Shebang** | Shebang / 释伴 | 脚本头部 | Shebang 行<br/>#!/usr/bin/env node |

---

## Git 与版本控制 (Git & Version Control)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **Repository** | 仓库 | Git 仓库 | Git 仓库 (Git Repository)<br/>代码仓库 (Code Repository) |
| **Branch** | 分支 | Git 分支 | 主分支 (Main Branch)<br/>开发分支 (Dev Branch) |
| **Commit** | 提交 | Git 提交 | 提交记录 (Commit History)<br/>提交信息 (Commit Message) |
| **Pull Request (PR)** | 拉取请求 / PR | 代码审查 | 拉取请求 (Pull Request)<br/>PR 审查 (PR Review) |
| **Merge** | 合并 | 分支合并 | 合并分支 (Merge Branch)<br/>合并冲突 (Merge Conflict) |
| **Hook** | 钩子 | Git 钩子 | pre-commit 钩子<br/>pre-push 钩子 |

---

## CI/CD 与自动化 (CI/CD & Automation)

| English | 中文 | 使用场景 | 示例 |
|---------|------|---------|------|
| **Workflow** | 工作流 | CI/CD 工作流 | 测试工作流 (Test Workflow)<br/>发布工作流 (Publish Workflow) |
| **Pipeline** | 流水线 | CI/CD 流水线 | CI/CD 流水线<br/>构建流水线 (Build Pipeline) |
| **Action** | Action / 动作 | GitHub Actions | GitHub Actions<br/>自定义 Action (Custom Action) |
| **Job** | 任务 | CI 任务 | 构建任务 (Build Job)<br/>测试任务 (Test Job) |
| **Matrix** | 矩阵 | 矩阵策略 | 矩阵策略 (Matrix Strategy)<br/>多平台矩阵 (Multi-platform Matrix) |
| **Runner** | 运行器 | CI 运行器 | GitHub Runner<br/>自托管运行器 (Self-hosted Runner) |

---

## 特殊术语说明 (Special Terms)

### 1. Agent 的使用区分

- **代理** (Agent): 用于系统组件、技术实现
  - 示例: "代理系统"、"代理执行引擎"、"代理调度"

- **智能体** (Intelligent Agent): 用于概念讨论、理论模型
  - 示例: "智能体模型"、"智能体行为"、"PDA 智能体循环"

### 2. Session vs Conversation

- **会话** (Session): 技术层面的会话管理
  - 示例: "会话管理"、"会话状态"、"会话持久化"

- **对话** (Conversation): 用户层面的交互
  - 示例: "多轮对话"、"对话历史"、"对话上下文"

### 3. Authentication vs Authorization

- **鉴权** (Authentication): 系统级身份验证
  - 示例: "鉴权机制"、"鉴权凭证"、"API 鉴权"

- **授权** (Authorization): 权限授予
  - 示例: "授权控制"、"权限授权"

### 4. Stream/Streaming

- **流式** (Streaming): 统一使用"流式"
  - 示例: "流式输出"、"流式调用"、"流式响应"
  - ❌ 避免: "流输出"、"流调用"

---

## 缩写与简称 (Abbreviations)

| 缩写 | 英文全称 | 中文 |
|------|---------|------|
| **CLI** | Command Line Interface | 命令行界面 |
| **API** | Application Programming Interface | 应用程序接口 |
| **SDK** | Software Development Kit | 软件开发工具包 |
| **HTTP** | Hypertext Transfer Protocol | 超文本传输协议 |
| **SSE** | Server-Sent Events | 服务器推送事件 |
| **RPC** | Remote Procedure Call | 远程过程调用 |
| **LSP** | Language Server Protocol | 语言服务器协议 |
| **MCP** | Model Context Protocol | 模型上下文协议 |
| **TUI** | Text-based User Interface | 文本用户界面 |
| **CI/CD** | Continuous Integration/Continuous Deployment | 持续集成/持续部署 |
| **PR** | Pull Request | 拉取请求 |
| **AVX2** | Advanced Vector Extensions 2 | 高级向量扩展 2 |

---

## 使用原则 (Usage Guidelines)

### 1. 优先使用中文术语
- ✅ 推荐: "配置系统"、"会话管理"、"事件总线"
- ❌ 避免: "Config System"、"Session Management"、"Event Bus"

### 2. 专有名词保持英文
- ✅ 保持: GitHub Actions, Bun, TypeScript, OpenAI
- ❌ 不翻译: "吉特哈布动作"、"邦"、"类型脚本"

### 3. 首次出现时中英对照
- ✅ 推荐: "会话管理 (Session Management)"
- 后续使用: "会话管理"

### 4. 技术文档中的代码相关术语
- 代码中: 保持英文 (如变量名、函数名)
- 文档中: 使用中文翻译

### 5. 避免混用
- ❌ 错误: "Session 管理"、"会话 Management"
- ✅ 正确: "会话管理" 或 "Session Management"

---

## 版本历史 (Version History)

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| v1.0 | 2026-03-03 | 初始版本，基于现有章节内容整理 |

---

## 贡献指南 (Contribution Guidelines)

如果发现术语表中缺失的术语或翻译不当，请：

1. 检查该术语在现有章节中的使用情况
2. 确保翻译符合技术文档的专业性和准确性
3. 提交 PR 更新本术语表
4. 同步更新相关章节中的术语使用

---

**注意**: 本术语表是动态文档，随着书籍内容的扩展会持续更新。所有作者和审校人员应遵循此术语表的规范，确保全书术语的一致性。
