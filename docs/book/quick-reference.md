# OpenCode 书籍写作 - 快速参考指南

> **用途**: 编写过程中的快速查阅手册
> **更新日期**: 2026-03-02

---

## 📚 章节速查表

### 基础篇（第1-6章）

| 章节 | 标题 | 核心内容 | 代码量 | 状态 |
|------|------|---------|--------|------|
| 第1章 | 全景架构与设计哲学 | Cursor vs Claude Code, 架构概览 | 500行 | ⏳ 待编写 |
| 第2章 | 环境搭建 | Bun, Monorepo, TypeScript | 500行 | ⏳ 待编写 |
| 第3章 | CLI触发原理 | PATH, 符号链接, Shebang | 300行 | ⏳ 待编写 |
| 第4章 | 最小可用的CLI | yargs, 全局命令 | 200行 | ⏳ 待编写 |
| 第5章 | 配置系统基础 | Zod, JSONC, 多层级配置 | 200行 | ⏳ 待编写 |
| 第6章 | 让AI能说话 | @ai-sdk, 流式调用 | 300行 | ⏳ 待编写 |

### 核心篇（第7-12章）

| 章节 | 标题 | 核心内容 | 代码量 | 状态 |
|------|------|---------|--------|------|
| 第7章 | 多轮对话 | Session管理, 消息存储 | 500行 | ⏳ 待编写 |
| 第8章 | 测试基础 | Bun Test, TDD, AAA模式 | 300行 | ⏳ 待编写 |
| 第9章 | 从文件到数据库 | SQLite, Drizzle ORM, 迁移 | 500行 | ⏳ 待编写 |
| 第10章 | 流式输出 | streamText, AbortController | 200行 | ⏳ 待编写 |
| 第11章 | 工具系统 | Tool Schema, 工具执行 | 400行 | ⏳ 待编写 |
| 第12章 | 权限系统 | 规则匹配, 通配符 | 300行 | ⏳ 待编写 |

### 进阶篇（第13-17章）

| 章节 | 标题 | 核心内容 | 代码量 | 状态 |
|------|------|---------|--------|------|
| 第13章 | 更多工具 | Bash, Edit, Grep | 300行 | ⏳ 待编写 |
| 第14章 | 实例隔离 | Instance, 状态管理 | 300行 | ⏳ 待编写 |
| 第15章 | 事件总线 | EventBus, 发布订阅 | 300行 | ⏳ 待编写 |
| 第16章 | TUI界面 | @opentui, SolidJS | 2000行 | ⏳ 待编写 |
| 第17章 | HTTP Server与SSE | Hono, SSE, CORS | 500行 | ⏳ 待编写 |

### 高级篇（第18-21章）

| 章节 | 标题 | 核心内容 | 代码量 | 状态 |
|------|------|---------|--------|------|
| 第18章 | LSP集成 | 语言服务器协议 | 800行 | ⏳ 待编写 |
| 第19章 | MCP协议 | 模型上下文协议 | 600行 | ⏳ 待编写 |
| 第20章 | 代理系统 | Agent, 子代理, 编排 | 600行 | ⏳ 待编写 |
| 第21章 | 高级测试技术 | 集成测试, Mock, 覆盖率 | 500行 | ⏳ 待编写 |

### 发布篇（第22章）

| 章节 | 标题 | 核心内容 | 代码量 | 状态 |
|------|------|---------|--------|------|
| 第22章 | 优化与发布 | 性能优化, CI/CD | 200行 | ⏳ 待编写 |

**总计**: 22章 = 11,500行代码

---

## 🎯 重点章节详解

### 第5章: 配置系统基础

**为什么重要**: 第6章需要使用配置文件,必须先讲解配置系统原理

**核心概念**:
- 配置优先级: 全局 → 项目 → 环境变量 → 托管
- Zod类型验证
- JSONC格式支持
- 深度合并策略

