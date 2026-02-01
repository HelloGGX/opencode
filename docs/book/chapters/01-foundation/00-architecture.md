# 1.0 全景架构与设计哲学

在开始编写 OpenCode 的第一行代码之前，我们面临的第一个、也是最核心的抉择，并不是“选什么库”，而是 **“它应该长什么样”**。

我们要构建的是一个 AI 编码助手（AI Coding Assistant）。当我们环顾现有的主流产品，如 GitHub Copilot 或 Cursor，它们**在用户层面**大多以 IDE 插件或“魔改 IDE”的形式存在。这似乎是一个非常符合直觉的选择：开发者在哪里写代码，工具就应该在哪里。

但如果我们仅从**交互入口的形态**出发，盲目跟随这个直觉，可能会陷入一种架构上的“局部最优”。要理解这一点，我们需要从 Agent 的本质需求出发，重新推演这个形态选择的过程。

假如我们选择做一个 VS Code 扩展，现代 IDE 的确为插件提供了相当丰富的能力。插件可以访问整个 workspace，读取和修改文件，感知 Git 状态，甚至通过终端间接执行 shell 命令。像 GitHub Copilot 这样的产品，已经证明了这些能力在“代码补全”和“局部生成”场景下的巨大价值。

但关键问题不在于 **“能不能”**，而在于 **“适不适合”**。

IDE 插件的设计初衷，是作为**编辑器的附属能力**存在的。根据 VS Code 的扩展 API 规范，插件主要通过公开的 API 与编辑器的生命周期、命令面板、界面组件和工作区交互。这些能力高度集中在编辑器上下文之内，旨在扩展或增强编辑体验，而不是作为一个长期运行、可自治、可编排的行动主体。

从这个角度看，IDE 插件并不是一个合格的 Agent 宿主，它更像是一个 UI 层。即便在 GitHub Copilot 这样的产品中也是如此：插件本身并不承担复杂推理、多步规划或执行控制的职责，它的核心作用是采集上下文、触发请求并呈现结果，而真正的智能中枢始终运行在 IDE 之外的独立进程或服务中。

相反，如果我们将视角转向 **终端（CLI）模式**，Agent 所处的环境就发生了本质变化。终端并不是某个应用的扩展点，而是开发者的“操作系统层级”入口。它允许 Agent 以脚本化的方式操作一切：编辑多文件、运行测试、分析构建结果、监控日志，甚至在后台持续迭代和自我修正。

这种执行模型与 Agent 的“多步推理”（multi-step reasoning）高度契合。Agent 可以像一名人类工程师一样，将一个高层目标不断拆解为可执行的子任务，顺序执行，并根据中间反馈动态调整策略。这正是当前 IDE 插件体系难以自然承载的能力。

考虑到 OpenCode 是一个**开源且本地运行**的 AI 编码助手，我们还需要优先保证可移植性和低门槛。IDE 插件通常强依赖某一具体编辑器（如 VS Code 或 JetBrains），这不仅提高了用户的切换成本，也限制了开源贡献者的参与方式——并不是每个人都使用同一个 IDE。

而终端工具则天然具有跨平台优势。它可以作为一个独立的 CLI 程序运行，通过 pip、brew 等方式安装，适配任何操作系统和编辑环境。用户只需在 shell 中输入：

```
opencode --task "build a web app"
```

就可以启动一个完整的编码 Agent，而无需改变既有的开发习惯。

从安全角度看，本地 Agent 还需要一个相对清晰的“执行边界”，以应对潜在的风险操作（例如运行生成的脚本或构建命令）。终端形态更容易与容器化机制（如 Docker）集成，使 Agent 能够在隔离环境中运行和测试，而不污染主机系统。这一点在 IDE 插件模型中实现起来则更加复杂，往往需要额外的权限设计和安全假设。

此外，开源社区长期形成的开发工具范式，本身就高度偏好 CLI。Git、npm、Cargo 等工具的成功，很大程度上来自于它们的**组合性**：用户可以将这些命令自由地嵌入脚本、流水线或自动化系统中。一个以终端为核心形态的 AI 编码助手，也可以自然地融入这些工作流，例如作为 GitHub Actions 的一个步骤，或被 Neovim / Vim 作为外部命令调用。

当然，这并不意味着我们要完全放弃图形化界面。更合理的路径是将终端中的 Agent 视为核心“引擎”，并在其之上构建可选的GUI层。例如，Agent 在 CLI 中执行核心逻辑，但通过 WebSocket 将执行进度、决策过程和中间结果实时可视化到浏览器或其他前端界面中。

这是一种渐进式的架构选择：

> **先定义一个独立、自治、可编排的编码 Agent，再为它提供 IDE 作为可选界面。**

在这种架构下，IDE 不再是能力的边界，而只是 Agent 的众多“窗口”之一。这也更符合我们对 AI 编码助手本质的判断：它应该是一个独立的能力层，而不是某个特定编辑器的附属功能。

## 1.0.1 架构的演进：从一个 while 循环说起

确立了“以终端为核心”的形态后，摆在我们面前的下一个问题是：如何组织代码？

如果我们忽略所有的工程细节，我们需要思考一个基本问题：一个 AI 编码助手本质上是什么？它既不是简单的“输入-输出”函数，也不是单纯的聊天机器人。它本质上是一个具备副作用（Side Effect）的无限状态机。

一个功能往往可以抽象为一个函数：给它一个输入，它返回一个输出。对于 AI 助手来说，最简单的模型如下：

```javascript
const answer = await llm.ask("帮我写一个快速排序")
console.log(answer)
```
但当我们尝试用这种模式去处理真实世界的工程问题时，很快就会遇到挑战。

### 为什么需要“循环”？
假设用户提出了一个任务：“帮我修复项目中的类型报错”。这时，AI 仅靠一次“输入-输出”是无法完成任务的。因为它必须先查看具体的报错信息，读取代码文件，尝试修改，然后再次确认是否还有报错。

这意味着，智能体的行为不再是一次性的，而是一个持续的、有反馈的过程。为了实现这种持续性，我们必须引入一个最基础的结构：while 循环。

为了理解这一点，我们可以试着写一个最简陋的 Agent 。我们暂且叫它 SimpleAgent。假设我们要写一个程序，让它不断地检查环境并做出反应，最直观的代码可能是这样的：
```javascript
// ❌ 这是一个有缺陷的初步设计
function runAgent() {
  while (true) {
    const userInput = getUserInput();
    const result = llm.think(userInput);
    console.log(result);
  }
}
```
这段代码虽然能跑。但作为一个工程实现，它存在两个致命的痛点：
1. 缺乏终止条件：在计算机科学中，任何递归或循环都需要一个基准情形（Base Case）来退出。对于 LLM Agent 而言，无限循环意味着无限消耗 Token。
2. 状态丢失：runAgent 函数内部是无状态的。Agent 不知道自己上一轮做了什么，也不知道距离目标还有多远。它就像一条金鱼，每一轮循环都是全新的开始。

为了解决这两个问题，我们需要将 Agent 从一个简单的函数升级为一个对象（Object），利用对象来通过内存维持状态。

### 引入状态与约束
我们需要设计一种数据结构，它不仅能描述 Agent 的“当前位置”，还能描述它的“资源约束”。这就引出了 JavaScript 中类（Class）的概念，它可以将状态、行为与约束封装在一个稳定的对象边界内，使 Agent 不再是一次性计算，而是一个可持续演化的系统实体。基于这一点，我们声明一个 SimpleAgent 类。

让我们通过重构代码来看看，如何通过引入 energy（资源约束）和 goal（目标）来解决上述痛点。

