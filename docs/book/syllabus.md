《从零构建 OpenCode：开源 AI 编码代理》  
一本资深工程师视角的实战指南，还原敏捷开发迭代过程
。小节结构：
- **子节目标**：具体输出。
- **决策还原**：Why & Trade-off。
- **代码伴读**：贴合实际文件，Context + Problem + Insight。
- **练习**：读者在fork仓库中实现。
- **技术难点** & **企业级价值**：独立总结。

### 第1章：环境搭建与Monorepo骨架（Iteration 1: Bootstrap）
**迭代目标**：构建最小monorepo环境，运行“Hello World” CLI（基于根package.json、turbo.json、bunfig.toml）。输出：可安装依赖、运行简单脚本的开发环境。
1.1 **Monorepo基础配置**（贴合turbo.json、package.json）：Naive单repo vs. Turbo多包协作（Why：高效构建；Trade-off：学习曲线陡）。代码：初始化pnpm-workspaces。
1.2 **Bun作为包管理器**（贴合bunfig.toml、bun.lock）：Why Bun（极速+TS支持）vs. npm（兼容广但慢）。Insight：trustedDependencies防供应链攻击。
1.3 **Nix与flake配置**（贴合flake.nix）：Naive Docker vs. Nix可复现环境。练习：nix run nixpkgs#opencode。
1.4 **忽略与格式化设置**（贴合.gitignore、.editorconfig）：企业级代码质量基石。
**技术难点**：overrides处理patchedDependencies（patches/目录）。**企业级价值**：大规模团队CI/CD（.github/workflows），避免环境漂移。

### 第2章：核心服务器启动与API基础（Iteration 2: Server Core）
**迭代目标**：实现headless服务器（src/server/、src/index.ts入口），支持基本端点。输出：可运行服务器，响应查询。
2.1 **入口文件初始化**（贴合src/index.ts）：CLI启动流（imports from cli/、config/）。Why：统一入口；Trade-off：vs. 多入口碎片化。
2.2 **Hono框架集成**（贴合provider/依赖，假设Hono在package.json）：Naive Express vs. Hono轻量（Cloudflare兼容）。代码：zod-validator输入验证。
2.3 **环境变量加载**（贴合src/env/）：Insight：sst-env.d.ts类型安全，防配置泄露。
2.4 **基本HTTP路由**（贴合src/server/）：Golden path起点，处理简单AI查询。
**技术难点**：@cloudflare/workers-types跨环境兼容。**企业级价值**：API rate limit预备，SST部署（sst.config.ts）防DDoS。

### 第3章：AI模型提供商抽象与工具调用（Iteration 3: AI Integration）
**迭代目标**：引入模型层（src/provider/），实现基本Tool（src/tool/）。输出：可调用AI的子系统。
3.1 **提供商抽象**（贴合src/provider/）：@ai-sdk统一接口（Why：provider-agnostic，README强调；Trade-off：vs. 硬码OpenAI维护重）。Naive：单模型。
3.2 **工具调用循环**（贴合src/tool/、src/agent/）：Reasoning→Tool Call→Observation。代码：ulid ID生成（src/id/）。
3.3 **本地模型支持**（贴合Ollama提及）：Insight：零代码上传隐私优先。
3.4 **权限预集成**（贴合src/permission/）：ask/allow/deny基础。
**技术难点**：异步循环处理（src/scheduler/）。**企业级价值**：成本控制（rate limit），数据合规（端到端加密预备）。

### 第4章：权限引擎与安全沙箱（Iteration 4: Security Layer）
**迭代目标**：构建Permissions（src/permission/），集成到Tool。输出：安全代理执行。
4.1 **权限模型设计**（贴合src/permission/）：glob匹配、per-agent（Why：企业安全；Trade-off：vs. 无权限风险高）。Naive：无校验。
4.2 **审计日志实现**（贴合src/storage/或logs/）：Insight：remeda工具高并发。
4.3 **确认UI集成**（贴合src/question/）：ask模式用户交互。
4.4 **工具级deny**（贴合src/tool/）：bash glob应用。
**技术难点**：并发审计防死锁。**企业级价值**：RBAC（src/auth/预备），GDPR合规审计。