**实际项目对应**:
```
packages/opencode/src/config/
├── config.ts (53KB)
├── paths.ts (5.5KB)
└── tui.ts (3.7KB)
```

**关键代码**:
```typescript
// 配置加载顺序
const configs = [
  await loadFile(globalPath),      // 1. 全局
  await loadFile(projectPath),     // 2. 项目
  await loadFile(envPath),         // 3. 环境变量
  await loadFile(managedPath),     // 4. 托管 (最高优先级)
]
const merged = configs.reduce((acc, cfg) => mergeDeep(acc, cfg), {})
```

---

### 第8章: 测试基础

**为什么重要**: 测试应该从一开始就引入,而不是等到第21章

**核心概念**:
- Bun Test基础 (describe/it/expect)
- 测试隔离 (临时目录)
- TDD工作流 (Red-Green-Refactor)
- AAA模式 (Arrange-Act-Assert)

**实际项目对应**:
```
packages/opencode/test/
├── config/ (3个测试)
├── session/ (7个测试)
├── util/ (10个测试)
└── ... (总计79个测试文件)
```

**关键代码**:
```typescript
import { describe, it, expect } from "bun:test"
import { tmpdir } from "../fixture/fixture"

describe("Config", () => {
  it("should load config", async () => {
    await using tmp = await tmpdir() // 自动清理
    const config = await Config.load(tmp.path)
    expect(config).toBeDefined()
  })
})
```

---

### 第9章: 从文件到数据库

**为什么重要**: 实际项目使用SQLite,而不是JSON文件

**核心概念**:
- SQLite基础
- Drizzle ORM (类型安全查询)
- 数据库迁移 (版本化Schema)
- 事务支持

**实际项目对应**:
```
packages/opencode/src/storage/
├── db.ts (4.4KB)
├── schema.ts
└── json-migration.ts (14.5KB)

migration/
├── 20240101000000_init/
├── 20240201000000_add_messages/
└── 20240301000000_add_todos/
```

**关键代码**:
```typescript
// 定义Schema
export const SessionTable = sqliteTable("sessions", {
  id: text("id").primaryKey(),
  createdAt: integer("created_at", { mode: "timestamp" }).notNull(),
})

// 类型安全查询
const sessions = await db
  .select()
  .from(SessionTable)
  .orderBy(desc(SessionTable.createdAt))
```

---

### 第17章: HTTP Server与SSE通信

**为什么重要**: 实际项目支持多端(TUI/Web/Desktop),需要HTTP Server

**核心概念**:
- Hono框架
- SSE实时推送
- 事件总线集成
- CORS跨域

**实际项目对应**:
```
packages/opencode/src/server/
├── server.ts (20KB)
└── routes/
    ├── session.ts
    ├── project.ts
    └── config.ts
```

**关键代码**:
```typescript
import { Hono } from "hono"
import { streamSSE } from "hono/streaming"

const app = new Hono()

app.get("/events", (c) => {
  return streamSSE(c, async (stream) => {
    Bus.subscribe((event) => {
      stream.writeSSE({
        data: JSON.stringify(event),
        event: event.type,
      })
    })
  })
})
```

---

## 📝 写作模板

### 章节开头模板

```markdown
# 第X章：章节标题

## X.0 引言：从实际问题说起

[用一个具体场景引入本章主题]

想象你是OpenCode的用户...

本章我们将从第一性原理出发，[核心目标]。更重要的是，你会理解为什么要这样设计，以及在构建自己的[相关工具]时如何做出正确的决策。

## X.1 [第一个主题]

### X.1.1 [子主题]

[内容]
```

### 迭代过程模板

```markdown
### 迭代过程

```
┌──────────────────────────────────────────────────────────────┐
│ 迭代 X.1: [迭代名称]                                           │
│ ──────────────────────────────────────────────────────────── │
│ 新增: [新增内容]                                               │
│ 功能: [实现功能]                                               │
│                                                               │
│ 代码:                                                          │
│ [代码示例]                                                     │
│                                                               │
│ 遇到问题: [遇到的问题]                                         │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ 迭代 X.2: [迭代名称]                                           │
│ ──────────────────────────────────────────────────────────── │
│ [内容]                                                         │
└──────────────────────────────────────────────────────────────┘
```
```

