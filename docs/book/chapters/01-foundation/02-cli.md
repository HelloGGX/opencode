# 1.2 CLI 骨架：一条命令的完整生命周期

## 2.0 引言：从 opencode --version 说起

想象你是 OpenCode 的新用户。你刚刚看到 GitHub 上的 README，按照说明执行了安装命令，我们以bun 命令为例：

```bash
$ bun add -g opencode-ai
```

接着会输出安装成功的信息：

```bash
bun add -g opencode-ai
bun add v1.3.9 (cf6cdbbb)
installed opencode-ai@1.1.59 with binaries:
- opencode
```

随着屏幕上一阵字符滚动，终端提示安装成功。紧接着，你习惯性地输入了验证命令：

```bash
$ opencode --version
1.1.59
```

屏幕上准确无误地打印出了版本号。这看起来是一个再寻常不过的场景，但我们不妨深入思考一下：当我们在终端敲下 opencode 并按下回车时，底层究竟发生了什么？

操作系统是如何在一个浩如烟海的文件系统中，精准找到属于 OpenCode 的可执行代码的？一个普通的 JavaScript 文本文件，又是如何摇身一变，成为系统级指令的？

本章我们将从第一性原理出发，追踪这条命令的完整生命周期，详细解读从用户按下回车的那一刻，到屏幕上显示版本号其背后的技术细节。更重要的是，你会理解为什么要这样设计，以及在构建自己的 CLI 工具时如何做出正确的决策。

## 2.1 寻址：操作系统如何找到 opencode？

### 2.1.1 环境变量与符号链接（Symbolic Link）

当我们在终端输入一个非内置的系统命令时，操作系统必须要知道这个程序实体存放在硬盘的具体位置。为了追踪它的真实路径，我们可以借助 shell 的内置命令 type 来看看：

```bash
$ type opencode
opencode is /Users/gavin/.bun/bin/opencode
```

输出结果清晰地指明了一个绝对路径。那么，shell 是如何知道去 /Users/gavin/.bun/bin/ 这个目录下查找的呢？这里就是 PATH 环境变量发挥作用的地方。

我们可以输出命令：

```bash
echo $PATH
```

你会看到类似这样的输出：

```bash
/Users/gavin/.bun/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

PATH 环境变量是一个以冒号分隔的目录列表。你可以将其理解为一个“全局搜索目录白名单”。当我们执行外部命令时，shell 会按照 PATH 中定义的目录顺序（以冒号分隔）逐一遍历，拼接并检查文件是否存在且具备可执行权限。一旦找到匹配项，就会调用 execve() 系统调用将其交由内核加载。

在我的环境中，shell 找到的路径是：

```bash
/Users/gavin/.bun/bin/opencode
```

但这还没有结束。如果我们继续追踪这个路径下的文件属性：

```bash
$ ls -la /Users/gavin/.bun/bin/opencode
lrwxrwxrwx@ 1 gavin  staff  55  2 12 10:55 /Users/gavin/.bun/bin/opencode -> ../install/global/node_modules/opencode-ai/bin/opencode
```

细心的你一定注意到了输出结果开头的字母 l，以及路径末尾的箭头 ->。这说明，/Users/gavin/.bun/bin/opencode 根本就不是一个真实的代码文件，而是一个符号链接（Symbolic Link）。它本质上是一个“引用”，指向了包的实际安装目录：

```bash
/Users/gavin/.bun/install/global/node_modules/opencode-ai/bin/opencode
```

我们可以通过下面的命令再次验证符号关系：

```bash
$ readlink /Users/gavin/.bun/bin/opencode
../install/global/node_modules/opencode-ai/bin/opencode
```

你看，这就是符号链接的作用。它就像一个指向另一个文件的指针，而不是文件本身。这种设计确保了"安装位置"与"实际位置"的解耦。包管理器保持目标路径稳定，替换该路径下的内容，而符号链接保持不变。

当你运行 `bun update opencode-ai` 时：

1. bun 更新 `node_modules/opencode/` 中的文件
2. 符号链接指向的路径不变
3. 下次运行 `opencode` 时自动使用新版本

这也是现代包管理器实现全局命令机制的通用设计：用一个稳定的入口路径，指向可被替换的版本目录。

### 2.1.2 桥接的纽带：package.json

但这引出了下一个问题：包管理器是怎么知道要创建 opencode 这个命令名的？答案在 `package.json` 中。

```json
// packages/opencode/package.json
{
  "name": "opencode",
  "bin": {
    "opencode": "./bin/opencode"
  }
}
```

这个配置告诉bun："当用户安装这个包时，请创建一个名为 `opencode` 的命令，指向 `./bin/opencode` 文件,也就是上一节我们提到的符号链接：`~/.bun/bin/opencode` , 该链接直接指向该包的实际安装目录：`~/.bun/install/global/node_modules/opencode-ai`。

### 2.1.3 实验：验证这个机制

让我们创建一个最简单的可执行包来验证这个机制：

```bash
# 创建测试目录
mkdir my-cli-test
cd my-cli-test

