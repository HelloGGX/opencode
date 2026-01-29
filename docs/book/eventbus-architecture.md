# EventBus 事件驱动架构详解

**OpenCode 核心设计模式之一**

---

## 📊 架构概览

EventBus 是 OpenCode 的**核心通信机制**，实现了模块间的松耦合通信。

### 核心组件

```
src/bus/
├── index.ts          # Bus 核心实现（发布/订阅）
├── bus-event.ts      # 事件定义框架
└── global.ts         # 跨实例通信（GlobalBus）
```

### 设计特点

- ✅ **类型安全**：基于 Zod 的事件 payload 验证
- ✅ **实例隔离**：本地 Bus 自动清理，防止内存泄漏
- ✅ **跨实例通信**：GlobalBus 支持多项目协作
- ✅ **通配符订阅**：`subscribeAll("*")` 监听所有事件
- ✅ **SSE 集成**：实时推送到 Web/Desktop 客户端

---

## 🎯 核心 API

### 1. 定义事件

```typescript
// src/bus/bus-event.ts
export namespace BusEvent {
  export function define<Type extends string, Properties extends ZodType>(
    type: Type,
    properties: Properties
  ) {
    return { type, properties }
  }
}

// 使用示例
export const Event = {
  Created: BusEvent.define(
    "session.created",
    z.object({
      sessionID: z.string(),
      info: Session.Info
    })
  )
}
```

### 2. 发布事件

```typescript
// src/bus/index.ts
export async function publish<Definition extends BusEvent.Definition>(
  def: Definition,
  properties: z.output<Definition["properties"]>
) {
  const payload = { type: def.type, properties }
  
  // 1. 通知本地订阅者
  for (const key of [def.type, "*"]) {
    const subscribers = state().subscriptions.get(key)
    for (const sub of subscribers ?? []) {
      await sub(payload)
    }
  }
  
  // 2. 通知全局订阅者（跨实例）
  GlobalBus.emit("event", {
    directory: Instance.directory,
    payload
  })
}

// 使用示例
await Bus.publish(Session.Event.Created, {
  sessionID: "abc123",
  info: session
})
```

### 3. 订阅事件

```typescript
// 订阅特定事件
const unsub = Bus.subscribe(Session.Event.Updated, async (evt) => {
  console.log("Session updated:", evt.properties.info)
})

// 订阅所有事件
const unsubAll = Bus.subscribeAll(async (event) => {
  console.log("Event:", event.type, event.properties)
})

// 取消订阅
unsub()
```

---

## 📋 事件清单（30+ 个）

### 会话事件（Session）

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `session.created` | 创建新会话 | `{ sessionID, info }` |
| `session.updated` | 更新会话信息 | `{ sessionID, info }` |
| `session.deleted` | 删除会话 | `{ sessionID, info }` |
| `session.error` | 会话错误 | `{ sessionID, error }` |
| `session.diff` | 会话差异计算 | `{ sessionID, diff }` |
| `session.compacted` | 消息压缩完成 | `{ sessionID }` |
| `session.status` | 会话状态变更 | `{ sessionID, status }` |

### 消息事件（Message）

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `message.updated` | 更新消息 | `{ sessionID, info }` |
| `message.removed` | 删除消息 | `{ sessionID, messageID }` |
| `message.part.updated` | 更新消息部分 | `{ part, delta? }` |
| `message.part.removed` | 删除消息部分 | `{ sessionID, messageID, partID }` |

### 权限事件（Permission）

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `permission.asked` | 请求权限 | `{ sessionID, requestID, info }` |
| `permission.replied` | 权限响应 | `{ sessionID, requestID, action }` |

### 文件事件（File）

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `file.edited` | 文件编辑 | `{ file }` |
| `file.watcher.updated` | 文件变更 | `{ file, event }` |

### 命令事件（Command）

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `command.executed` | 命令执行 | `{ name, sessionID }` |

### TUI 事件（TuiEvent）

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `tui.prompt.append` | 追加提示词 | `{ text }` |
| `tui.command.execute` | 执行命令 | `{ command }` |
| `tui.toast.show` | 显示提示 | `{ title, message, variant }` |
| `tui.session.select` | 选择会话 | `{ sessionID }` |

### 其他事件

| 事件类型 | 触发时机 | Payload |
|---------|---------|---------|
| `todo.updated` | 待办更新 | `{ sessionID, todos }` |
| `vcs.branch.updated` | Git 分支切换 | `{ branch }` |
| `installation.update.available` | 版本更新 | `{ version }` |
| `server.instance.disposed` | 实例销毁 | `{ directory }` |

---

## 🔄 典型使用场景

### 场景 1：会话状态同步

**问题**：会话创建后需要同步到云端、更新 UI、记录日志

**解决方案**：使用 EventBus 解耦