### 决策对比模板

```markdown
### 关键决策点

| 问题 | 方案A | 方案B | 选择 | 理由 |
|------|-------|-------|------|------|
| [问题] | [方案A] | [方案B] | [选择] | [理由] |
```

### 代码示例模板

```markdown
### 关键代码

```typescript
// 文件位置: src/xxx/xxx.ts
// 解决问题: [问题描述]
// 设计原则: [设计原则]

[代码]
```
```

### 常见错误模板

```markdown
### 常见错误与解决方案

#### ❌ 错误1: [错误描述]

```typescript
// 错误做法
[错误代码]
```

**问题**: [问题说明]

✅ **正确做法**:

```typescript
// 正确做法
[正确代码]
```
```

### 章节结尾模板

```markdown
### 本章小结

通过本章,我们学习了:

✅ **[要点1]**: [说明]
✅ **[要点2]**: [说明]
✅ **[要点3]**: [说明]

[总结性陈述]

### 读者练习

1. **基础练习**: [练习1]
2. **进阶练习**: [练习2]
3. **挑战练习**: [练习3]
```

---

## 🎨 写作风格指南

### DO (应该做的)

✅ **使用第一人称复数**: "我们来看看..."
✅ **使用具体场景**: "想象你是OpenCode的新用户..."
✅ **展示演进过程**: 从简单到复杂,从问题到解决方案
✅ **提供决策依据**: 不仅告诉"怎么做",更要说明"为什么"
✅ **代码注释精简**: 只在关键处添加注释
✅ **使用可视化**: 流程图、对比表、迭代框图

### DON'T (不应该做的)

❌ **逐行解释代码**: 读者能看懂基本语法
❌ **堆砌完整文件**: 只展示关键部分
❌ **跳过问题直接给答案**: 要展示思考过程
❌ **使用模糊表述**: "可能"、"大概"、"也许"
❌ **过度使用术语**: 首次出现时要解释
❌ **忽略边界情况**: 要讨论异常处理

---

## 📊 代码示例规范

### 文件路径标注

```typescript
// ✅ 好的示例
// packages/opencode/src/config/config.ts

export namespace Config {
  export async function load() {
    // ...
  }
}
```

```typescript
// ❌ 不好的示例
// 没有文件路径标注

export namespace Config {
  export async function load() {
    // ...
  }
}
```

### 代码长度控制

- **完整示例**: 不超过50行
- **代码片段**: 不超过20行
- **超长代码**: 拆分为多个片段,分别讲解

### 导入语句

```typescript
// ✅ 好的示例 - 包含必要的导入
import { Database } from "bun:sqlite"
import { drizzle } from "drizzle-orm/bun-sqlite"

const db = drizzle({ client: sqlite })
```

```typescript
// ❌ 不好的示例 - 缺少导入
const db = drizzle({ client: sqlite }) // drizzle从哪来?
```

---

## 🔍 实际项目对照表

### 核心模块对照

| 章节 | 实际项目路径 | 文件数 | 代码量 |
|------|-------------|--------|--------|
| 第5章 | `src/config/` | 8个文件 | 53KB |
| 第6章 | `src/provider/` | 5个文件 | - |
| 第7章 | `src/session/` | 15个文件 | - |
| 第8章 | `test/` | 79个文件 | - |
| 第9章 | `src/storage/` | 5个文件 | 27KB |
| 第11-13章 | `src/tool/` | 24个工具 | - |
| 第12章 | `src/permission/` | - | - |
| 第15章 | `src/bus/` | - | - |
| 第16章 | `src/cli/cmd/tui/` | 大量组件 | - |
| 第17章 | `src/server/` | 7个文件 | 20KB |
| 第18章 | `src/lsp/` | - | - |
| 第19章 | `src/mcp/` | - | - |
| 第20章 | `src/agent/` | - | - |