# 创建 package.json
cat > package.json << 'EOF'
{
  "name": "my-cli-test",
  "bin": {
    "hello": "./bin/hello"
  }
}
EOF

# 创建 bin 目录和可执行文件
mkdir bin
cat > bin/hello << 'EOF'
#!/usr/bin/env node
console.log("Hello from my CLI!")
EOF

# 赋予执行权限
chmod +x bin/hello

# 本地安装（创建符号链接）
npm link

# 测试
hello
# 输出: Hello from my CLI!
```

**现在你理解了第一个关键机制：`package.json` 的 `bin` 字段让普通文件变成了全局命令。**

现在我们深入这个可执行文件 `bin/opencode`，看看到底写了什么。

## 2.2 Shebang - 让文本文件变成可执行程序

现在我们知道符号链接指向了 `bin/opencode`，我们使用 cat 命令查看它的内容：

```bash
#!/usr/bin/env node
const childProcess = require("child_process")
```

这看起来很奇怪，第一行的 #!/usr/bin/env node 显然不是合法的 JS 代码。那么，V8 引擎在解析它时为什么没有抛出语法错误（SyntaxError）？

其实，这段代码首先面对的并不是 JS 引擎，而是操作系统的内核。

`#!/usr/bin/env node` 被称为 **Shebang**（也叫 Hashbang），它是 Unix/Linux 系统的一个特殊机制。在 Unix/Linux 系统中，当内核尝试加载一个文件时，会检查文件头部的前两个字节。如果发现是 0x23 和 0x21（即 #!），内核便会意识到：“这是一个纯文本脚本，我不能直接运行它，我需要调用后面的路径来解释它。”

> **为什么叫 Shebang？**
>
> - `#` 读作 "sharp" 或 "hash"
> - `!` 读作 "bang"
> - 合起来就是 "shebang"

### 2.2.3 寻找解释器：为什么是 /usr/bin/env

直觉上，既然我们需要 Node.js 环境，直接写绝对路径似乎是最严谨的：

```bash
#!/usr/local/bin/node
```

但这种硬编码（Hardcoding）方案在真实的软件工程中是极其脆弱的。在 macOS 上，Node 可能安装在 /usr/local/bin；在 Linux 上，可能在 /usr/bin；如果用户使用了 NVM，路径又会深藏在用户的 Home 目录中。

为了抹平这种环境碎片化，业界约定俗成的最佳实践是使用 env 工具：#!/usr/bin/env node。这样做的巧妙之处在于，内核会先调用系统自带的 env 程序，再由 env 程序去当前用户的 PATH 环境变量中动态搜寻 node 所在的位置。这是一种非常优雅的**“动态决议（Dynamic Resolution）”**策略。

### 2.2.4 实验：理解 Shebang 的作用

让我们创建两个文件对比：

**没有 Shebang 的文件：**