```typescript
interface AgentState {
  // 资源约束：用于解决无限循环和成本控制问题
  energy: number;
  // 核心指令：Agent 存在的终极目的
  goal: string;
  // 可变状态：描述 Agent 当前在环境中的位置
  position: number;
  // 验收标准：当 position === targetPosition 时，任务结束
  targetPosition: number;
}

class SimpleAgent {
  // 使用 private 封装状态，避免外部随意修改导致状态机混乱
  private state: AgentState;

  constructor() {
    // 初始化状态，这相当于 Agent 的“出厂设置”
    this.state = {
      energy: 100,
      goal: "reach target",
      position: 0,
      targetPosition: 10
    };
  }
  
  // ... 后续逻辑
}
```
我们可以将以上代码与真实的 LLM 编码助手场景进行一一映射：
- energy (资源约束)：这不仅仅是一个数字。在真实场景中，它对应着 Context Window（上下文窗口） 的剩余空间，或者是用户的 API 预算。每次 Agent 执行操作（Act），都会消耗能量。当能量耗尽时，无论任务是否完成，Agent 都必须强制停止。这是为了防止程序陷入死循环而设计的“熔断机制”。
- goal (核心指令)：对应 System Prompt（系统提示词）。它定义了 Agent 的行为边界，例如“你是一个资深的 TypeScript 程序员”。
- position (当前上下文)：这是一个典型的可变状态（Mutable State）。在编码助手中，它代表当前代码库的状态（Git Diff）、报错信息或文件内容。随着 Agent 的运行，这个状态会不断发生变化（即副作用）。
- targetPosition (验收标准)：Agent 怎么知道自己做完了？在前面的简单循环中，Agent 是不知道停下来的。而在状态机模型中，当 position 与 targetPosition 重合（例如：单元测试全绿），即视为任务达成。

### 感知-决策-行动 循环
有了状态，Agent 依然是静止的。为了让它动起来，我们需要实现一个驱动循环。但在实现循环之前，必须先定义循环体内的逻辑。
在控制论中，智能体的行为通常遵循 P-D-A 范式：
1. 感知 (Perceive)：从环境中获取信息，更新内部认知。
2. 决策 (Decide)：基于感知到的信息和当前目标，选择下一个动作。
3. 行动 (Act)：执行动作，产生副作用，改变环境。

#### 感知 (Perceive)
Agent 无法直接处理原始的物理世界数据，它需要将环境信息抽象为它能理解的数据格式。

```typescript
// Agent perceives its environment
  private perceive(): { distanceToTarget: number; energyLevel: string } {
    const distance = Math.abs(this.state.targetPosition - this.state.position);
    // 将连续的数值离散化为状态标签，降低决策的复杂度
    const energyLevel = this.state.energy > 50 ? "high" : this.state.energy > 20 ? "medium" : "low";

    return { distanceToTarget: distance, energyLevel };
  }
```
> 注意这里的一个细节：我们在 perceive 中并没有直接返回 energy 的数值，而是将其转换为了 'high' | 'medium' | 'low' 这样的语义化标签。这种处理方式在 AI 领域非常常见——通过抽象降低决策模型的输入维度。对于 LLM 来说，"High" 比 "87" 更容易作为 prompt 的一部分进行推理。

#### 决策 (Decide)
有了感知数据，Agent 就需要做出判断。这部分逻辑构成了 Agent 的“大脑”。

```typescript
// Agent decides what to do
  private decide(perception: { distanceToTarget: number; energyLevel: string }): string {
    // 优先级 1：生存优先。如果能量过低，必须休息，否则任务会失败。
    if (perception.energyLevel === "low") {
      return "rest";
    }

    // 优先级 2：任务优先。如果还未到达目标，继续移动。
    if (perception.distanceToTarget > 0) {
      return "move";
    }

    // 优先级 3：任务完成。
    return "celebrate";
  }
```
这里的 decide 函数是一个纯函数（Pure Function），它完全依赖于输入产生输出，没有副作用。在真实的 AI Agent 中，这个函数内部通常就是一次 LLM API Call。我们把感知到的环境（Prompt）发给 LLM，它返回一个意图（Intent）。

#### 行动 (Act)
最后，我们需要一个函数来执行决策。这是整个系统中唯一产生副作用的地方。

```typescript
// Agent takes action
  private act(action: string): boolean {
    console.log(`🤖 Agent action: ${action}`);

    switch (action) {
      case "move":
        // 模拟向目标逼近的副作用
        if (this.state.position < this.state.targetPosition) {
          this.state.position++;
        } else if (this.state.position > this.state.targetPosition) {
          this.state.position--;
        }
        // 关键点：行动必然伴随着资源的消耗
        this.state.energy -= 10;
        console.log(`   Moved to position ${this.state.position} (Energy: ${this.state.energy})`);
        break;

      case "rest":
        this.state.energy += 30;
        console.log(`   Resting... (Energy: ${this.state.energy})`);
        break;

      case "celebrate":
        console.log(`   🎉 Goal achieved! Reached position ${this.state.position}`);
        return true; // 信号：任务完成
    }

    return false; // 信号：继续运行
  }
```

#### 组装运行时 (Runtime)
现在，我们将上述所有部件组装在一起，形成最终的 run 方法。你会发现，这其实就是我们最开始那个简陋 while 循环的完全体进化版。

```typescript
public run(): void {
    console.log("🚀 Agent starting...\n");

    let isRunning = true;
    let step = 0;
    
    // 这里的 20 是一个硬性的 Safety Limit，防止程序失控
    while (isRunning && step < 20) { 
      step++;
      console.log(`--- Step ${step} ---`);

      // 1. 感知
      const perception = this.perceive();
      // 2. 决策
      const decision = this.decide(perception);
      // 3. 行动（并获取反馈）
      const missionComplete = this.act(decision);

      // 检查终止条件：任务完成
      if (missionComplete) {
        isRunning = false;
      }

      // 检查终止条件：资源耗尽（熔断机制）
      if (this.state.energy <= 0) {
        console.log("💀 Agent ran out of energy!");
        isRunning = false;
      }

      console.log(); 
    }

    console.log("🏁 Agent stopped.");
  }
```

下面是完整的代码实现：

```typescript
interface AgentState {
  energy: number;
  goal: string;
  position: number;
  targetPosition: number;
}

class SimpleAgent {
  private state: AgentState;

  constructor() {
    this.state = {
      energy: 100,
      goal: "reach target",
      position: 0,
      targetPosition: 10
    };
  }

  // Agent perceives its environment
  private perceive(): { distanceToTarget: number; energyLevel: string } {
    const distance = Math.abs(this.state.targetPosition - this.state.position);
    const energyLevel = this.state.energy > 50 ? "high" : this.state.energy > 20 ? "medium" : "low";

    return { distanceToTarget: distance, energyLevel };
  }

  // Agent decides what to do
  private decide(perception: { distanceToTarget: number; energyLevel: string }): string {
    if (perception.energyLevel === "low") {
      return "rest";
    }

    if (perception.distanceToTarget > 0) {
      return "move";
    }

    return "celebrate";
  }

  // Agent takes action
  private act(action: string): boolean {
    console.log(`🤖 Agent action: ${action}`);

    switch (action) {
      case "move":
        // Move towards target
        if (this.state.position < this.state.targetPosition) {
          this.state.position++;
        } else if (this.state.position > this.state.targetPosition) {
          this.state.position--;
        }
        this.state.energy -= 10;
        console.log(`   Moved to position ${this.state.position} (Energy: ${this.state.energy})`);
        break;

      case "rest":
        this.state.energy += 30;
        console.log(`   Resting... (Energy: ${this.state.energy})`);
        break;

      case "celebrate":
        console.log(`   🎉 Goal achieved! Reached position ${this.state.position}`);
        return true; // Mission complete
    }

    return false; // Continue running
  }

  // Main agent loop
  public run(): void {
    console.log("🚀 Agent starting...\n");

    let isRunning = true;
    let step = 0;

    while (isRunning && step < 20) { // Safety limit
      step++;
      console.log(`--- Step ${step} ---`);

      // Agent cycle: Perceive -> Decide -> Act
      const perception = this.perceive();
      const decision = this.decide(perception);
      const missionComplete = this.act(decision);

      // Check if mission is complete
      if (missionComplete) {
        isRunning = false;
      }

      // Check if agent is out of energy
      if (this.state.energy <= 0) {
        console.log("💀 Agent ran out of energy!");
        isRunning = false;
      }

      console.log(); // Empty line for readability
    }

    console.log("🏁 Agent stopped.");
  }
}

// Run the agent
const agent = new SimpleAgent();
agent.run();
```