```typescript
// src/session/index.ts - 发布者
export async function create(input: CreateInput) {
  const session = { /* ... */ }
  
  // 保存到本地存储
  await Storage.write(["session", projectID, session.id], session)
  
  // 发布事件（一次发布，多处响应）
  await Bus.publish(Event.Created, {
    sessionID: session.id,
    info: session
  })
  
  return session
}

// src/share/share.ts - 订阅者 1：云端同步
Bus.subscribe(Session.Event.Created, async (evt) => {
  await syncToCloud(evt.properties.info)
})

// src/server/server.ts - 订阅者 2：实时推送到客户端
Bus.subscribeAll(async (event) => {
  await stream.writeSSE({
    data: JSON.stringify(event)
  })
})

// src/util/log.ts - 订阅者 3：日志记录
Bus.subscribeAll(async (event) => {
  log.info("Event:", event.type, event.properties)
})
```

**优势**：
- ✅ 发布者无需知道订阅者
- ✅ 新增订阅者无需修改发布者
- ✅ 易于测试和调试

---

### 场景 2：文件变更自动格式化

**问题**：文件编辑后自动格式化，但不影响编辑逻辑

**解决方案**：

```typescript
// src/tool/write.ts - 发布文件编辑事件
export const WriteTool = Tool.define("write", {
  async execute(params, ctx) {
    await Bun.write(filepath, params.content)
    
    // 发布事件
    await Bus.publish(File.Event.Edited, {
      file: filepath
    })
    
    return { title: "File written", output: filepath }
  }
})

// src/format/index.ts - 订阅并格式化
export function init() {
  Bus.subscribe(File.Event.Edited, async (payload) => {
    const file = payload.properties.file
    
    // 检查是否需要格式化
    if (!shouldFormat(file)) return
    
    // 自动格式化
    await formatFile(file)
  })
}
```

**优势**：
- ✅ 格式化逻辑与编辑逻辑分离
- ✅ 可以轻松禁用格式化功能
- ✅ 支持多个格式化器

---

### 场景 3：实时 UI 更新（SSE）

**问题**：Web/Desktop 客户端需要实时接收服务器事件

**解决方案**：

```typescript
// src/server/server.ts - SSE 端点
app.get("/event/subscribe", async (c) => {
  return streamSSE(c, async (stream) => {
    // 订阅所有事件
    const unsub = Bus.subscribeAll(async (event) => {
      await stream.writeSSE({
        data: JSON.stringify(event)
      })
    })
    
    // 连接关闭时取消订阅
    c.req.raw.signal.addEventListener("abort", () => {
      unsub()
    })
  })
})

// 客户端 - 接收事件
const events = await sdk.event.subscribe()

for await (const event of events.stream) {
  switch (event.type) {
    case "session.message.part":
      updateUI(event.properties.part)
      break
    case "session.status":
      updateStatus(event.properties.status)
      break
  }
}
```

**优势**：
- ✅ 实时双向通信
- ✅ 自动重连机制
- ✅ 类型安全的事件处理

---

### 场景 4：跨实例通信（GlobalBus）

**问题**：多个 OpenCode 实例需要协作（如多项目工作区）

**解决方案**：

```typescript
// src/bus/global.ts - 全局事件总线
export const GlobalBus = new EventEmitter<{
  event: [{
    directory?: string
    payload: any
  }]
}>()

// src/bus/index.ts - 发布到全局
export async function publish(def, properties) {
  // ... 本地发布
  
  // 发布到全局（跨实例）
  GlobalBus.emit("event", {
    directory: Instance.directory,
    payload: { type: def.type, properties }
  })
}

// src/cli/cmd/tui/worker.ts - 订阅全局事件
GlobalBus.on("event", (event) => {
  // 转发到 RPC 客户端
  Rpc.emit("global.event", event)
})
```

**优势**：
- ✅ 多实例协作
- ✅ 统一的事件接口
- ✅ 实例隔离与通信并存

---

## 🏗️ 架构设计

### 分层架构

```
┌─────────────────────────────────────────────┐
│              应用层                          │
│  (Session, Tool, Permission, etc.)          │
└──────────────────┬──────────────────────────┘
                   │ publish/subscribe
┌──────────────────▼──────────────────────────┐
│           EventBus 层                        │
│  ┌──────────────────────────────────────┐  │
│  │  Local Bus (Instance-scoped)         │  │
│  │  - 类型安全的发布/订阅                │  │
│  │  - 自动清理（dispose）                │  │
│  │  - 通配符订阅                         │  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │  Global Bus (Cross-instance)         │  │
│  │  - 跨实例通信                         │  │
│  │  - EventEmitter 实现                  │  │
│  └──────────────────────────────────────┘  │
└──────────────────┬──────────────────────────┘
                   │ emit
┌──────────────────▼──────────────────────────┐
│           传输层                             │
│  - SSE (Server-Sent Events)                 │
│  - RPC (Worker Communication)               │
│  - WebSocket (未来扩展)                     │
└─────────────────────────────────────────────┘
```

### 生命周期管理