```bash
# 创建文件
cat > no-shebang.js << 'EOF'
console.log("Hello without shebang")
EOF

# 尝试直接执行
chmod +x no-shebang.js
./no-shebang.js
# 错误: ./no-shebang.js: line 1: syntax error near unexpected token `"Hello without shebang"'
```

**有 Shebang 的文件：**

```bash
# 创建文件
cat > with-shebang.js << 'EOF'
#!/usr/bin/env node
console.log("Hello with shebang")
EOF

# 直接执行
chmod +x with-shebang.js
./with-shebang.js
# 输出: Hello with shebang
```

**关键洞察：Shebang 让文本文件变成了可执行程序。**

## 2.3 启动器模式（Launcher Pattern）：解耦的艺术

在理清了系统执行机制后，我们回过头来审视 CLI 工具本身的架构设计。

既然 bin/opencode 是最终的入口，为了方便开发，我们很容易写出如下的架构：直接在入口文件中引入所有的业务逻辑。

```javascript
#!/usr/bin/env node
// 糟糕的设计示范：入口即业务
const { startAIContext } = require("../src/core/ai")
const { parseArgs } = require("../src/utils/parser")

const args = parseArgs(process.argv)
startAIContext(args)
```

这段代码在本地运行良好。但如果我们将其投入生产环境，会面临两个严峻的挑战：

1. 性能损耗：对于大型 CLI 工具，庞大的 JS 代码量会导致 Node.js 启动时产生明显的冷启动延迟（JIT 编译耗时）。
2. 多语言/多架构分发：为了极致的性能，现代 CLI（如 OpenCode）往往会使用 Rust、Go 或 Bun 将核心逻辑预编译为机器码（Binary）。此时，针对不同的操作系统（macOS, Linux, Windows）和 CPU 架构（x64, arm64），我们会生成多个不同的二进制产物。

如果入口直接绑定了业务代码，我们该如何根据用户的当前环境，动态地执行对应的二进制文件呢？

为了解决这个难题，我们需要引入一个中间层——启动器模式（Launcher Pattern）。

在这种模式下，入口文件（bin/opencode）被彻底剥夺了业务处理能力，它的职责被缩减为一个纯粹的环境监测与路由器。

### 2.3.1 实现一个基础启动器

我们需要在启动器中创建一个 run 函数，用来拉起真正的目标程序。一开始，我们可能会写出这样简单的代码：

```javascript
// 第一版：基础启动器（存在缺陷）
const childProcess = require("child_process")

function run(targetPath) {
  // 衍生子进程执行真实的业务产物
  childProcess.spawnSync(targetPath, process.argv.slice(2))
}
```

这段代码利用 child_process.spawnSync 将用户传入的命令行参数（process.argv.slice(2)）透传给目标程序。但它存在一个致命的问题：由于子进程的标准输入输出（I/O）没有与当前终端建立联系，CLI 打印的各种漂亮的高亮颜色和交互式提示符都将丢失（即：响应丢失）。

为了解决这个问题，我们需要在子进程与主进程之间建立“管道”：

```javascript
// 第二版：完善标准输出映射
const childProcess = require("child_process")

function run(targetPath) {
  const result = childProcess.spawnSync(target, process.argv.slice(2), {
    // 关键设计：将子进程的 stdio 继承到父进程
    stdio: "inherit",
  })
  // 处理进程状态码，保证信号透传
  if (result.error) {
    console.error(result.error.message)
    process.exit(1)
  }

  // 优雅退出，透传业务代码的退出码
  const code = typeof result.status === "number" ? result.status : 0
  process.exit(code)
}
```

通过配置 stdio: "inherit"，我们将主进程的 stdin, stdout, stderr 完全代理给了子进程。此时，用户在终端的视觉体验与直接运行目标程序完全一致。启动器成功隐形了。

### 2.3.2 动态架构路由：精准分发

既然拥有了执行器，下一步就是如何找到正确的二进制文件。这就需要用到 Node.js 提供的内置模块 os。
最容易想到的办法是，直接根据操作系统的平台（Platform）和架构（Arch）进行字符串拼接：

```javascript
// 第一版：基础路由（过于理想化）
const os = require("os")