通过这段 SimpleAgent 的实现，我们构建了一个最小化的智能体模型。它不仅解决了最初“无限循环”和“状态丢失”的问题，还引入了资源约束和感知抽象的概念。

然而，细心的读者可能会发现，当前的 SimpleAgent 依然存在一个巨大的局限性：它的行为逻辑是硬编码的（写死在 switch-case 中）。

1. **交互的阻塞性**： 当 `callLLM` 正在进行网络请求时（这通常需要几秒甚至更久），整个程序是“假死”的。用户无法中断当前的执行，无法输入新的指令修正方向，甚至连实时的流式输出（Streaming）都很难优雅地插入到这个同步循环中，

2. **能力的耦合**： SimpleAgent使用了很多`console.log`, 如果某天我们需要为它开发一个 VS Code 插件或者 Web 界面，上这段逻辑就必须重写，因为 VS Code 不需要 console.log，而是需要 window.showInformationMessage。我们需要一套机制，让核心逻辑“看不见”用户界面，无论是 TUI、Web 还是 IDE，对核心逻辑来说都应该只是不同的“渲染端”。

3. **状态的易失性**： 所有的上下文（context）都保存在内存变量中。一旦用户关闭终端，或者程序因网络波动崩溃，所有的对话历史、AI 对项目结构的理解瞬间归零。

为了解决这些问题，我们需要需要对上述代码重新进行架构设计。

首先，为了解决耦合问题，我们将“大脑”与“肢体”分离。Core 层只负责思考和决策，不负责显示；UI 层只负责渲染，不负责逻辑。两者之间不能直接调用，必须通过事件（Event） 或 消息（Message） 进行通信。这样，Core 层就不再依赖于 console.log，而是发布一个 MessageUpdated 事件，无论是 TUI 还是 Web UI，监听到这个事件后自行决定如何渲染。

其次，为了解决阻塞问题，我们将同步的 while 循环改为异步的事件驱动模型。AI 的思考、工具的执行、文件的读写，都被抽象为系统中的异步任务。

最后，为了解决易失性，我们需要引入一个持久化的基础设施层（Infra），实时将内存中的状态同步到硬盘上。

这就自然演化出了 OpenCode 的分层架构。它不再是一个简单的脚本，而更像是一个运行在本地的微型操作系统。

从底层基础设施到顶层用户界面，OpenCode 共分为 5 个核心层次：

```mermaid
graph TB

%% ===== Global Compact Style =====
classDef ui fill:#F3F6FB,stroke:#4A6FA5,color:#1F2A44
classDef comm fill:#F7F9F5,stroke:#6B8E23,color:#2F3B1F
classDef core fill:#FFF6E5,stroke:#C47A00,stroke-width:1.5px,color:#4A2E00
classDef infra fill:#F5F5F5,stroke:#666666,color:#222222
classDef ext fill:#F0F7F7,stroke:#2E8B8B,color:#083A3A
classDef ai fill:#F9F0F5,stroke:#8B3A62,color:#3A1024

%% ================= UI =================
subgraph UI["用户界面层 (UI Layer)"]
    direction TB
    TUI[TUI Terminal<br/>OpenTUI + SolidJS]:::ui
    WebApp[Web App<br/>SolidJS + Vite]:::ui
    Desktop[Desktop<br/>Tauri]:::ui
    VSCode[VSCode Extension<br/>Extension API]:::ui
end

%% ================= Comm =================
subgraph Comm["通信层 (Communication)"]
    direction TB
    HTTP[HTTP Server<br/>Hono]:::comm
    RPC[RPC Worker]:::comm
    SSE[SSE Events<br/>实时推送]:::comm
    WS[WebSocket<br/>双向通信]:::comm
end

%% ================= Core =================
subgraph Core["核心业务层 (Core Layer)"]
    direction TB
    Session[Session<br/>会话管理]:::core
    Agent[Agent<br/>代理系统]:::core
    Permission[Permission<br/>权限控制]:::core
    Message[Message<br/>消息处理]:::core
    Question[Question<br/>交互问答]:::core
    Tool[Tool<br/>工具系统]:::core
    EventBus[EventBus<br/>事件总线<br/>30+ 事件类型]:::core
end

%% ================= Infra =================
subgraph Infra["基础设施层 (Infrastructure)"]
    direction TB
    Storage[Storage<br/>存储抽象]:::infra
    FileWatch[FileWatcher<br/>文件监控]:::infra
    Snapshot[Snapshot<br/>快照系统]:::infra
    Config[Config<br/>配置系统]:::infra
    Instance[Instance<br/>实例管理]:::infra
    Scheduler[Scheduler<br/>定时任务]:::infra
end

%% ================= Ext =================
subgraph Ext["扩展层 (Extension)"]
    direction TB
    LSP[LSP<br/>15+ 语言服务器]:::ext
    MCP[MCP<br/>模型上下文协议]:::ext
    Plugin[Plugin<br/>插件系统]:::ext
    Skill[Skill<br/>技能系统]:::ext
    Command[Command<br/>命令系统]:::ext
    Worktree[Worktree<br/>沙盒管理]:::ext
end

%% ================= AI =================
subgraph AI["AI 提供商层 (AI Provider)"]
    direction TB
    Provider[Provider 抽象层<br/>@ai-sdk]:::ai
    OpenAI[OpenAI]:::ai
    Anthropic[Anthropic]:::ai
    Google[Google]:::ai
    Others[... 20+ 提供商]:::ai
end

%% ================= Relations =================
UI --> Comm --> Core
Core --> Infra
Core --> Ext
Infra --> AI
Ext --> AI

```

### 架构层次说明

**1. 用户界面层 (UI Layer)**
- **职责**：作为系统的“皮肤”，负责呈现信息和接收指令。
- **演进思考**：为了支持“一次编写，多端运行”，我们将 TUI、Web、Desktop 视为平等的消费者。注意这里我们选用了 SolidJS 来构建 TUI，这是为了在终端中也能获得现代前端框架的组件化开发体验。

**2. 通信层 (Communication)**
- **职责**：连接前端和后端，处理数据传输。
- **演进思考**：由于 UI 层与 Core 层完全分离，我们需要一种高效的通信机制。对于 Web 端，我们使用 Hono 提供 HTTP 接口；对于 TUI 内部的 Worker 通信，我们实现了轻量级的 RPC。这一层确保了 Core 层永远不需要知道“谁在调用我”。

**3. 核心业务层 (Core Layer)**
- **职责**：系统的“大脑”，实现核心业务逻辑。
- **关键模块**：
    - **EventBus**：整个架构的血管。通过事件总线，Session 模块可以发布 `MessageCreated` 事件，而完全不需要知道 Permission 模块正在监听这个事件。
    - **Agent & Tool**：负责实际的推理和执行。

**4. 基础设施层 (Infrastructure)**
- **职责**：系统的“地基”，提供底层能力支撑。
- **演进思考**：为了解决状态易失性，Storage 模块将所有状态持久化为 JSON 文件；FileWatcher 则像神经末梢一样，感知文件系统的每一次微小变动，让 AI 能够“看见”代码的修改。

**5. 扩展层 (Extension)**
- **职责**：提供可扩展能力。
- **演进思考**：一个封闭的系统是没有生命力的。通过 LSP 和 MCP（Model Context Protocol）协议，我们将 OpenCode 变成了一个开放平台，能够接入外部的工具和知识库。

---

## 1.0.2 核心设计哲学
OpenCode 的架构设计遵循 5 大核心理念，这些理念贯穿整个系统设计。

### 1. 事件驱动架构 (Event-Driven Architecture)

在很多传统的 CLI 工具中，模块间的调用是直接的（Direct Call）。例如，当 AI 生成代码后，直接调用文件写入函数。这种紧耦合导致了扩展极其困难。