### 第5章：多代理系统与动态调用（Iteration 5: Multi-Agent）
**迭代目标**：实现primary/subagent（src/agent/），@mention。输出：多代理协作。
5.1 **代理定义模板**（贴合AGENTS.md、src/agent/）：Markdown文件（Why：易维护；Trade-off：vs. 硬码不易扩展）。Naive：单代理。
5.2 **动态调用机制**（贴合src/agent/、src/task/工具）：@general搜索（fuzzysort）。
5.3 **会话管理**（贴合src/session/）：父子session。
5.4 **内置代理实现**（贴合AGENTS.md：build/plan/general）：读写权限差异。
**技术难点**：动态解析（src/cli/代理加载）。**企业级价值**：团队协作（共享session），Agentic Workflow（2026趋势）。

### 第6章：TUI交互层与Keybinding（Iteration 6: Terminal UI）
**迭代目标**：构建TUI（src/cli/、假设console/辅助），支持导航。输出：终端交互。
6.1 **SolidJS集成**（贴合package.json Solid依赖）：响应式UI（Why：轻量；Trade-off：vs. React生态大）。Naive：纯CLI。
6.2 **会话导航**（贴合src/session/）：<Leader>+Left/Right（src/pty/或shell/）。
6.3 **Keybinding自定义**（贴合src/command/）：Insight：opentui焦点（CONTRIBUTING）。
6.4 **高亮与格式**（贴合src/format/）：marked-shiki。
**技术难点**：终端兼容（src/bun/）。**企业级价值**：无痕审计（水印集成），远程控制预备。

### 第7章：多端客户端扩展（Iteration 7: Cross-Platform）
**迭代目标**：添加Desktop/Web（packages/app/假设，sdks/vscode/）。输出：多端访问。
7.1 **Tauri跨平台**（贴合package.json Tauri）：Why：轻量打包；Trade-off：vs. Electron资源重。Naive：只TUI。
7.2 **VSCode扩展**（贴合sdks/vscode/）：LSP桥接。
7.3 **Web Console**（贴合packages/web/假设）：Vite+tailwind。
7.4 **共享UI逻辑**（贴合src/global/）：router集成。
**技术难点**：跨端兼容（src/ide/）。**企业级价值**：SSO（src/auth/），分布式团队远程。

### 第8章：LSP与上下文增强（Iteration 8: Code Intelligence）
**迭代目标**：集成LSP（src/lsp/），Context加载。输出：智能分析。
8.1 **Tree-sitter解析**（贴合parsers-config.ts）：75+语言（Why：高效；Trade-off：vs. 自定义重）。Naive：无Context。
8.2 **自动加载**（贴合src/project/）：worktree上下文（src/worktree/）。
8.3 **Edit/Write工具**（贴合src/tool/）：diff/patch。
8.4 **性能参数**（贴合src/agent/）：maxSteps、temperature调优。
**技术难点**：解析性能（src/scheduler/）。**企业级价值**：代码审查模板，供应链审计。

### 第9章：高级扩展与自定义（Iteration 9: Extensibility）
**迭代目标**：自定义Tool/MCP（src/mcp/、src/plugin/），企业特性。输出：插件子系统。
9.1 **MCP协议**（贴合src/mcp/）：标准化（Why：兼容；Trade-off：vs. 自定义差）。Naive：内置tools。
9.2 **插件系统**（贴合src/plugin/）：@opencode-ai/plugin。
9.3 **本地模型流程**（贴合src/provider/）：Ollama+GGUF。
9.4 **自我迭代**（贴合src/agent/）：用OpenCode改进。
**技术难点**：协议集成（src/bus/）。**企业级价值**：多租户（src/enterprise/假设），成本防范。

### 第10章：生产级交付与优化（Iteration 10: Production Ready）
**迭代目标**：CI/CD（nix/、infra/），Benchmark。输出：部署子系统。
10.1 **Nix/Docker部署**（贴合nix/、Dockerfile）：可复现（Why：vs. Docker曲线陡）。
10.2 **CI脚本**（贴合script/、.github/）：自动化测试。
10.3 **错误/日志处理**（贴合src/storage/、logs/）：监控。
10.4 **基准调优**（贴合src/snapshot/）：reasoningEffort。
**技术难点**：云集成（infra/）。**企业级价值**：Kubernetes-ready，性能优化防幻觉。

**附录**：opencode.json示例、源码索引（src/映射）、部署Checklist、Roadmap。