const platformMap = { darwin: "darwin", linux: "linux", win32: "windows" }
const archMap = { x64: "x64", arm64: "arm64", arm: "arm" }

const platform = platformMap[os.platform()] || os.platform()
const arch = archMap[os.arch()] || os.arch()

const binaryName = platform === "windows" ? "opencode.exe" : "opencode"
const basePackageName = `opencode-${platform}-${arch}` // 例如: opencode-darwin-arm64
```

这段代码看似合理，但如果我们直接将其作为最终方案，在日常开发和线上运维时就会遇到麻烦。比如，当核心开发者在本地编译了一个新的二进制包想要测试时，难道要每次都去替换 node_modules 里的文件吗？

优秀的底层工具一定会提供“逃生舱（Escape Hatch）”。在执行默认的路由逻辑之前，我们需要允许用户通过环境变量或特定文件来强行指定二进制文件的路径。

```javascript
// 插入在基础路由之前的逻辑
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath) // 优先读取环境变量
}

// 检查同级目录下是否存在 .opencode 缓存/标识文件
const scriptPath = fs.realpathSync(__filename)
const scriptDir = path.dirname(scriptPath)
const cached = path.join(scriptDir, ".opencode")

if (fs.existsSync(cached)) {
  run(cached)
}
```

> **`fs.realpathSync` 的作用：** 解析符号链接，获取脚本的真实路径。这确保即使启动器通过符号链接调用，也能正确定位脚本所在目录。

你看，通过增加这两步，我们将控制权交还给了开发者。只有当这两者都不存在时，我们才继续执行后续的平台匹配逻辑。缓存机制则确保了在后续的运行中，我们可以快速定位到正确的二进制文件，而无需重复计算。

### 2.3.3 深入硬件与系统底层：AVX2 与 Musl

刚才构建的 opencode-linux-x64 真的能覆盖所有 Linux 的 x64 机器吗？

当我们把基于上述逻辑的 CLI 发布后，很快就会收到两类 Issue 报错：一类是 Illegal instruction (core dumped)，另一类是 Error: unsupported architecture or missing dynamic link library。
这暴露了两个更底层的环境差异问题：

#### 第一个问题来自 CPU 指令集。

现代 AI 相关的底层运算为了追求极致的性能，通常会利用 CPU 的 SIMD（单指令多数据流）扩展指令集，其中最核心的就是 AVX2。但是，并非所有 x64 架构的 CPU 都支持 AVX2（比如一些老旧的服务器或廉价 VPS）。如果让不支持 AVX2 的 CPU 强行运行包含了 AVX2 指令的二进制程序，就会直接触发非法指令错误。

为了解决这个问题，我们需要引入一个探针函数 supportsAvx2()，通过读取操作系统的硬件信息来判断：

```javascript
function supportsAvx2() {
  if (arch !== "x64") return false // AVX2 是 x86 架构特有的

  if (platform === "linux") {
    try {
      // 在 Linux 下，直接读取 /proc/cpuinfo 是最可靠的做法
      return /(^|\s)avx2(\s|$)/i.test(fs.readFileSync("/proc/cpuinfo", "utf8"))
    } catch {
      return false
    }
  }

  if (platform === "darwin") {
    // macOS 下通过 sysctl 系统调用获取
    const result = childProcess.spawnSync("sysctl", ["-n", "hw.optional.avx2_0"], { encoding: "utf8" })
    return (result.stdout || "").trim() === "1"
  }

  // Windows 下则需要通过 PowerShell 调用 Kernel32.dll 的 API
  // 篇幅所限，此处省略具体的 PowerShell 注入代码...
  return false
}
```

#### 第二个问题来自 C 语言标准库（libc）。

在 Linux 生态中，绝大多数发行版（如 Ubuntu, CentOS）使用的是 glibc。但近年来，以极小体积著称的 Alpine Linux 在 Docker 容器中变得极为流行，而 Alpine 使用的是 musl libc。用 glibc 编译的二进制文件是无法在 Alpine 上运行的。

因此，在 Linux 平台下，我们还必须检测当前系统是否是 musl 环境：

```javascript
function isMusl() {
  try {
    if (fs.existsSync("/etc/alpine-release")) return true
  } catch {
    // ignore
  }
  try {
    const result = childProcess.spawnSync("ldd", ["--version"], { encoding: "utf8" })
    const text = ((result.stdout || "") + (result.stderr || "")).toLowerCase()
    if (text.includes("musl")) return true
  } catch {
    // ignore
  }
  return false
}
```

### 2.3.4 降级策略：构建后备依赖队列
明确了环境差异后，我们不能仅仅拼出一个包名就结束了。试想，如果系统支持 AVX2，但包管理器因为网络问题只下载了 Baseline（基础版）的包，程序是不是就该直接崩溃？
更稳妥的设计是降级（Fallback）策略。我们需要构建一个数组，按照“最优匹配 -> 次优匹配 -> 基础兼容”的顺序，生成一个可能存在的包名列表。

```javascript
const names = (() => {
  const avx2 = supportsAvx2();
  const baseline = arch === "x64" && !avx2;

  // 以 Linux 为例，构建完整的降级队列
   if (platform === "linux") {
    if (isMusl) {
      if (arch === "x64") {
        if (baseline) return [`${base}-baseline-musl`, `${base}-musl`, `${base}-baseline`, base]
        return [`${base}-musl`, `${base}-baseline-musl`, base, `${base}-baseline`]
      }
      return [`${base}-musl`, base]
    }

    if (arch === "x64") {
      if (baseline) return [`${base}-baseline`, base, `${base}-baseline-musl`, `${base}-musl`]
      return [base, `${base}-baseline`, `${base}-musl`, `${base}-baseline-musl`]
    }
    return [base, `${base}-musl`]
  }
  
  // macOS / Windows 的处理
  if (arch === "x64") {
    if (baseline) return [`${base}-baseline`, base];
    return [base, `${base}-baseline`];
  }
  return [base];
})();
```
你看这个 names 数组的返回值，它体现了一种极具韧性的工程思维：哪怕当前是最特殊的 Linux + x64 + 不支持 AVX2 + musl 环境，它也会优先找 -baseline-musl 的包；如果找不到，再退而求其次，一层层回退，直到最后尝试标准的 base 包。


### 2.3.5 目录穿透：应对 Monorepo 的包提升

万事俱备，最后一步就是拿着这个 names 数组去文件系统里找真实的可执行文件了。

按照常规思路，包的结构应该是固定的，直接去同级的 node_modules 下寻找即可。但别忘了，现代前端工程广泛采用 pnpm 或 Yarn workspace 等 Monorepo 方案。这意味着，opencode-linux-x64 这个底层的 binding 包，可能并不会老老实实地待在当前目录的 node_modules 里，而是被“提升（Hoisted）”到了更外层的项目根目录中。

因此，我们需要编写一个基于 while(true) 或 for(;;) 的向上遍历算法，逐层探测 node_modules：

```javascript
function findBinary(startDir) {
  let current = startDir
  for (;;) {
    const modules = path.join(current, "node_modules")
    if (fs.existsSync(modules)) {
      for (const name of names) {
        const candidate = path.join(modules, name, "bin", binary)
        if (fs.existsSync(candidate)) return candidate
      }
    }
    const parent = path.dirname(current)
    if (parent === current) {
      return
    }
    current = parent
  }
}