**设计理念**：模块间通过事件解耦，发布者和订阅者互不依赖。
**代码示例**：

```typescript
// 发布者：Session 模块创建会话后发布事件
// src/session/index.ts
await Storage.write(["session", projectID, sessionID], session)

// 发布事件，不关心谁在监听
Bus.publish(Session.Event.Created, {
  sessionID,
  info: session
})

// 订阅者1：Share 模块监听并同步到云端
// src/share/share.ts
Bus.subscribe(Session.Event.Created, async (evt) => {
  await syncToCloud(evt.properties.info)
})

// 订阅者2：Analytics 模块监听并记录统计
// src/analytics/index.ts
Bus.subscribe(Session.Event.Created, async (evt) => {
  await trackEvent("session_created", {
    sessionID: evt.properties.sessionID
  })
})
```

**优势**：
- ✅ **松耦合**：Session 模块不需要知道 Share 和 Analytics 的存在
- ✅ **易扩展**：新增订阅者无需修改发布者代码
- ✅ **可观测**：所有事件可统一监控和日志记录
- ✅ **异步处理**：事件处理不阻塞主流程

**挑战**：
- ⚠️ **调试复杂**：事件链路追踪需要专门工具
- ⚠️ **顺序保证**：需要额外机制保证事件处理顺序
- ⚠️ **内存泄漏**：订阅者未正确清理会导致内存泄漏

**最佳实践**：
```typescript
// 使用 using 语法自动清理订阅
using unsub = Bus.subscribe(Event.Updated, handler)
// 作用域结束时自动取消订阅
```


### 2. 实例隔离 (Instance Isolation)

在开发多项目并行的工具时，全局变量（Global State）是万恶之源。如果用户同时打开两个终端分别操作两个项目，全局变量会导致状态冲突。
**设计理念**：每个项目目录对应一个独立实例，状态完全隔离，资源自动清理。
**代码示例**：

```typescript
// src/project/instance.ts
export function state<T>(
  init: () => T | Promise<T>,
  dispose?: (state: T) => void | Promise<void>
): () => T {
  // 使用 Symbol 作为 key，确保状态在不同实例间隔离
  const key = Symbol()
  
  return () => {
    // 获取当前实例（每个项目目录对应一个实例）
    const instance = getCurrentInstance()
    
    // 懒加载：首次访问时才初始化
    if (!instance.state.has(key)) {
      const value = init()
      instance.state.set(key, value)
      
      // 注册清理函数，实例销毁时自动调用
      if (dispose) {
        instance.disposers.push(() => dispose(value))
      }
    }
    
    return instance.state.get(key)
  }
}

// 使用示例：创建实例级状态
const sessions = Instance.state(
  () => new Map<string, Session>(),
  (map) => {
    // 清理所有会话
    for (const session of map.values()) {
      session.cleanup()
    }
  }
)

// 不同项目的状态互不影响
// /project-a → Instance A → sessions Map A
// /project-b → Instance B → sessions Map B
```

**优势**：
- ✅ **多项目支持**：可同时处理多个项目，状态不污染
- ✅ **资源隔离**：每个实例有独立的配置、会话、工具
- ✅ **自动清理**：实例销毁时自动释放资源（数据库连接、文件监控等）
- ✅ **内存安全**：避免内存泄漏和状态混乱

**挑战**：
- ⚠️ **内存开销**：多实例会增加内存占用
- ⚠️ **跨实例通信**：需要 GlobalBus 支持
- ⚠️ **状态同步**：跨实例状态同步需要额外机制

**实际应用场景**：
```typescript
// 场景1：用户同时打开两个项目
// Terminal 1: cd /project-a && opencode tui
// Terminal 2: cd /project-b && opencode tui
// 两个实例完全独立，互不干扰

// 场景2：Worktree 沙盒隔离
// 主项目：/main-project
// 沙盒1：/main-project/.opencode/worktree/brave-falcon
// 沙盒2：/main-project/.opencode/worktree/cosmic-nebula
// 每个沙盒都是独立实例
```

### 3. 类型安全优先 (Type-Safe First)

**设计理念**：使用 Zod 实现运行时验证 + TypeScript 编译时检查，双重保障类型安全。

**代码示例**：

```typescript
// src/config/config.ts
import { z } from "zod"

// 1. 定义 Zod Schema
export const Config = z.object({
  model: z.string().optional(),
  agent: z.record(z.string(), AgentConfig).optional(),
  permission: z.record(z.string(), z.any()).optional(),
  mcp: z.record(z.string(), McpConfig).optional()
})

// 2. 自动推导 TypeScript 类型
export type Config = z.infer<typeof Config>
// 等价于：
// type Config = {
//   model?: string
//   agent?: Record<string, AgentConfig>
//   permission?: Record<string, any>
//   mcp?: Record<string, McpConfig>
// }

// 3. 运行时验证
const data = await Bun.file("opencode.md").json()
const config = Config.parse(data)
// 如果 data 不符合 schema，会抛出详细错误：
// ZodError: [
//   {
//     "code": "invalid_type",
//     "expected": "string",
//     "received": "number",
//     "path": ["model"]
//   }
// ]

// 4. 安全的类型推断
config.model // TypeScript 知道这是 string | undefined
config.agent?.build // 自动补全，类型安全
```

**优势**：
- ✅ **编译时检查**：TypeScript 在编写代码时就能发现类型错误
- ✅ **运行时验证**：Zod 在运行时验证数据，防止非法输入
- ✅ **自动生成类型**：无需手动维护类型定义
- ✅ **详细错误信息**：Zod 提供精确的错误路径和原因

**挑战**：
- ⚠️ **学习成本**：需要学习 Zod API
- ⚠️ **性能开销**：运行时验证有轻微性能损耗
- ⚠️ **Schema 维护**：复杂类型的 Schema 可能冗长

**最佳实践**：
```typescript
// 使用 .safeParse() 处理可能失败的验证
const result = Config.safeParse(data)
if (!result.success) {
  console.error("配置验证失败：", result.error.format())
  // 使用默认配置
  return defaultConfig
}
return result.data

// 使用 .transform() 转换数据
const PortSchema = z.string().transform((val) => parseInt(val, 10))
const config = z.object({
  port: PortSchema // "3000" → 3000
})
```


### 4. 渐进式增强 (Progressive Enhancement)

**设计理念**：从 MVP 到完整系统，每个迭代都可独立运行，降低开发风险。

**迭代路线图**：

```mermaid
graph LR
    I1[Iteration 1<br/>CLI 骨架<br/>28 个命令] --> I2[Iteration 2<br/>配置系统<br/>EventBus]
    I2 --> I3[Iteration 3<br/>AI 集成<br/>20+ 提供商]
    I3 --> I4[Iteration 4<br/>文件系统<br/>工具系统]
    I4 --> I5[Iteration 5<br/>会话处理<br/>权限控制]
    I5 --> I6[Iteration 6<br/>TUI 界面<br/>实时交互]
    I6 --> I7[Iteration 7<br/>LSP 集成<br/>代码智能]
    I7 --> I8[Iteration 8<br/>MCP 协议<br/>插件生态]
    I8 --> I9[Iteration 9<br/>多端部署<br/>Web/Desktop]
    
    style I1 fill:#e1f5e1
    style I2 fill:#e1f5e1
    style I3 fill:#e1f5e1
    style I4 fill:#fff3cd
    style I5 fill:#fff3cd
    style I6 fill:#fff3cd
    style I7 fill:#f8d7da
    style I8 fill:#f8d7da
    style I9 fill:#f8d7da
```

**每个迭代的可运行状态**：

```bash
# Iteration 1: 可运行基础命令
$ opencode --version
opencode version 1.1.39

$ opencode --help
Commands:
  session    Manage sessions
  project    Manage projects
  ...

# Iteration 2: 可加载配置
$ opencode config get model
anthropic/claude-3-5-sonnet-20241022

# Iteration 3: 可调用 AI
$ opencode session new --agent build
Session created: sess_abc123

$ opencode session prompt sess_abc123 "Hello"
AI: Hello! How can I help you today?

# Iteration 4: 可执行文件操作
$ opencode tool exec read --path README.md
[Tool Output] File content...

# Iteration 5: 可处理完整会话
$ opencode tui
[TUI 界面启动，支持实时交互]
```