### 依赖对照

| 技术 | 实际依赖 | 讲解章节 |
|------|---------|---------|
| Bun | `bun` | 第2章 |
| Turbo | `turbo` | 第2章 |
| Yargs | `yargs` | 第4章 |
| Zod | `zod` | 第5章 |
| AI SDK | `@ai-sdk/*` | 第6章 |
| SQLite | `bun:sqlite` | 第9章 |
| Drizzle ORM | `drizzle-orm` | 第9章 |
| Hono | `hono` | 第17章 |
| SolidJS | `solid-js` | 第16章 |
| OpenTUI | `@opentui/*` | 第16章 |

---

## ⏱️ 编写时间估算

### 每章预估时间

| 章节类型 | 代码量 | 预估时间 | 说明 |
|---------|--------|---------|------|
| 基础章节 | 200-500行 | 2-3天 | 第1-4章 |
| 核心章节 | 500-1000行 | 3-5天 | 第5-15章 |
| 复杂章节 | 1000-2000行 | 5-7天 | 第16章 |
| 高级章节 | 500-800行 | 3-5天 | 第17-21章 |

### 总体时间规划

- **Phase 1** (第1-6章): 5周 - 基础篇
- **Phase 2** (第7-12章): 6周 - 核心篇
- **Phase 3** (第13-17章): 6周 - 进阶篇
- **Phase 4** (第18-21章): 4周 - 高级篇
- **Phase 5** (第22章): 2周 - 发布篇

**总计**: 23周 (约6个月)

---

## 📋 检查清单

### 章节完成检查

- [ ] 引言部分清晰,引入实际场景
- [ ] 迭代过程完整,展示演进路径
- [ ] 代码示例可运行,包含必要导入
- [ ] 决策对比清晰,说明选择理由
- [ ] 常见错误列举,提供解决方案
- [ ] 实际项目对应,标注文件路径
- [ ] 读者练习设计,难度递进
- [ ] 本章小结完整,总结要点

### 代码质量检查

- [ ] 代码可以直接运行
- [ ] 包含必要的类型定义
- [ ] 错误处理完整
- [ ] 注释精简到位
- [ ] 符合项目代码风格
- [ ] 没有安全漏洞

### 文档质量检查

- [ ] 没有错别字
- [ ] 术语使用一致
- [ ] 格式统一规范
- [ ] 图表清晰易懂
- [ ] 链接有效可访问

---

## 🎯 下一步行动

### 立即开始 (本周)

1. ✅ 完成写作计划调整
2. ⏳ 开始编写第0章
3. ⏳ 准备第1章代码示例

### 短期目标 (1-2周)

1. ⏳ 完成第0-1章
2. ⏳ 搭建代码仓库
3. ⏳ 准备练习题

### 中期目标 (1-3个月)

1. ⏳ 完成第0-5.6章 (包括新增章节)
2. ⏳ 第一轮技术审校
3. ⏳ 收集早期反馈

---

## 📞 资源链接

### 项目资源

- **代码仓库**: https://github.com/opencode-ai/opencode
- **文档**: https://opencode.ai/docs
- **社区**: https://discord.gg/opencode

### 写作资源

- **写作计划**: `docs/book/writing-plan-v2-final.md`
- **调整总结**: `docs/book/writing-plan-v2-summary.md`
- **对比报告**: `docs/book/writing-plan-comparison.md`
- **本参考指南**: `docs/book/quick-reference.md`

### 技术文档

- **Bun**: https://bun.sh/docs
- **Drizzle ORM**: https://orm.drizzle.team/docs
- **Hono**: https://hono.dev/docs
- **AI SDK**: https://sdk.vercel.ai/docs

---

**快速参考指南版本**: v1.0
**最后更新**: 2026-03-02
**下次更新**: 完成第0章后