const resolved = findBinary(scriptDir)
if (!resolved) {
  console.error(
    "It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing " +
      names.map((n) => `\"${n}\"`).join(" or ") +
      " package",
  )
  process.exit(1)
}
```
至此，一个工业级的启动器（Launcher）就完整闭环了。

回顾整个过程，你会发现，从最初几行的 childProcess.spawnSync，演变到需要处理环境变量、架构差异、硬件指令集、C 标准库、再到应对包管理器的提升机制。代码量增加了十倍不止。

这不是为了复杂而复杂。底层的工具代码之所以长成这样，是被无数个极端的真实用户环境“逼”出来的。理解了这些，你也就真正理解了现代跨平台 CLI 工具设计的核心思想。

## 2.4 总结

现在我们可以画出 `opencode --version` 的完整执行流程：

```
用户输入: opencode --version
    ↓
操作系统在 PATH 中查找 opencode
    ↓
找到: /usr/local/bin/opencode (符号链接)
    ↓
指向: node_modules/opencode/bin/opencode
    ↓
Node.js 执行启动器脚本
    ↓
检查 OPENCODE_BIN_PATH 环境变量?
    ├─ 是 → 直接执行指定路径
    └─ 否 → 继续
        ↓
    检查缓存文件 .opencode 是否存在?
        ├─ 是 → 直接执行缓存
        └─ 否 → 继续
            ↓
        检测平台和架构 (如: darwin + arm64)
            ↓
        x64 架构? 检测 AVX2 指令集支持
            ↓
        Linux 平台? 检测 musl libc
            ↓
        生成候选包名列表 (按优先级排序)
            ↓
        向上查找 node_modules
            ↓
        找到二进制文件: node_modules/opencode-darwin-arm64/bin/opencode
            ↓
        使用 spawnSync 执行
            ↓
        传递参数: ["--version"]
            ↓
        继承 stdio (用户看到输出)
            ↓
        等待执行完成
            ↓
        传递退出码
            ↓
        用户看到: opencode version 1.1.39