**优势**：
- ✅ **快速验证**：每个迭代都能看到实际效果
- ✅ **降低风险**：问题早发现，避免大规模返工
- ✅ **易于理解**：学习曲线平缓，逐步深入
- ✅ **灵活调整**：可根据反馈调整后续迭代

**挑战**：
- ⚠️ **架构规划**：需要提前设计好模块边界
- ⚠️ **重构成本**：架构不合理会导致后期重构
- ⚠️ **功能取舍**：需要明确每个迭代的核心功能

### 5. 约定优于配置 (Convention over Configuration)

**设计理念**：提供合理的默认值，减少配置复杂度，同时保持灵活性。

**代码示例**：

```typescript
// 默认配置（内部实现）
const defaults = {
  model: "anthropic/claude-3-5-sonnet-20241022",
  agent: "build",
  permission: {
    edit: { "*": "ask" },    // 编辑文件需要询问
    bash: { "*": "ask" },    // 执行命令需要询问
    read: { "*": "allow" }   // 读取文件自动允许
  }
}

// 用户只需配置差异
// .opencode/opencode.jsonc
{
  // JSONC 支持注释
  "model": "openai/gpt-4",
  "permission": {
    "edit": {
      "*.ts": "allow"  // TypeScript 文件自动允许编辑
    }
  }
}

// 最终配置 = 远程配置 + 全局配置 + 项目配置（按优先级合并）
const finalConfig = mergeConfigConcatArrays(remoteConfig, globalConfig, projectConfig)
```

**约定示例**：

| 约定 | 说明 | 覆盖方式 |
|------|------|---------|
| 配置文件路径 | `opencode.jsonc` 或 `opencode.json` | 环境变量 `OPENCODE_CONFIG` |
| 配置目录 | `.opencode/opencode.jsonc` | 环境变量 `OPENCODE_CONFIG_DIR` |
| 存储目录 | `~/.opencode/storage/` | 配置项 `storage.path` |
| 默认 Agent | `build` | 配置项 `agent` |
| 默认模型 | Claude 3.5 Sonnet | 配置项 `model` |
| Agent 定义 | `.opencode/agent/**/*.md` | 配置项 `agent` |
| Command 定义 | `.opencode/command/**/*.md` | 配置项 `command` |
| Plugin 定义 | `.opencode/plugin/*.{ts,js}` | 配置项 `plugin` |

**优势**：
- ✅ **开箱即用**：无需配置即可运行
- ✅ **降低门槛**：新手友好，减少学习成本
- ✅ **保持灵活**：高级用户可自定义所有配置
- ✅ **最佳实践**：默认值体现最佳实践

**挑战**：
- ⚠️ **默认值设计**：需要精心设计，平衡通用性和特殊需求
- ⚠️ **文档维护**：需要清晰文档说明约定和覆盖方式
- ⚠️ **版本兼容**：默认值变更需要考虑向后兼容

**最佳实践**：
```bash
# 使用环境变量覆盖约定
export OPENCODE_MODEL="openai/gpt-4"
export OPENCODE_CONFIG="/path/to/custom/config.jsonc"

# 使用配置文件覆盖约定
# opencode.jsonc
{
  "model": "openai/gpt-4",
  "storage": {
    "path": "/custom/path"
  }
}

# 使用命令行参数覆盖约定
opencode session new --model "openai/gpt-4"
```

---

## 1.0.3 技术栈选型理由

OpenCode 的技术栈选型经过深思熟虑，每个技术都有明确的理由和权衡。

| 技术 | 选型理由 | 优势 | Trade-off |
|------|---------|------|-----------|
| **Bun** | 3-5x 快于 npm，原生 TypeScript | • 快速安装和执行<br/>• 内置测试框架<br/>• 原生 TypeScript | • 生态成熟度 < Node.js<br/>• 部分包不兼容 |
| **Turbo** | 增量构建，10x 提升大型项目 | • 智能缓存<br/>• 任务依赖管理<br/>• 并行执行 | • 配置复杂度增加<br/>• 学习曲线 |
| **Zod** | 类型安全，运行时验证 | • 编译时 + 运行时双重保障<br/>• 自动生成类型<br/>• 详细错误信息 | • 学习成本<br/>• 轻微性能开销 |
| **@ai-sdk** | 统一接口，20+ 提供商 | • 提供商无关<br/>• 流式响应<br/>• 工具调用支持 | • 抽象层性能损耗<br/>• 部分提供商特性受限 |
| **OpenTUI** | 终端原生组件，高性能 | • 响应式更新<br/>• 组件化开发<br/>• 跨平台支持 | • 学习曲线陡峭<br/>• 生态较小 |
| **SolidJS** | 细粒度响应式，小体积 | • 性能优异<br/>• 无虚拟 DOM<br/>• 体积小 | • 生态 < React<br/>• 社区较小 |
| **Playwright** | 跨浏览器 E2E 测试 | • 多浏览器支持<br/>• 自动等待<br/>• 录制功能 | • 测试速度较慢<br/>• 资源占用高 |
| **Hono** | 轻量级 Web 框架 | • 极快的路由<br/>• 边缘运行时支持<br/>• TypeScript 优先 | • 生态较新<br/>• 中间件较少 |

### 为什么选择 Bun？

```typescript
// 性能对比（安装 50 个包）
npm install    // ~45 秒
yarn install   // ~30 秒
pnpm install   // ~20 秒
bun install    // ~8 秒  ← 3-5x 提升

// 原生 TypeScript 支持
// 无需编译，直接运行
bun run src/index.ts

// 内置测试框架
import { test, expect } from "bun:test"

test("example", () => {
  expect(1 + 1).toBe(2)
})
```

### 为什么选择 Turbo？

```json
// turbo.json - 智能缓存和依赖管理
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],  // 先构建依赖包
      "outputs": ["dist/**"],   // 缓存输出目录
      "cache": true             // 启用缓存
    }
  }
}

// 性能对比（构建 10 个包）
// 首次构建
npm run build    // ~120 秒
turbo build      // ~120 秒

// 二次构建（无变更）
npm run build    // ~120 秒（重新构建）
turbo build      // ~2 秒（使用缓存）← 60x 提升
```

### 为什么选择 @ai-sdk？

```typescript
// 统一接口，切换提供商只需改配置
import { streamText } from "ai"
import { anthropic } from "@ai-sdk/anthropic"
import { openai } from "@ai-sdk/openai"

// 使用 Anthropic
const result = await streamText({
  model: anthropic("claude-3-5-sonnet-20241022"),
  messages: [{ role: "user", content: "Hello" }]
})

// 切换到 OpenAI，代码无需修改
const result = await streamText({
  model: openai("gpt-4"),
  messages: [{ role: "user", content: "Hello" }]
})

// 支持 20+ 提供商
// OpenAI, Anthropic, Google, Mistral, Cohere, Groq, 
// Together AI, Fireworks, Perplexity, DeepSeek...
```


---

## 1.0.4 数据流向图

理解数据如何在系统中流动，是掌握架构的关键。