```typescript
// 实例创建时
const instance = Instance.create(directory)

// 自动初始化 Bus
const state = {
  subscriptions: new Map<string, Subscription[]>()
}

// 实例销毁时
Instance.dispose(directory)

// 自动清理订阅
for (const wildcard of state.subscriptions.get("*") ?? []) {
  wildcard({
    type: "server.instance.disposed",
    properties: { directory }
  })
}
```

---

## 🎨 最佳实践

### 1. 事件命名规范

```typescript
// ✅ 好的命名：模块.资源.动作
"session.created"
"message.part.updated"
"permission.asked"

// ❌ 不好的命名
"create_session"
"update"
"event1"
```

### 2. Payload 设计

```typescript
// ✅ 好的 Payload：包含足够信息
z.object({
  sessionID: z.string(),
  info: Session.Info,
  metadata: z.record(z.any()).optional()
})

// ❌ 不好的 Payload：信息不足
z.object({
  id: z.string()
})
```

### 3. 订阅管理

```typescript
// ✅ 好的实践：保存取消订阅函数
const unsub = Bus.subscribe(Event.Updated, handler)

// 组件卸载时取消订阅
onCleanup(() => {
  unsub()
})

// ❌ 不好的实践：忘记取消订阅（内存泄漏）
Bus.subscribe(Event.Updated, handler)
```

### 4. 错误处理

```typescript
// ✅ 好的实践：捕获订阅者错误
Bus.subscribe(Event.Updated, async (evt) => {
  try {
    await processEvent(evt)
  } catch (error) {
    log.error("Event handler failed", { error })
  }
})

// ❌ 不好的实践：让错误传播
Bus.subscribe(Event.Updated, async (evt) => {
  await processEvent(evt) // 可能抛出错误
})
```

---

## 🔍 调试技巧

### 1. 事件日志

```typescript
// 订阅所有事件并记录
Bus.subscribeAll((event) => {
  console.log(`[Event] ${event.type}`, event.properties)
})
```

### 2. 事件追踪

```typescript
// 添加事件追踪中间件
const originalPublish = Bus.publish

Bus.publish = async (def, properties) => {
  console.time(`Event: ${def.type}`)
  await originalPublish(def, properties)
  console.timeEnd(`Event: ${def.type}`)
}
```

### 3. 事件重放

```typescript
// 记录所有事件
const eventLog: any[] = []

Bus.subscribeAll((event) => {
  eventLog.push({
    timestamp: Date.now(),
    event
  })
})

// 重放事件（用于调试）
async function replay() {
  for (const { event } of eventLog) {
    await Bus.publish(event.type, event.properties)
  }
}
```

---

## 📊 性能考虑

### 1. 订阅者数量

- ✅ 每个事件类型：< 10 个订阅者
- ⚠️ 通配符订阅：谨慎使用，影响性能

### 2. 事件频率

- ✅ 低频事件：会话创建、删除
- ⚠️ 高频事件：消息部分更新（流式输出）
  - 使用 `delta` 增量更新
  - 考虑批量发送

### 3. Payload 大小

- ✅ 小 Payload：< 1KB
- ⚠️ 大 Payload：> 100KB
  - 考虑只发送引用（ID）
  - 订阅者按需加载完整数据

---

## 🎯 企业级价值

### 1. 可扩展性

- ✅ 新增功能无需修改现有代码
- ✅ 插件系统基于事件驱动
- ✅ 微服务架构的基础

### 2. 可测试性

```typescript
// 测试事件发布
test("should publish session created event", async () => {
  const events: any[] = []
  
  Bus.subscribe(Session.Event.Created, (evt) => {
    events.push(evt)
  })
  
  await Session.create({ /* ... */ })
  
  expect(events).toHaveLength(1)
  expect(events[0].properties.sessionID).toBeDefined()
})
```

### 3. 可观测性

- ✅ 事件日志记录
- ✅ 性能监控
- ✅ 错误追踪
- ✅ 审计日志

### 4. 实时性

- ✅ SSE 实时推送
- ✅ 多客户端同步
- ✅ 协作编辑基础

---

## 📚 相关资源

### 代码位置

- `src/bus/index.ts` - Bus 核心实现
- `src/bus/bus-event.ts` - 事件定义框架
- `src/bus/global.ts` - 全局事件总线

### 使用示例

- `src/session/index.ts` - 会话事件
- `src/share/share.ts` - 云端同步
- `src/format/index.ts` - 文件格式化
- `src/server/server.ts` - SSE 推送

### 测试

- `test/session/session.test.ts` - 会话事件测试
- `test/permission/next.test.ts` - 权限事件测试

---

## 🎓 学习建议

### 初学者

1. 理解发布/订阅模式
2. 学习事件定义（BusEvent.define）
3. 实践简单的事件发布和订阅

### 进阶开发者

1. 掌握跨实例通信（GlobalBus）
2. 实现 SSE 实时推送
3. 优化高频事件性能

### 架构师

1. 设计事件驱动架构
2. 实现事件溯源（Event Sourcing）
3. 构建 CQRS 模式

---

**EventBus 是 OpenCode 架构的核心支柱，理解它对于掌握整个系统至关重要！**