```


下一章我们将介绍 OpenCode CLI 的完整架构。并逐步实现核心功能。


## 代码附录

```javascript
#!/usr/bin/env node

const childProcess = require("child_process")
const fs = require("fs")
const path = require("path")
const os = require("os")

function run(target) {
  const result = childProcess.spawnSync(target, process.argv.slice(2), {
    stdio: "inherit",
  })
  if (result.error) {
    console.error(result.error.message)
    process.exit(1)
  }
  const code = typeof result.status === "number" ? result.status : 0
  process.exit(code)
}

const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath)
}

const scriptPath = fs.realpathSync(__filename)
const scriptDir = path.dirname(scriptPath)

//
const cached = path.join(scriptDir, ".opencode")
if (fs.existsSync(cached)) {
  run(cached)
}

const platformMap = {
  darwin: "darwin",
  linux: "linux",
  win32: "windows",
}
const archMap = {
  x64: "x64",
  arm64: "arm64",
  arm: "arm",
}

let platform = platformMap[os.platform()]
if (!platform) {
  platform = os.platform()
}
let arch = archMap[os.arch()]
if (!arch) {
  arch = os.arch()
}
const base = "opencode-" + platform + "-" + arch
const binary = platform === "windows" ? "opencode.exe" : "opencode"

function supportsAvx2() {
  if (arch !== "x64") return false

  if (platform === "linux") {
    try {
      return /(^|\s)avx2(\s|$)/i.test(fs.readFileSync("/proc/cpuinfo", "utf8"))
    } catch {
      return false
    }
  }

  if (platform === "darwin") {
    try {
      const result = childProcess.spawnSync("sysctl", ["-n", "hw.optional.avx2_0"], {
        encoding: "utf8",
        timeout: 1500,
      })
      if (result.status !== 0) return false
      return (result.stdout || "").trim() === "1"
    } catch {
      return false
    }
  }

  if (platform === "windows") {
    const cmd =
      '(Add-Type -MemberDefinition "[DllImport(""kernel32.dll"")] public static extern bool IsProcessorFeaturePresent(int ProcessorFeature);" -Name Kernel32 -Namespace Win32 -PassThru)::IsProcessorFeaturePresent(40)'

    for (const exe of ["powershell.exe", "pwsh.exe", "pwsh", "powershell"]) {
      try {
        const result = childProcess.spawnSync(exe, ["-NoProfile", "-NonInteractive", "-Command", cmd], {
          encoding: "utf8",
          timeout: 3000,
          windowsHide: true,
        })
        if (result.status !== 0) continue
        const out = (result.stdout || "").trim().toLowerCase()
        if (out === "true" || out === "1") return true
        if (out === "false" || out === "0") return false
      } catch {
        continue
      }
    }

    return false
  }

  return false
}