```mermaid
sequenceDiagram
    participant User as 用户
    participant UI as 用户界面<br/>(TUI/Web/VSCode)
    participant Comm as 通信层<br/>(HTTP/RPC)
    participant Session as Session<br/>会话管理
    participant Permission as Permission<br/>权限控制
    participant LLM as LLM<br/>AI 提供商
    participant Tool as Tool<br/>工具系统
    participant Bus as EventBus<br/>事件总线
    participant Storage as Storage<br/>存储系统
    
    User->>UI: 输入提示词<br/>"创建 README.md"
    UI->>Comm: HTTP POST /session/prompt
    Comm->>Session: session.prompt()
    
    Session->>Permission: 检查权限
    Permission-->>Session: 权限通过
    
    Session->>LLM: streamText()
    LLM-->>Session: 流式响应<br/>"我将创建 README.md"
    Session->>Bus: publish(Message.Updated)
    Bus->>UI: SSE 推送更新
    UI-->>User: 实时显示响应
    
    LLM-->>Session: 工具调用<br/>write(path: "README.md")
    Session->>Permission: 检查 write 权限
    Permission->>UI: 询问用户
    UI-->>User: 显示权限请求
    User->>UI: 批准
    UI->>Permission: 用户批准
    Permission-->>Session: 权限通过
    
    Session->>Tool: tool.execute()
    Tool->>Storage: 写入文件
    Storage-->>Tool: 写入成功
    Tool->>Bus: publish(File.Edited)
    Bus->>UI: SSE 推送文件变更
    Tool-->>Session: 工具执行结果
    
    Session->>LLM: 继续对话
    LLM-->>Session: "README.md 已创建"
    Session->>Storage: 保存会话状态
    Session->>Bus: publish(Session.Updated)
    Bus->>UI: SSE 推送完成
    UI-->>User: 显示完成状态
```

### 关键流程说明

**1. 用户输入 → 会话处理**
```typescript
// 用户在 TUI 中输入提示词
const input = "创建 README.md 文件"

// TUI 发送 HTTP 请求
await fetch("/session/prompt", {
  method: "POST",
  body: JSON.stringify({
    sessionID: "sess_abc123",
    content: input
  })
})

// Session 处理请求
await Session.prompt({
  sessionID: "sess_abc123",
  content: input
})
```

**2. 权限检查**
```typescript
// 在执行敏感操作前检查权限
await Permission.ask({
  permission: "write",
  patterns: ["README.md"],
  sessionID: "sess_abc123"
})

// 如果配置为 "ask"，会弹出确认对话框
// 如果配置为 "allow"，自动通过
// 如果配置为 "deny"，抛出异常
```

**3. AI 流式响应**
```typescript
// 调用 AI 提供商
const stream = await streamText({
  model: anthropic("claude-3-5-sonnet-20241022"),
  messages: [...history, { role: "user", content: input }],
  tools: { write: WriteTool }
})

// 处理流式事件
for await (const event of stream.fullStream) {
  switch (event.type) {
    case "text-delta":
      // 实时显示文本
      Bus.publish(Message.Event.PartUpdated, {
        text: event.textDelta
      })
      break
    
    case "tool-call":
      // 执行工具调用
      await Tool.execute(event.toolName, event.args)
      break
  }
}
```

**4. 事件通知**
```typescript
// 工具执行后发布事件
Bus.publish(File.Event.Edited, {
  file: "README.md"
})

// 多个订阅者响应
Bus.subscribe(File.Event.Edited, async (evt) => {
  // 订阅者1：自动格式化
  await formatFile(evt.properties.file)
})

Bus.subscribe(File.Event.Edited, async (evt) => {
  // 订阅者2：更新文件树
  await refreshFileTree()
})
```

**5. 状态持久化**
```typescript
// 保存会话状态
await Storage.write(
  ["session", projectID, sessionID],
  session
)

// 保存消息内容
await Storage.write(
  ["message", sessionID, messageID],
  message
)
```

---

## 1.0.5 前置知识清单

在开始学习本书之前，请评估你的知识储备。

### 必备知识（⭐⭐⭐）

**1. TypeScript 基础**

你需要理解以下概念：

```typescript
// 类型注解
const name: string = "OpenCode"
const age: number = 1

// 接口
interface User {
  id: string
  name: string
  email?: string  // 可选属性
}

// 泛型
function identity<T>(value: T): T {
  return value
}

// 类型推断
const users = [{ id: "1", name: "Alice" }]
// TypeScript 自动推断 users 类型为 { id: string; name: string }[]

// 联合类型
type Status = "idle" | "busy" | "error"

// 交叉类型
type UserWithRole = User & { role: string }

// async/await
async function fetchData(): Promise<string> {
  const response = await fetch("/api/data")
  return response.text()
}

// 命名空间
namespace Utils {
  export function log(msg: string) {
    console.log(msg)
  }
}
```

**自测题**：
```typescript
// 你能理解这段代码吗？
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

// 问题：
// 1. 泛型 <T> 的作用是什么？
// 2. Symbol() 为什么能确保状态隔离？
// 3. 闭包在这里如何应用？
// 4. dispose 函数的作用是什么？
```

**推荐学习资源**：
- [TypeScript 官方文档](https://www.typescriptlang.org/docs/)
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/)

**2. Git 基础**

你需要掌握：

```bash
# 基本操作
git clone <url>           # 克隆仓库
git status                # 查看状态
git add .                 # 添加文件
git commit -m "message"   # 提交
git push                  # 推送
git pull                  # 拉取

# 分支管理
git branch feature        # 创建分支
git checkout feature      # 切换分支
git merge feature         # 合并分支

# 高级概念（OpenCode 使用）
git worktree add <path>   # 创建工作树
git worktree list         # 列出工作树
```

**自测题**：
- 什么是工作区、暂存区、仓库？
- `git reset` 和 `git revert` 的区别？
- 什么是 Git worktree？

**3. 终端操作**

你需要熟悉：

```bash
# 文件操作
cd /path/to/dir          # 切换目录
ls -la                   # 列出文件
mkdir -p dir/subdir      # 创建目录
rm -rf dir               # 删除目录
cat file.txt             # 查看文件
echo "text" > file.txt   # 写入文件

# 环境变量
export VAR=value         # 设置环境变量
echo $VAR                # 读取环境变量

# 包管理器
npm install              # 安装依赖
npm run build            # 运行脚本
```

### 推荐知识（⭐⭐）

**1. Node.js 生态**

```json
// package.json 配置
{
  "name": "@opencode/core",
  "version": "1.0.0",
  "type": "module",           // ESM 模块
  "exports": {
    ".": "./src/index.ts"     // 导出路径
  },
  "dependencies": {
    "zod": "^3.22.0"
  },
  "scripts": {
    "build": "tsc",
    "test": "bun test"
  }
}
```

**2. 设计模式**

OpenCode 使用的设计模式：

```typescript
// 观察者模式（EventBus）
Bus.subscribe(Event.Created, handler)

// 工厂模式（Provider）
Provider.create({ providerID: "openai", modelID: "gpt-4" })

// 单例模式（Instance）
const instance = Instance.current()

// 策略模式（Permission）
Permission.evaluate(rule, context)
```

**3. 函数式编程**

```typescript
// 高阶函数
const map = <T, U>(arr: T[], fn: (item: T) => U): U[] => {
  return arr.map(fn)
}

// 闭包
function counter() {
  let count = 0
  return () => ++count
}

// 纯函数
function add(a: number, b: number): number {
  return a + b  // 无副作用
}
```

### 加分项（⭐）

- 🔹 Monorepo 经验（Turborepo/Nx/Lerna）
- 🔹 LSP 协议了解（Language Server Protocol）
- 🔹 终端 UI 开发（Ink/Blessed/Charm）
- 🔹 AI/LLM 基础知识（Prompt Engineering）
- 🔹 Rust 基础（部分工具用 Rust 实现）

### 自测题答案

**TypeScript 自测题答案**：
1. **泛型 <T>**：允许函数适用于任意类型，保持类型安全
2. **Symbol()**：每次调用返回唯一值，确保不同实例的 key 不冲突
3. **闭包**：返回的函数捕获了 `key` 变量，形成闭包
4. **dispose**：清理函数，在实例销毁时释放资源

**Git 自测题答案**：
- **工作区/暂存区/仓库**：工作区是当前目录，暂存区是 `git add` 后的状态，仓库是 `git commit` 后的历史
- **reset vs revert**：reset 修改历史，revert 创建新提交撤销
- **worktree**：允许同一仓库有多个工作目录，用于并行开发


---

## 1.0.6 学习路线图

本书共 15 章，建议按以下路线学习，每个阶段都有明确的里程碑验证。

### 学习阶段规划

```mermaid
gantt
    title OpenCode 学习路线图（12 周）
    dateFormat  YYYY-MM-DD
    section 基础篇
    第1章 环境搭建           :a1, 2025-02-01, 3d
    第2章 配置系统           :a2, after a1, 4d
    第3章 AI集成             :a3, after a2, 4d
    第4章 文件系统           :a4, after a3, 3d
    第5章 工具系统           :a5, after a4, 3d
    
    section 核心篇
    第6章 会话处理           :b1, after a5, 5d
    第7章 权限控制           :b2, after b1, 3d
    第8章 TUI界面            :b3, after b2, 5d
    第9章 工具扩展           :b4, after b3, 4d
    第10章 LSP集成           :b5, after b4, 5d
    
    section 工程篇
    第11章 MCP协议           :c1, after b5, 4d
    第12章 测试体系           :c2, after c1, 4d
    第13章 国际化             :c3, after c2, 3d
    第14章 CI/CD              :c4, after c3, 3d
    
    section 优化篇
    第15章 性能优化           :d1, after c4, 5d
```

### 详细学习计划

**Week 1-2: 基础篇（第 1-5 章）**

| 天数 | 章节 | 学习内容 | 验证目标 |
|------|------|---------|---------|
| Day 1-3 | 第1章 | Monorepo + Bun + CLI | ✅ 能运行 `opencode --version` |
| Day 4-7 | 第2章 | 配置系统 + EventBus | ✅ 能加载配置并发布事件 |
| Day 8-11 | 第3章 | AI 提供商集成 | ✅ 能调用 LLM 并获得响应 |
| Day 12-14 | 第4-5章 | 文件系统 + 工具 | ✅ 能执行文件操作工具 |

**Week 3-4: 核心篇（第 6-10 章）**

| 天数 | 章节 | 学习内容 | 验证目标 |
|------|------|---------|---------|
| Day 15-19 | 第6章 | 会话处理循环 | ✅ 能创建会话并处理多轮对话 |
| Day 20-22 | 第7章 | 权限控制系统 | ✅ 能配置权限规则并验证 |
| Day 23-27 | 第8章 | TUI 界面开发 | ✅ 能启动 TUI 并交互 |
| Day 28-31 | 第9-10章 | 工具扩展 + LSP | ✅ 能使用 LSP 代码智能 |

**Week 5-6: 工程篇（第 11-14 章）**

| 天数 | 章节 | 学习内容 | 验证目标 |
|------|------|---------|---------|
| Day 32-35 | 第11章 | MCP 协议集成 | ✅ 能连接 MCP 服务器 |
| Day 36-39 | 第12章 | 测试体系构建 | ✅ 能运行完整测试套件 |
| Day 40-42 | 第13章 | 国际化实现 | ✅ 能切换语言界面 |
| Day 43-45 | 第14章 | CI/CD 自动化 | ✅ 能自动构建和部署 |

**Week 7-12: 优化篇 + 实践项目**

| 周数 | 内容 | 目标 |
|------|------|------|
| Week 7 | 第15章 性能优化 | ✅ 理解性能优化策略 |
| Week 8-9 | 实践项目1：自定义 Agent | ✅ 开发完整的自定义 Agent |
| Week 10-11 | 实践项目2：MCP 服务器 | ✅ 开发自己的 MCP 服务器 |
| Week 12 | 实践项目3：插件开发 | ✅ 开发并发布插件 |

### 里程碑验证

每个阶段结束后，请完成以下验证：

**里程碑 1（Week 2）：基础环境搭建完成**

```bash
# 验证清单
✅ Bun 安装成功
✅ 项目依赖安装完成
✅ 能运行 opencode --version
✅ 能加载配置文件
✅ 能调用 AI 并获得响应
✅ 能执行基础工具（read/write）

# 验证命令
bun --version
opencode --version
opencode config get model
opencode session new --agent build
opencode tool exec read --path README.md
```

**里程碑 2（Week 4）：核心功能实现**

```bash
# 验证清单
✅ 能创建和管理会话
✅ 能处理多轮对话
✅ 权限系统正常工作
✅ TUI 界面可用
✅ LSP 代码智能可用

# 验证命令
opencode tui
# 在 TUI 中：
# 1. 创建新会话
# 2. 发送消息并获得响应
# 3. 执行文件操作（需要权限确认）
# 4. 使用代码跳转功能
```

**里程碑 3（Week 6）：工程化完成**

```bash
# 验证清单
✅ MCP 服务器连接成功
✅ 测试套件全部通过
✅ 多语言界面切换正常
✅ CI/CD 流程配置完成

# 验证命令
bun test
bun run test:e2e
opencode mcp list
# 切换语言并验证界面
```

**里程碑 4（Week 12）：完整项目交付**

```bash
# 验证清单
✅ 性能优化完成（缓存、节流等）
✅ 自定义 Agent 开发完成
✅ MCP 服务器开发完成
✅ 插件开发并发布

# 验证命令
# 运行自己开发的功能
opencode session new --agent my-custom-agent
# 连接自己的 MCP 服务器
# 使用自己开发的插件
```

### 学习建议

**1. 循序渐进，不要跳章**
- 每章都是后续章节的基础
- 跳章会导致理解困难

**2. 动手实践，运行所有代码**
- 不要只看代码，要实际运行
- 修改代码参数，观察效果变化

**3. 完成所有练习**
- 练习题设计用于巩固知识
- 参考答案仅供验证，先独立完成

**4. 记录学习笔记**
- 记录遇到的问题和解决方案
- 总结每章的核心概念

**5. 参与社区讨论**
- 加入 Discord 社区
- 分享学习心得和问题

---

## 1.0.7 从零开始：完整项目初始化

现在让我们开始实际操作，从零搭建 OpenCode 开发环境。

### 步骤 1：克隆仓库

```bash
# 克隆 OpenCode 仓库
git clone https://github.com/anomalyco/opencode.git
cd opencode

# 查看项目结构
ls -la

# 输出：
# drwxr-xr-x  .git/
# drwxr-xr-x  packages/
# -rw-r--r--  package.json
# -rw-r--r--  turbo.json
# -rw-r--r--  bunfig.toml
# -rw-r--r--  README.md
# ...

# 查看分支（默认分支是 dev）
git branch -a
# * dev
#   remotes/origin/dev
#   remotes/origin/main
```

**注意事项**：
- OpenCode 的默认分支是 `dev`，不是 `main`
- 如果需要稳定版本，切换到 `main` 分支

### 步骤 2：安装 Bun

Bun 是 OpenCode 的核心依赖，必须先安装。

```bash
# macOS/Linux 安装
curl -fsSL https://bun.sh/install | bash

# 验证安装
bun --version
# 输出：1.3.5

# 查看 Bun 路径
which bun
# 输出：/Users/username/.bun/bin/bun

# 如果命令未找到，添加到 PATH
echo 'export PATH="$HOME/.bun/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**Windows 用户**：
```bash
# 使用 WSL (Windows Subsystem for Linux)
wsl --install
# 然后在 WSL 中安装 Bun
curl -fsSL https://bun.sh/install | bash
```

**验证 Bun 功能**：
```bash
# 测试 Bun 运行 TypeScript
echo 'console.log("Hello from Bun!")' > test.ts
bun run test.ts
# 输出：Hello from Bun!

# 测试 Bun 包管理
bun --help
# 显示帮助信息
```

### 步骤 3：安装依赖

```bash
# 安装所有 workspace 依赖
bun install

# 安装过程输出：
# bun install v1.3.5
# 🔍 Resolving dependencies...
# 📦 Downloading packages...
# ⚙️  Installing packages...
# + 1234 packages installed [12.34s]

# 验证安装
ls node_modules/
# 输出：大量依赖包

# 检查 workspace 包
ls packages/
# 输出：
# app/
# console/
# desktop/
# docs/
# enterprise/
# function/
# opencode/
# plugin/
# sdk/
# ui/
# util/
# web/
```

**常见问题**：

```bash
# 问题1：安装失败 - 网络问题
# 解决：使用国内镜像
bun config set registry https://registry.npmmirror.com