const names = (() => {
  const avx2 = supportsAvx2()
  const baseline = arch === "x64" && !avx2

  if (platform === "linux") {
    const musl = (() => {
      try {
        if (fs.existsSync("/etc/alpine-release")) return true
      } catch {
        // ignore
      }

      try {
        const result = childProcess.spawnSync("ldd", ["--version"], { encoding: "utf8" })
        const text = ((result.stdout || "") + (result.stderr || "")).toLowerCase()
        if (text.includes("musl")) return true
      } catch {
        // ignore
      }

      return false
    })()

    if (musl) {
      if (arch === "x64") {
        if (baseline) return [`${base}-baseline-musl`, `${base}-musl`, `${base}-baseline`, base]
        return [`${base}-musl`, `${base}-baseline-musl`, base, `${base}-baseline`]
      }
      return [`${base}-musl`, base]
    }

    if (arch === "x64") {
      if (baseline) return [`${base}-baseline`, base, `${base}-baseline-musl`, `${base}-musl`]
      return [base, `${base}-baseline`, `${base}-musl`, `${base}-baseline-musl`]
    }
    return [base, `${base}-musl`]
  }

  if (arch === "x64") {
    if (baseline) return [`${base}-baseline`, base]
    return [base, `${base}-baseline`]
  }
  return [base]
})()

function findBinary(startDir) {
  let current = startDir
  for (;;) {
    const modules = path.join(current, "node_modules")
    if (fs.existsSync(modules)) {
      for (const name of names) {
        const candidate = path.join(modules, name, "bin", binary)
        if (fs.existsSync(candidate)) return candidate
      }
    }
    const parent = path.dirname(current)
    if (parent === current) {
      return
    }
    current = parent
  }
}

const resolved = findBinary(scriptDir)
if (!resolved) {
  console.error(
    "It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing " +
      names.map((n) => `\"${n}\"`).join(" or ") +
      " package",
  )
  process.exit(1)
}

run(resolved)

```


## 2.7 完整的 CLI 架构

现在我们可以总结 OpenCode CLI 的完整架构:

```
┌─────────────────────────────────────────────────────────┐
│                    OpenCode CLI 架构                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │  bin/opencode (启动器)                          │    │
│  │  - 平台检测                                      │    │
│  │  - 二进制文件查找                                │    │
│  │  - 参数透传                                      │    │
│  └──────────────┬─────────────────────────────────┘    │
│                 │                                        │
│                 ↓                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │  src/index.ts (主入口)                          │    │
│  │  - Yargs 初始化                                 │    │
│  │  - 中间件注册                                    │    │
│  │  - 命令注册                                      │    │
│  └──────────────┬─────────────────────────────────┘    │
│                 │                                        │
│                 ↓                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │  中间件层                                        │    │
│  │  - Log.init()                                   │    │
│  │  - 环境变量设置                                  │    │
│  │  - 命令记录                                      │    │
│  └──────────────┬─────────────────────────────────┘    │
│                 │                                        │
│                 ↓                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │  命令层 (28 个命令)                              │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │    │
│  │  │ models   │  │ session  │  │   mcp    │     │    │
│  │  └──────────┘  └──────────┘  └──────────┘     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │    │
│  │  │   tui    │  │   auth   │  │  github  │     │    │
│  │  └──────────┘  └──────────┘  └──────────┘     │    │
│  │  ... (22 个其他命令)                            │    │
│  └──────────────┬─────────────────────────────────┘    │
│                 │                                        │
│                 ↓                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │  核心业务层                                      │    │
│  │  - Instance.provide()                          │    │
│  │  - Provider.list()                             │    │
│  │  - Session.create()                            │    │
│  │  - ...                                          │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```