# 问题2：权限错误
# 解决：不要使用 sudo
# Bun 应该安装在用户目录，不需要 root 权限

# 问题3：依赖冲突
# 解决：清理缓存重试
rm -rf node_modules bun.lock
bun install
```

### 步骤 4：构建项目

```bash
# 使用 Turbo 构建所有包
bun run build

# 构建过程输出：
# • packages/util:build: cache miss, executing...
# • packages/ui:build: cache miss, executing...
# • packages/opencode:build: cache miss, executing...
# ...
# • packages/opencode:build: finished in 5.2s
# 
# Tasks:    12 successful, 12 total
# Cached:   0 cached, 12 total
# Time:     15.3s

# 二次构建（验证缓存）
bun run build

# 输出：
# • packages/util:build: cache hit, replaying logs...
# • packages/ui:build: cache hit, replaying logs...
# ...
# Tasks:    12 successful, 12 total
# Cached:   12 cached, 12 total
# Time:     1.2s  ← 快了 10x+
```

**构建产物检查**：
```bash
# 查看构建产物
ls packages/opencode/dist/
# 输出：
# index.js
# index.d.ts
# ...

# 查看其他包的构建产物
ls packages/ui/dist/
ls packages/util/dist/
```

### 步骤 5：验证安装

```bash
# 方式1：直接运行源码
bun run packages/opencode/src/index.ts --version
# 输出：opencode version 1.1.39

# 方式2：运行构建产物
node packages/opencode/dist/index.js --version
# 输出：opencode version 1.1.39

# 查看所有可用命令
bun run packages/opencode/src/index.ts --help

# 输出：
# opencode <command>
# 
# Commands:
#   acp              Start ACP server
#   mcp              Manage MCP servers
#   tui              Launch TUI interface
#   session          Manage sessions
#   project          Manage projects
#   tool             Execute tools
#   config           Manage configuration
#   ... (28 个命令)
# 
# Options:
#   --version        Show version number
#   --help           Show help
```

**创建全局命令（可选）**：
```bash
# 创建软链接
sudo ln -s $(pwd)/packages/opencode/src/index.ts /usr/local/bin/opencode

# 或者使用 bun link
cd packages/opencode
bun link
cd ../..

# 验证全局命令
opencode --version
# 输出：opencode version 1.1.39
```

### 步骤 6：配置开发环境

```bash
# 创建配置目录
mkdir -p .opencode

# 创建配置文件
cat > .opencode/opencode.md << 'EOF'
---
model: "anthropic/claude-3-5-sonnet-20241022"
agent: "build"
permission:
  edit:
    "*.ts": "allow"
    "*.md": "allow"
  bash:
    "*": "ask"
  read:
    "*": "allow"
provider:
  anthropic:
    apiKey: "your-api-key-here"
---

# OpenCode 配置

这是项目的 OpenCode 配置文件。
EOF

# 设置环境变量（推荐方式）
cat >> ~/.zshrc << 'EOF'

# OpenCode 配置
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
export OPENAI_API_KEY="sk-your-key-here"
EOF

source ~/.zshrc

# 验证配置
opencode config get model
# 输出：anthropic/claude-3-5-sonnet-20241022
```

**获取 API 密钥**：

1. **Anthropic (Claude)**
   - 访问：https://console.anthropic.com/
   - 注册账号并创建 API 密钥
   - 复制密钥：`sk-ant-...`

2. **OpenAI (GPT)**
   - 访问：https://platform.openai.com/api-keys
   - 创建 API 密钥
   - 复制密钥：`sk-...`

### 步骤 7：运行第一个会话

```bash
# 方式1：命令行模式
# 创建新会话
opencode session new --agent build
# 输出：Session created: sess_abc123def456

# 发送消息
opencode session prompt sess_abc123def456 "Hello, OpenCode!"
# 输出：
# AI: Hello! I'm OpenCode, an AI coding assistant. 
# How can I help you with your code today?

# 方式2：TUI 模式（推荐）
opencode tui

# TUI 界面启动：
# ┌─────────────────────────────────────┐
# │ OpenCode TUI                        │
# ├─────────────────────────────────────┤
# │ Sessions:                           │
# │   • New Session                     │
# │                                     │
# │ Prompt:                             │
# │ > _                                 │
# └─────────────────────────────────────┘

# 在 TUI 中输入：
# > 创建一个 hello.ts 文件，输出 "Hello, World!"

# AI 响应并执行工具：
# [Tool: write]
# path: hello.ts
# content: console.log("Hello, World!")
# 
# [Permission Request]
# Allow write to hello.ts? (y/n)
# > y
# 
# File created successfully!

# 验证文件
cat hello.ts
# 输出：console.log("Hello, World!")

# 运行文件
bun run hello.ts
# 输出：Hello, World!
```

### 常见问题排查

**问题1：Bun 安装失败**

```bash
# 症状
curl: (7) Failed to connect to bun.sh

# 原因：网络问题或防火墙

# 解决方案1：使用代理
export https_proxy=http://127.0.0.1:7890
curl -fsSL https://bun.sh/install | bash

# 解决方案2：手动下载
# 访问 https://github.com/oven-sh/bun/releases
# 下载对应平台的二进制文件
# 解压到 ~/.bun/bin/
```

**问题2：依赖安装失败**

```bash
# 症状
error: Failed to download package

# 解决方案：清理缓存
rm -rf node_modules bun.lock ~/.bun/install/cache
bun install

# 如果仍然失败，使用 npm
npm install
```

**问题3：构建失败**

```bash
# 症状
error TS2307: Cannot find module

# 原因：TypeScript 版本不兼容

# 解决方案：检查 Node.js 版本
node --version
# 需要 >= 18.0.0

# 升级 Node.js
nvm install 20
nvm use 20
```

**问题4：API 密钥未配置**

```bash
# 症状
Error: API key not found

# 解决方案：设置环境变量
export ANTHROPIC_API_KEY="sk-ant-..."

# 或在配置文件中设置
# .opencode/opencode.md
---
provider:
  anthropic:
    apiKey: "sk-ant-..."
---
```

**问题5：权限被拒绝**

```bash
# 症状
Permission denied: write to file.ts

# 原因：权限配置为 "deny"

# 解决方案：修改配置
# .opencode/opencode.md
---
permission:
  edit:
    "*.ts": "allow"  # 改为 allow
---
```

### 开发工具推荐

**VSCode 扩展**：

```bash
# 安装推荐扩展
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension oven.bun-vscode
code --install-extension bradlc.vscode-tailwindcss

# 配置 VSCode
mkdir -p .vscode
cat > .vscode/settings.json << 'EOF'
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
EOF

# 配置推荐扩展
cat > .vscode/extensions.json << 'EOF'
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "oven.bun-vscode"
  ]
}
EOF
```

**终端工具**：

```bash
# 安装 Oh My Zsh（可选）
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# 安装 zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# 安装 zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# 启用插件
# 编辑 ~/.zshrc
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

---

## 🎉 恭喜！环境搭建完成

现在你已经完成了 OpenCode 开发环境的搭建，可以开始学习后续章节了。

**下一步**：
- 📖 阅读 [1.1 Monorepo 架构设计](./01-monorepo.md)
- 💻 完成第1章的练习题
- 🔍 探索 OpenCode 源码结构

**遇到问题？**
- 📚 查看 [故障排除指南](../../appendix/D-troubleshooting.md)
- 💬 加入 [Discord 社区](https://discord.gg/opencode)
- 🐛 提交 [GitHub Issue](https://github.com/anomalyco/opencode/issues)

**学习建议**：
- ✅ 不要急于求成，扎实掌握每个概念
- ✅ 多动手实践，运行所有代码示例
- ✅ 完成每章的练习题
- ✅ 记录学习笔记和遇到的问题
- ✅ 参与社区讨论，分享学习心得

祝你学习愉快！🚀
