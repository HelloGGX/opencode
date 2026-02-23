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
接着大多数人会输入：

```bash
$ opencode --version
1.1.59
```
这行简单的命令，背后发生了什么？

在本章中，我们将追踪这条命令的完整生命周期，详细解读从用户按下回车的那一刻，到屏幕上显示版本号其背后的技术细节。更重要的是，你会理解为什么要这样设计，以及在构建自己的 CLI 工具时如何做出正确的决策。

## 2.1 操作系统如何找到 opencode？

### 2.1.1 追踪可执行文件的位置

当你在终端输入 `opencode` 时，操作系统需要知道去哪里找这个程序。让我们用 `type` 命令追踪一下：

```bash
$ type opencode
opencode is /Users/gavin/.bun/bin/opencode
```
type 是 shell 内置命令，用于解析命令来源。type 命令返回了具体的文件路径, 这说明：opencode 被解析为一个外部命令，并且对应的路径是 `/Users/gavin/.bun/bin/opencode`。
那么shell 是如何找到这个路径的？这里就是 PATH 环境变量发挥作用的地方。

我们可以输出命令：
```bash
echo $PATH
```
你会看到类似这样的输出：

```bash
/Users/gavin/.bun/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```
PATH 环境变量是一个以冒号分隔的目录列表。
当 shell 需要解析一个外部命令时，它会按顺序遍历这些目录：
1. 将命令名拼接到目录后
2. 判断该路径是否存在
3. 检查是否具有可执行权限
4. 找到第一个匹配项后停止搜索
5. 调用 execve() 系统调用交给内核加载执行

在我的环境中，shell 找到的路径是：
```bash
/Users/gavin/.bun/bin/opencode
```
我们继续往下追踪。

```bash
$ ls -la /Users/gavin/.bun/bin/opencode
lrwxrwxrwx@ 1 gavin  staff  55  2 12 10:55 /Users/gavin/.bun/bin/opencode -> ../install/global/node_modules/opencode-ai/bin/opencode
```
第一列的第一个字符是 l，表示这是一个符号链接（symbolic link）。
它并不是一个普通文件，而是一个“指向另一条路径的引用”。箭头后面的路径是一个相对路径, 其中的 .. 表示上一级目录，结合当前目录是：`/Users/gavin/.bun/bin/`, 因此最终解析后的绝对路径是：
```bash
/Users/gavin/.bun/install/global/node_modules/opencode-ai/bin/opencode
```
这才是真正被加载执行的文件。

我们可以通过下面的命令再次验证符号关系：

```bash
$ readlink /Users/gavin/.bun/bin/opencode
../install/global/node_modules/opencode-ai/bin/opencode
```

符号链接类似于 Windows 的快捷方式或 macOS 的别名。它是一个指向另一个文件的特殊文件。当 shell 执行该路径时，内核在加载阶段会解析链接并访问最终目标文件。
这种设计确保了"安装位置"与"实际位置"的解耦。包管理器保持目标路径稳定，替换该路径下的内容，而符号链接保持不变。

当你运行 `bun update opencode-ai` 时：
1. bun 更新 `node_modules/opencode/` 中的文件
2. 符号链接指向的路径不变
3. 下次运行 `opencode` 时自动使用新版本

这也是现代包管理器实现全局命令机制的通用设计：用一个稳定的入口路径，指向可被替换的版本目录。

那么总结下，当你运行 bun install -g <package>时，会将包安装到全局目录，如 ~/.bun/install/global/node_modules/<package>, 并在 ~/.bun/bin/ 创建一个符号链接，指向该包的可执行文件。当我们执行 `opencode` 时，shell 会按照 PATH 环境变量的顺序查找，最终找到 `/Users/gavin/.bun/bin/opencode` 这个符号链接，内核会解析它并加载执行 `../install/global/node_modules/opencode-ai/bin/opencode` 这个文件。

但这引出了下一个问题：包管理器是怎么知道要创建 opencode 这个命令名的？

答案在 `package.json` 中。当用户执行 `bun add -g opencode-ai` 时，opencode-ai 包会被安装到全局目录, 此时bun会解析包内的package.json 文件，并读取其中的 bin 字段:

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

### 2.1.2 实验：验证这个机制

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

现在我们知道符号链接指向了 `bin/opencode`，让我们看看这个文件的内容：

```bash
$ cat ~/.bun/bin/opencode
#!/usr/bin/env node

const childProcess = require("child_process")
const fs = require("fs")
const path = require("path")
const os = require("os")

// ... 更多代码
function run(target) {}
```
这看起来很奇怪，opencode是一个文本文件， 第一行 `#!/usr/bin/env node` 不是有效的 JavaScript 语法，但文件中包含的是 JavaScript 代码。那么它是如何工作的？

`#!/usr/bin/env node` 被称为 **Shebang**（也叫 Hashbang），它是 Unix/Linux 系统的一个特殊机制。

#### 什么是 Shebang？
当我们在终端尝试执行一个文本文件时，操作系统的内核（Kernel）不仅仅是把它当作文本读取，它会首先检查文件头部的前两个字节。如果这两个字节是 0x23 和 0x21，也就是字符 # 和 !，内核就会意识到：“嘿，这不是普通的文本，这是一个需要解释器来执行的脚本。”

这两个字符组合 #! 被读作 Shebang（由 Sharp 和 Bang 组合而成）。紧随其后的字符串，则是内核需要调用的解释器路径。

**为什么叫 Shebang？**
- `#` 读作 "sharp" 或 "hash"
- `!` 读作 "bang"
- 合起来就是 "shebang"

### 2.2.3 寻找解释器：为什么是 /usr/bin/env

既然知道了 Shebang 的作用，通过它指定 Node.js 为解释器似乎顺理成章。直觉告诉我们，可以这样写：

```bash
#!/usr/local/bin/node
```
这是一个关键的设计决策。让我们对比两种写法：

**方案 A：硬编码路径**
```bash
#!/usr/local/bin/node
```
不同系统的 Node.js 安装路径不同
- macOS 可能在 `/usr/local/bin/node`
- Linux 可能在 `/usr/bin/node`
- 用户可能用 nvm 安装在 `~/.nvm/versions/node/...`

**方案 B：使用 env **
```bash
#!/usr/bin/env node
```
env 会在 PATH 环境变量中查找 bun，无论用户是如何安装 Bun （Homebrew, npm, 官方脚本），只要配置了 PATH 都能找到。OpenCode 选择了更加灵活的方案B，因为 CLI 工具需要在各种环境中运行。


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

### 2.2.5 跨平台执行的真相：环境差异的抹平

在上一节中，我们了解了 Shebang (`#!/usr/bin/env node`) 的作用。但此时，你脑海中可能会浮现出一个疑问：

> “Windows 操作系统根本不认识 Shebang 这种 Unix 规范，那 CLI 工具的跨平台执行究竟是如何成立的？”

实际上，答案并不是 Windows 突然提供了对 Shebang 的支持，而是 Node.js 生态在**安装阶段（Install Phase）**替我们补齐了这一层缺失。

当我们执行 `npm install -g opencode` 时，npm 会读取 `package.json` 中的 `bin` 字段。在 macOS 或 Linux 上，系统内核能够直接识别并执行带有 Shebang 的脚本；但在 Windows 上，npm 会采取一种**“包装器（Wrapper）”**策略——它会额外生成一个名为 `opencode.cmd` 的批处理文件。

这个 `.cmd` 文件本质上是一个"启动包装器"，npm 使用 [cmd-shim](https://github.com/npm/cmd-shim) 包生成实际的批处理脚本，其内容如下：

```cmd
@IF EXIST "%~dp0\node.exe" (
  "%~dp0\node.exe" "%~dp0\node_modules\opencode\bin\opencode" %*
) ELSE (
  @SETLOCAL
  @SET PATHEXT=%PATHEXT:;.JS;=%
  node "%~dp0\node_modules\opencode\bin\opencode" %*
)
```

**关键语法解析：**

| 语法 | 含义 |
|------|------|
| `%~dp0` | 批处理文件所在的目录（d=驱动器，p=路径） |
| `%*` | 所有命令行参数的透传 |
| `PATHEXT` | 移除 `.JS` 扩展名，防止 `node` 被解析为 `node.js` |
| `@IF EXIST` | 检查同目录是否存在 `node.exe` |

从用户的视角来看，他们执行的是 `opencode --version` 这个"可执行程序"；但从操作系统的视角来看，这其实是一条完整的调用链：

`opencode.cmd (Windows包装器) -> Node 解释器 -> bin/opencode -> 业务逻辑`

也就是说，跨平台的关键并不是“写一份代码到处运行”，而是**在安装阶段，为不同的操作系统生成相应的入口桥接层**。Shebang 解决了 Unix 环境下的“如何被执行”问题，而 `.cmd` 包装器则解决了 Windows 下的同类问题。

## 2.3 启动器模式（Launcher Pattern）：为什么我们需要一个“中间人”？

既然操作系统的差异已经被 npm 提供的包装器抹平，且无论哪种平台，最终都会将执行权交还给我们在 `bin` 字段中定义的入口文件（例如 `bin/opencode`）。那么我们自然会产生一个直觉上的想法：**为什么不直接把所有的业务代码写在这个入口文件里呢？**

或者，在使用了 TypeScript 或 Bun 等现代工具栈后，我们可能会顺理成章地写出这样的配置：

```json
// package.json
{
  "bin": {
    "opencode": "./src/index.ts"
  }
}

```

```typescript
#!/usr/bin/env bun
// src/index.ts
console.log("OpenCode version 1.0.0");
// ... 大量复杂的业务逻辑

```

这种方案看起来非常直观：直接将源码作为入口，既不需要多余的文件，也不需要复杂的目录结构。

**但这里存在几个致命的问题。**

首先是**平台架构的差异**。随着 CLI 工具的演进，为了追求极致的启动性能，我们通常会使用 Bun 或 pkg 将代码预编译为特定平台的二进制文件。这就意味着，对于 macOS (Apple Silicon) 我们需要分发 `opencode-darwin-arm64`，而对于 Windows 则需要分发 `opencode-windows-x64.exe`。如果我们直接把入口指向某一个具体的业务文件，系统在运行时要如何动态地选择正确的二进制产物呢？

其次是**运行时的性能负担**。如果像上面的代码那样直接执行 `.ts` 源码，运行时就需要进行即时编译（JIT）或转译操作。这对于强调“秒级响应”的 CLI 工具来说，带来了不可忽视的性能损耗。

为了解决这些问题，我们需要改变思路。既然 npm 可以用一个 `.cmd` 文件作为 Windows 的“系统级包装器”，我们为什么不为 CLI 程序设计一个**“应用级包装器”**呢？

这就是所谓的**启动器模式（Launcher Pattern）**。

在启动器模式下，`bin/opencode` 不再承担任何实际的业务逻辑，它的职责发生了转变，变成了一个纯粹的**“中间人”**或者说是**“路由器”**。

```javascript
#!/usr/bin/env node
// bin/opencode (启动器)

// 1. 检查当前运行时的操作系统和架构 (os.platform, os.arch)
// 2. 处理 ESM / CJS 的兼容性兜底
// 3. 拦截并处理一些致命的运行时异常
// 4. 动态组装路径，查找并调用真正对应的二进制产物或入口文件

```

通过引入这样一个启动器层，我们将“如何启动”和“做什么业务”彻底解耦。

* **对于开发阶段**：启动器可以根据环境变量动态指向本地构建的 `dist/index.js`。
* **对于生产阶段**：启动器能够精准地将任务派发给针对当前系统优化过的二进制产物。

这也完美契合了软件工程中的第一性原理：当一个问题因为耦合过深而难以解决时，就引入一个中间层。npm 引入了 `.cmd` 抹平了系统差异，而 CLI 架构则引入了启动器（Launcher）抹平了底层环境与业务产物之间的差异。

### 2.3.1 启动器的完整实现

让我们逐段分析 `bin/opencode` 的代码：

```javascript
#!/usr/bin/env node

const childProcess = require("child_process")
const fs = require("fs")
const path = require("path")
const os = require("os")
```

**为什么用 CommonJS 而不是 ESM？**
- 这个文件需要在各种 Node.js 版本上运行（包括旧版本）
- CommonJS 兼容性最好，不需要 `package.json` 配置
- 作为启动器，它的职责很简单，不需要复杂的模块系统

**核心函数：执行目标程序**

```javascript
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
```

**关键设计点：**

1. **`spawnSync` vs `spawn`：**
   - `spawnSync`：同步执行，阻塞直到子进程结束
   - `spawn`：异步执行，立即返回
   - CLI 工具需要阻塞式执行，所以用 `spawnSync`

2. **`process.argv.slice(2)`：**
   - `process.argv[0]`：node 可执行文件路径
   - `process.argv[1]`：当前脚本路径（bin/opencode）
   - `process.argv.slice(2)`：用户传入的参数
   - 例如：`opencode --version` → `["--version"]`

3. **`stdio: "inherit"`：**
   - 子进程继承父进程的标准输入/输出/错误流
   - 用户看到的输出就像直接执行二进制文件一样
   - 不需要手动转发输出

4. **退出码传递：**
   - 正确传递子进程的退出码
   - 这对 CI/CD 脚本很重要（非零退出码表示失败）

### 2.3.2 优先级 1：环境变量覆盖

```javascript
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath)
}
```

**设计洞察：环境变量是最高优先级**

这允许：
- 开发者测试本地构建：`OPENCODE_BIN_PATH=./dist/opencode opencode --version`
- CI/CD 使用特定版本：`OPENCODE_BIN_PATH=/custom/path/opencode`
- 企业环境的特殊部署需求

**这是 Unix 哲学的体现：环境变量是配置的最高优先级。**

### 2.3.3 缓存机制：快速路径优化

在环境变量检查之后，启动器还会检查是否存在缓存的二进制文件：

```javascript
const scriptPath = fs.realpathSync(__filename)
const scriptDir = path.dirname(scriptPath)

const cached = path.join(scriptDir, ".opencode")
if (fs.existsSync(cached)) {
  run(cached)
}
```

**为什么需要缓存机制？**

1. **性能优化：** 如果二进制文件已经被解压到 `.opencode` 缓存位置，直接执行，跳过后续复杂的查找逻辑
2. **首次运行后的加速：** 某些安装方式会将二进制文件预先解压到这个位置
3. **开发便利：** 本地开发时可以手动放置二进制文件进行测试

**`fs.realpathSync` 的作用：** 解析符号链接，获取脚本的真实路径。这确保即使启动器通过符号链接调用，也能正确定位脚本所在目录。

### 2.3.4 平台检测：找到正确的二进制文件

```javascript
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
```

**平台检测的设计考量：**

1. **映射表 vs 直接使用：**
   - Node.js 返回 `darwin`，但包名可能用 `macos`
   - 映射表提供了一层抽象，便于调整命名约定
   - 回退机制确保未知平台也能尝试运行

2. **二进制文件命名约定：**
   ```
   opencode-darwin-arm64      # macOS Apple Silicon
   opencode-darwin-x64        # macOS Intel
   opencode-linux-x64         # Linux x86_64
   opencode-linux-arm64       # Linux ARM64
   opencode-windows-x64.exe   # Windows x64
   ```

3. **为什么这样命名？**
   - 允许在 `node_modules` 中同时存在多个平台的二进制文件
   - npm/bun 可以根据平台只下载对应的包（可选依赖）
   - 便于 CI/CD 构建和分发

### 2.3.5 AVX2 检测：CPU 指令集优化

对于 x64 架构，启动器会检测 CPU 是否支持 AVX2 指令集，以决定使用哪个版本的二进制文件：

```javascript
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
```

**为什么需要 AVX2 检测？**

1. **性能差异：** AVX2 是高级向量扩展指令集，支持它的 CPU 可以运行优化版本的二进制文件，性能更好
2. **兼容性：** 不支持 AVX2 的老旧 CPU 需要使用 `baseline` 版本，否则程序会崩溃
3. **跨平台检测：** 不同操作系统检测方式不同：
   - **Linux：** 解析 `/proc/cpuinfo` 文件中的 flags
   - **macOS：** 调用 `sysctl` 命令查询 `hw.optional.avx2_0`
   - **Windows：** 通过 PowerShell 调用 Windows API `IsProcessorFeaturePresent(40)`

**Windows 检测的特殊处理：** 尝试多种 PowerShell 可执行文件名称（`powershell.exe`、`pwsh.exe`、`pwsh`、`powershell`），以适应不同 Windows 版本和 PowerShell 安装情况。

### 2.3.6 musl 检测：Alpine Linux 兼容性

对于 Linux 平台，启动器还会检测是否使用 musl libc（如 Alpine Linux）：

```javascript
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
```

**为什么需要 musl 检测？**

1. **libc 差异：** Linux 发行版使用两种主要的 C 标准库：
   - **glibc：** 大多数发行版（Ubuntu、Debian、CentOS 等）
   - **musl：** Alpine Linux 等轻量级发行版

2. **二进制不兼容：** 针对 glibc 编译的二进制文件无法在 musl 系统上运行，反之亦然

3. **检测策略：**
   - 首先检查 `/etc/alpine-release` 文件是否存在（Alpine 的特征文件）
   - 其次通过 `ldd --version` 输出判断是否包含 "musl" 字样

### 2.3.7 候选名称优先级：智能匹配策略

基于 AVX2 和 musl 检测结果，启动器会生成多个候选包名，按优先级排序：

```javascript
const names = (() => {
  const avx2 = supportsAvx2()
  const baseline = arch === "x64" && !avx2

  if (platform === "linux") {
    const musl = /* ... musl 检测逻辑 ... */

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
```

**候选名称示例：**

| 平台 | 架构 | AVX2 | musl | 候选名称列表 |
|------|------|------|------|-------------|
| Linux | x64 | ✅ | ❌ | `["opencode-linux-x64", "opencode-linux-x64-baseline", "opencode-linux-x64-musl", "opencode-linux-x64-baseline-musl"]` |
| Linux | x64 | ❌ | ❌ | `["opencode-linux-x64-baseline", "opencode-linux-x64", ...]` |
| Linux | x64 | ✅ | ✅ | `["opencode-linux-x64-musl", "opencode-linux-x64-baseline-musl", ...]` |
| Linux | arm64 | - | ✅ | `["opencode-linux-arm64-musl", "opencode-linux-arm64"]` |
| macOS | x64 | ✅ | - | `["opencode-darwin-x64", "opencode-darwin-x64-baseline"]` |
| macOS | arm64 | - | - | `["opencode-darwin-arm64"]` |

**设计洞察：** 优先级列表确保：
1. 优先使用最匹配的二进制文件（AVX2 优化版或 musl 版）
2. 如果最优选择不存在，回退到兼容版本
3. 最大化兼容性，减少"找不到二进制文件"的错误

### 2.3.8 向上递归查找：解决 Monorepo 问题

实际的 `findBinary` 函数使用预定义的 `names` 数组进行查找：

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
```

**为什么需要向上查找？**

考虑 Monorepo 场景：

```
/project
├── node_modules
│   ├── opencode
│   │   └── bin/opencode (启动器)
│   └── opencode-darwin-arm64
│       └── bin/opencode (真正的二进制文件)
└── packages
    └── my-app
        └── node_modules (可能为空)
```

当用户在 `/project/packages/my-app` 目录下执行 `opencode` 时：
- 启动器位于 `/project/node_modules/opencode/bin/opencode`
- 但二进制文件在 `/project/node_modules/opencode-darwin-arm64/bin/opencode`
- 向上查找确保能找到正确的二进制文件

**算法分析：**
1. 从当前目录开始
2. 检查 `node_modules` 是否存在
3. 按优先级遍历 `names` 数组中的候选包名
4. 检查 `bin/opencode` 或 `bin/opencode.exe` 是否存在
5. 如果找不到，向上一级目录继续查找
6. 直到找到或到达文件系统根目录

**时间复杂度：** O(d × m)，其中 d 是目录深度，m 是候选名称数量（通常 1-4 个）

### 2.3.9 错误处理：用户友好的提示

```javascript
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

**错误信息设计的三个原则：**

1. **说明问题：** "package manager failed to install"
2. **给出原因：** "for your platform"
3. **提供解决方案：** 列出所有可能的候选包名，用户可以选择安装

**对比糟糕的错误信息：**
```javascript
// ❌ 糟糕的错误信息
console.error("Binary not found")

// ✅ 好的错误信息（OpenCode 的做法）
console.error(
  'It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing "opencode-linux-x64" or "opencode-linux-x64-baseline" package'
)
```

### 2.3.10 完整流程图

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

**现在你理解了第三个关键机制：启动器模式通过一个轻量级的 Node.js 脚本，实现了跨平台的二进制文件分发和执行。**

---

## 策略 B：Bun 的预编译二进制模式

现在让我们看看另一种截然不同的策略。当你使用 Bun 安装 opencode 时，会发生什么？

### 2.3.12 Bun 安装的实际验证

在 Windows + Git Bash 环境中执行：

```bash
$ which opencode
/c/Users/Administrator/.bun/bin/opencode

$ ls -la /c/Users/Administrator/.bun/bin/opencode
-rwxr-xr-x 1 Administrator 197121 152492032 Dec 31 22:00 /c/Users/Administrator/.bun/bin/opencode
```

**惊人的发现：**

| 对比项 | npm 安装 | Bun 安装 |
|-------|---------|---------|
| **文件类型** | `lrwxr-xr-x`（符号链接） | `-rwxr-xr-x`（普通文件） |
| **文件大小** | ~2KB（脚本） | ~152MB（二进制） |
| **实际内容** | Node.js 启动器脚本 | Windows PE 可执行文件 |
| **是否需要 Node.js** | 是 | 否 |

### 2.3.13 为什么 Bun 不需要启动器？

Bun 采用了一种更现代化的分发策略：

```
npm install -g opencode:
  符号链接 → JS启动器 → 查找平台二进制 → 执行
  （多层包装，依赖 Node.js 运行时）

bun install -g opencode:
  直接下载预编译二进制 → 放到 PATH
  （单层，自包含，无需运行时）
```

**Bun 的优势：**

1. **零依赖**：可执行文件是自包含的，不需要系统预装 Node.js
2. **启动更快**：没有脚本解析和子进程创建的开销
3. **简化分发**：每个平台一个文件，无需复杂的启动器逻辑

**Bun 的实现原理：**

Bun 使用 `bun build --compile` 将 TypeScript/JavaScript 代码编译为原生二进制：

```typescript
// 开发时的源码
// packages/opencode/src/index.ts
import { cli } from "./cli"
cli.parse(process.argv)
```

```bash
# 构建时编译为原生二进制
bun build --compile --target=windows-x64 ./src/index.ts --outfile opencode-windows-x64

# 生成的 opencode-windows-x64 是一个完整的可执行文件
# 包含：Bun 运行时 + 你的代码 + 依赖
```

### 2.3.10 两种策略的权衡

| 维度 | npm 策略 | Bun 策略 |
|-----|---------|---------|
| **兼容性** | ✅ 更好（任何有 Node.js 的环境） | ⚠️ 需要对应平台的预编译版本 |
| **性能** | ⚠️ 有启动开销 | ✅ 原生速度 |
| **文件大小** | ✅ 小（几 KB 脚本） | ❌ 大（包含运行时，~150MB） |
| **复杂度** | ⚠️ 需要启动器逻辑 | ✅ 简单直接 |
| **调试** | ✅ 可阅读源码 | ⚠️ 二进制难以调试 |
| **生态系统** | ✅ npm 生态成熟 | ⚠️ Bun 生态较新 |

**选择建议：**

- **选择 npm 策略**：如果你的 CLI 需要支持各种环境（包括旧系统），或者需要让用户能轻松阅读/修改源码
- **选择 Bun 策略**：如果你追求极致性能，且目标用户愿意使用 Bun 作为包管理器

### 2.3.11 混合策略：未来的方向

实际上，OpenCode 项目采用了**混合策略**：

```
源码仓库
├── bin/opencode           # npm 策略：Node.js 启动器脚本
├── src/index.ts           # 源码入口
└── build/
    ├── opencode-darwin-arm64    # Bun 编译：macOS ARM
    ├── opencode-darwin-x64      # Bun 编译：macOS x64
    ├── opencode-linux-x64       # Bun 编译：Linux
    └── opencode-windows-x64.exe # Bun 编译：Windows
```

- **npm 用户**：获得启动器脚本，由脚本找到对应平台的二进制
- **Bun 用户**：直接获得对应平台的预编译二进制

这种设计兼顾了两者的优势。

---

但这引出了下一个问题：无论通过哪种方式启动，二进制文件接收到 `--version` 参数后，如何解析和处理？


## 2.4 第四站：参数解析 - 从手动到 Yargs

### 2.4.1 问题：二进制文件如何处理 --version？

现在我们的命令已经能够执行到真正的 OpenCode 二进制文件了。但二进制文件内部是如何处理 `--version` 参数的？

让我们从最简单的方式开始，逐步演进到 OpenCode 的实际实现。

**方案 1：手动解析 process.argv**

```typescript
// src/index.ts - 第一版（不推荐）
const args = process.argv.slice(2)

if (args[0] === "--version") {
  console.log("opencode version 1.0.0")
  process.exit(0)
}

if (args[0] === "--help") {
  console.log("Usage: opencode [options]")
  console.log("Options:")
  console.log("  --version  Show version")
  console.log("  --help     Show help")
  process.exit(0)
}

console.log("Hello, OpenCode!")
```

**这个方案的问题：**
- ❌ 无法处理 `-v` 短选项
- ❌ 无法处理 `--log-level=DEBUG` 这样的键值对
- ❌ 无法处理多个选项的组合
- ❌ 代码很快变得难以维护
- ❌ 没有类型安全

**当你需要支持 28 个命令时，这种方式完全不可行。**

### 2.4.2 技术选型：为什么选择 Yargs？

在 Node.js 生态中，有多个命令行解析库可选：

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **手动解析** | 零依赖、完全控制 | 代码量大、易出错 | 简单脚本 |
| **minimist** | 轻量(~5KB) | 功能有限、无类型 | 简单 CLI |
| **commander.js** | 简洁 API、流行 | 不支持中间件 | 中等复杂度 |
| **Yargs** | 功能完整、中间件、自动帮助 | 体积较大(~200KB) | 复杂 CLI |
| **oclif** | 企业级、插件系统 | 过于复杂、学习曲线陡 | 大型 CLI 框架 |

**OpenCode 选择 Yargs 的原因：**
- ✅ 支持子命令（28 个命令需要良好的组织）
- ✅ 中间件机制（统一的日志初始化）
- ✅ 自动生成帮助信息
- ✅ TypeScript 类型支持
- ✅ 成熟稳定，社区活跃

**这是一个"体积 vs 功能"的权衡：**
- 对于简单工具，200KB 可能太重
- 对于复杂 CLI（28 个命令），这个代价是值得的

### 2.4.3 Yargs 基础：从 Hello World 开始

让我们从最简单的例子开始理解 Yargs：

```typescript
// src/index.ts - 使用 Yargs
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

const cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .version("1.0.0")
  .help()

cli.parse()
```

**关键概念解析：**

1. **`hideBin(process.argv)`：**
   ```typescript
   // process.argv 的内容：
   // ['/path/to/node', '/path/to/script', '--version']
   
   // hideBin 移除前两个元素：
   // ['--version']
   ```
   这是 Yargs 的标准用法，只保留用户参数。

2. **链式调用（Builder Pattern）：**
   ```typescript
   yargs(...)
     .scriptName("opencode")  // 返回 yargs 实例
     .version("1.0.0")        // 返回 yargs 实例
     .help()                  // 返回 yargs 实例
   ```
   每个方法返回 `this`，支持链式调用。

3. **自动功能：**
   - `.version()` 自动处理 `--version` 和 `-v`
   - `.help()` 自动处理 `--help` 和 `-h`
   - 自动生成格式化的帮助信息

**测试一下：**

```bash
$ bun run src/index.ts --version
1.0.0

$ bun run src/index.ts --help
opencode

Options:
  --version  Show version number  [boolean]
  --help     Show help            [boolean]
```

**Yargs 自动为我们做了什么？**
- ✅ 解析 `--version` 和 `--help`
- ✅ 生成格式化的帮助信息
- ✅ 处理短选项别名（`-v`, `-h`）
- ✅ 验证参数类型


```javascript
// 平台和架构映射
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
  platform = os.platform()  // 回退到原始值
}
let arch = archMap[os.arch()]
if (!arch) {
  arch = os.arch()  // 回退到原始值
}
```

**平台检测的设计考量:**
- Node.js 的 `os.platform()` 返回 `darwin`、`linux`、`win32` 等
- 我们需要将其映射到二进制文件的命名约定
- 回退机制确保在未知平台上也能尝试运行

```javascript
const base = "opencode-" + platform + "-" + arch
const binary = platform === "windows" ? "opencode.exe" : "opencode"
```

**二进制文件命名约定:**
- `opencode-darwin-arm64` (macOS Apple Silicon)
- `opencode-linux-x64` (Linux x86_64)
- `opencode-windows-x64` (Windows x64)

这种命名方式允许在 `node_modules` 中同时存在多个平台的二进制文件。


```javascript
function findBinary(startDir) {
  let current = startDir
  for (;;) {
    const modules = path.join(current, "node_modules")
    if (fs.existsSync(modules)) {
      const entries = fs.readdirSync(modules)
      for (const entry of entries) {
        if (!entry.startsWith(base)) {
          continue
        }
        const candidate = path.join(modules, entry, "bin", binary)
        if (fs.existsSync(candidate)) {
          return candidate
        }
      }
    }
    const parent = path.dirname(current)
    if (parent === current) {
      return  // 到达文件系统根目录
    }
    current = parent
  }
}
```

**二进制文件查找算法:**

这个函数实现了一个**向上递归查找**的策略:

1. 从当前目录开始
2. 检查 `node_modules` 是否存在
3. 遍历所有以 `opencode-{platform}-{arch}` 开头的包
4. 检查 `bin/opencode` 或 `bin/opencode.exe` 是否存在
5. 如果找不到,向上一级目录继续查找
6. 直到找到或到达文件系统根目录

**为什么需要向上查找?**

考虑以下场景:

```
/project
├── node_modules
│   ├── opencode
│   │   └── bin/opencode (启动器)
│   └── opencode-darwin-arm64
│       └── bin/opencode (真正的二进制文件)
└── packages
    └── my-app
        └── node_modules (可能为空)
```

当用户在 `/project/packages/my-app` 目录下执行 `opencode` 时:
- 启动器位于 `/project/node_modules/opencode/bin/opencode`
- 但二进制文件在 `/project/node_modules/opencode-darwin-arm64/bin/opencode`
- 向上查找确保能找到正确的二进制文件


```javascript
const resolved = findBinary(scriptDir)
if (!resolved) {
  console.error(
    'It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing the "' +
      base +
      '" package',
  )
  process.exit(1)
}

run(resolved)
```

**错误处理的用户体验设计:**
- 提供清晰的错误信息,告诉用户问题所在
- 给出具体的解决方案(手动安装对应平台的包)
- 包含平台信息(`base` 变量),方便用户复制粘贴

### 2.2.3 完整流程图

```
用户执行: opencode --version
    ↓
操作系统查找 PATH 中的 opencode
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
    检测平台和架构
        ↓
    构造包名: opencode-darwin-arm64
        ↓
    向上查找 node_modules
        ↓
    找到二进制文件: node_modules/opencode-darwin-arm64/bin/opencode
        ↓
    使用 spawnSync 执行
        ↓
    传递参数: ["--version"]
        ↓
    继承 stdio
        ↓
    等待执行完成
        ↓
    传递退出码
```


### 2.2.4 练习:验证启动器

创建一个测试脚本来验证启动器的行为:

```bash
# 1. 测试环境变量覆盖
OPENCODE_BIN_PATH=/bin/echo opencode "Hello from env override"

# 2. 测试参数传递
opencode --help

# 3. 测试退出码
opencode some-invalid-command
echo $?  # 应该输出非零值
```

## 2.3 从简单到复杂:命令行参数解析的演进

现在我们有了可执行的入口,下一个问题是:**如何优雅地处理命令行参数?**

### 2.3.1 方案 1:手动解析 process.argv

最直接的方式是手动解析 `process.argv`:

```typescript
// src/index.ts - 第一版
const args = process.argv.slice(2)

if (args[0] === "--version") {
  console.log("opencode version 1.0.0")
  process.exit(0)
}

if (args[0] === "--help") {
  console.log("Usage: opencode [options]")
  console.log("Options:")
  console.log("  --version  Show version")
  console.log("  --help     Show help")
  process.exit(0)
}

console.log("Hello, OpenCode!")
```

**问题暴露:**
- ❌ 代码很快变得难以维护
- ❌ 没有类型安全
- ❌ 无法处理复杂的参数组合(如 `--log-level=DEBUG`)
- ❌ 错误处理繁琐


### 2.3.2 方案 2:使用 Yargs 框架

当我们需要支持 28 个命令时,手动解析显然不可行。这就是为什么 OpenCode 选择了 **Yargs**。

**为什么选择 Yargs 而不是其他方案?**

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **手动解析** | 零依赖、完全控制 | 代码量大、易出错 | 简单脚本 |
| **minimist** | 轻量(~5KB) | 功能有限、无类型 | 简单 CLI |
| **commander.js** | 简洁 API、流行 | 不支持中间件 | 中等复杂度 |
| **Yargs** | 功能完整、中间件、自动帮助 | 体积较大(~200KB) | 复杂 CLI |
| **oclif** | 企业级、插件系统 | 过于复杂、学习曲线陡 | 大型 CLI 框架 |

OpenCode 选择 Yargs 的原因:
- ✅ 支持子命令(28 个命令需要良好的组织)
- ✅ 中间件机制(统一的日志初始化)
- ✅ 自动生成帮助信息
- ✅ TypeScript 类型支持
- ✅ 成熟稳定,社区活跃

### 2.3.3 Yargs 基础:从 Hello World 开始

让我们从最简单的例子开始:

```typescript
// src/index.ts - 使用 Yargs
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

const cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .version("1.0.0")
  .help()

cli.parse()
```

**关键概念解析:**

1. **`hideBin(process.argv)`**:
   - `process.argv` 包含 `[node路径, 脚本路径, ...用户参数]`
   - `hideBin` 移除前两个元素,只保留用户参数
   - 这是 Yargs 的标准用法

2. **`scriptName("opencode")`**:
   - 设置帮助信息中显示的命令名称
   - 影响自动生成的使用说明

3. **链式调用**:
   - Yargs 使用 Builder 模式
   - 每个方法返回 `this`,支持链式调用


现在运行:

```bash
bun run src/index.ts --version
# 输出: 1.0.0

bun run src/index.ts --help
# 输出:
# opencode
#
# Options:
#   --version  Show version number  [boolean]
#   --help     Show help            [boolean]
```

**Yargs 自动为我们做了什么?**
- ✅ 解析 `--version` 和 `--help`
- ✅ 生成格式化的帮助信息
- ✅ 处理短选项别名(`-v`, `-h`)
- ✅ 验证参数类型

### 2.3.4 添加全局选项

OpenCode 需要两个全局选项:日志控制。

```typescript
const cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .version("1.0.0")
  .help()
  .option("print-logs", {
    describe: "print logs to stderr",
    type: "boolean",
  })
  .option("log-level", {
    describe: "log level",
    type: "string",
    choices: ["DEBUG", "INFO", "WARN", "ERROR"],
  })

cli.parse()
```

**选项定义的关键字段:**
- `describe`:帮助信息中的描述
- `type`:参数类型(`boolean`、`string`、`number`、`array`)
- `choices`:限制可选值(自动验证)

测试:

```bash
bun run src/index.ts --log-level=DEBUG
# Yargs 会验证值是否在 choices 中

bun run src/index.ts --log-level=INVALID
# 错误: Invalid values:
#   Argument: log-level, Given: "INVALID", Choices: "DEBUG", "INFO", "WARN", "ERROR"
```


## 2.5 第五站：命令系统 - 从一个到二十八个

### 2.5.1 问题：如何组织大量命令？

OpenCode 有 28 个命令：
- `opencode tui` - 启动 TUI 界面
- `opencode session new` - 创建会话
- `opencode models` - 列出模型
- `opencode mcp list` - 列出 MCP 服务器
- ...

如果把所有逻辑写在一个文件中，会导致：
- ❌ 文件过大（数千行）
- ❌ 难以维护
- ❌ 团队协作困难
- ❌ 无法独立测试

### 2.5.2 解决方案：命令模块化

Yargs 支持通过 `.command()` 注册子命令：

```typescript
// src/index.ts
import { ModelsCommand } from "./cli/cmd/models"
import { SessionCommand } from "./cli/cmd/session"
import { McpCommand } from "./cli/cmd/mcp"

const cli = yargs(hideBin(process.argv))
  .middleware(/* ... */)
  .command(ModelsCommand)
  .command(SessionCommand)
  .command(McpCommand)
  // ... 25 个其他命令

cli.parse()
```

每个命令是一个独立的模块，遵循统一的接口。

### 2.5.3 命令定义的四个关键部分

让我们通过 `ModelsCommand` 来理解命令的结构：

```typescript
// src/cli/cmd/models.ts
export const ModelsCommand = {
  command: "models [provider]",
  describe: "list all available models",
  builder: (yargs) => {
    return yargs
      .positional("provider", {
        describe: "provider ID to filter models by",
        type: "string",
      })
      .option("verbose", {
        describe: "use more verbose model output",
        type: "boolean",
      })
  },
  handler: async (args) => {
    // 命令实现
  },
}
```

#### 1. command：命令签名

```typescript
command: "models [provider]"
```

**语法规则：**
- `models`：命令名称（必需）
- `[provider]`：可选位置参数（用方括号包裹）
- `<required>`：必需位置参数（用尖括号包裹）

**示例：**
```bash
opencode models              # provider = undefined
opencode models openai       # provider = "openai"
```

#### 2. describe：命令描述

```typescript
describe: "list all available models"
```

这个描述会出现在：
- `opencode --help` 的命令列表中
- `opencode models --help` 的顶部

#### 3. builder：参数定义

```typescript
builder: (yargs) => {
  return yargs
    .positional("provider", {
      describe: "provider ID to filter models by",
      type: "string",
    })
    .option("verbose", {
      describe: "use more verbose model output",
      type: "boolean",
    })
}
```

**Builder 函数的职责：**
- 定义位置参数（`.positional()`）
- 定义选项参数（`.option()`）
- 设置参数验证规则
- 返回配置后的 yargs 实例

**位置参数 vs 选项参数：**
```bash
opencode models openai --verbose
#               ^^^^^^  ^^^^^^^^
#               位置参数  选项参数
```

#### 4. handler：命令实现

```typescript
handler: async (args) => {
  // args.provider: string | undefined
  // args.verbose: boolean | undefined
  
  await Instance.provide({
    directory: process.cwd(),
    async fn() {
      const providers = await Provider.list()
      
      if (args.provider) {
        const provider = providers[args.provider]
        if (!provider) {
          UI.error(`Provider not found: ${args.provider}`)
          return
        }
        printModels(args.provider, args.verbose)
        return
      }
      
      // 列出所有提供商的模型
      for (const providerID of Object.keys(providers)) {
        printModels(providerID, args.verbose)
      }
    },
  })
}
```

**Handler 函数的关键点：**

1. **类型安全的参数访问：**
   - `args.provider` 自动推断为 `string | undefined`
   - `args.verbose` 自动推断为 `boolean | undefined`
   - TypeScript 会在编译时检查类型错误

2. **实例隔离（Instance.provide）：**
   ```typescript
   await Instance.provide({
     directory: process.cwd(),
     async fn() { /* ... */ }
   })
   ```
   
   这是 OpenCode 的核心模式：
   - 每个命令在独立的实例上下文中执行
   - `directory` 指定项目目录
   - 实例销毁时自动清理资源
   - 避免不同项目的状态污染

3. **错误处理：**
   ```typescript
   if (!provider) {
     UI.error(`Provider not found: ${args.provider}`)
     return  // 提前返回，不继续执行
   }
   ```
   
   使用 `UI.error()` 而不是 `console.error()`：
   - 统一的错误输出格式
   - 支持颜色和样式
   - 可以被测试框架捕获

### 2.5.4 命令辅助函数：cmd()

你可能注意到，OpenCode 的实际代码使用了 `cmd()` 包装器：

```typescript
export const ModelsCommand = cmd({
  command: "models [provider]",
  describe: "list all available models",
  builder: (yargs) => { /* ... */ },
  handler: async (args) => { /* ... */ },
})
```

让我们看看 `cmd()` 的实现：

```typescript
// src/cli/cmd/cmd.ts
export function cmd<T>(definition: CommandModule<{}, T>): CommandModule<{}, T> {
  return {
    ...definition,
    handler: async (args) => {
      try {
        await definition.handler(args)
      } catch (error) {
        if (error instanceof NamedError) {
          UI.error(error.message)
          process.exit(1)
        }
        throw error
      }
    },
  }
}
```

**`cmd()` 的作用：**

1. **统一的错误处理：**
   - 捕获 `NamedError`（业务错误）
   - 使用 `UI.error()` 显示友好的错误信息
   - 退出码设置为 1

2. **保持类型安全：**
   - 泛型 `<T>` 保留原始参数类型
   - TypeScript 可以正确推断 `args` 的类型

3. **未捕获的错误会继续抛出：**
   - 系统错误（如内存溢出）不会被吞掉
   - 保留完整的堆栈跟踪

**为什么需要 `NamedError`？**

```typescript
// 业务错误：用户友好的错误信息
throw new NamedError("Provider not found: openai")
// 输出: ✖ Provider not found: openai

// 系统错误：保留堆栈跟踪
throw new Error("Unexpected null pointer")
// 输出: 完整的堆栈跟踪
```

这种区分让用户看到的错误信息更友好，同时保留了调试能力。


### 2.5.5 嵌套命令：session 子命令

OpenCode 有一些命令包含子命令，如 `session`：

```bash
opencode session new          # 创建会话
opencode session list         # 列出会话
opencode session prompt       # 发送消息
```

实现嵌套命令：

```typescript
// src/cli/cmd/session.ts
export const SessionCommand = cmd({
  command: "session",
  describe: "manage sessions",
  builder: (yargs) => {
    return yargs
      .command(SessionNewCommand)
      .command(SessionListCommand)
      .command(SessionPromptCommand)
      .demandCommand(1, "You must specify a subcommand")
  },
  handler: () => {
    // 父命令不需要实现，只用于组织子命令
  },
})

export const SessionNewCommand = cmd({
  command: "new",
  describe: "create a new session",
  builder: (yargs) => {
    return yargs.option("agent", {
      describe: "agent to use",
      type: "string",
    })
  },
  handler: async (args) => {
    // 实现创建会话的逻辑
  },
})
```

**关键点：**

1. **`.demandCommand(1, ...)`：**
   - 要求至少指定一个子命令
   - 如果用户只输入 `opencode session`，会显示错误

2. **父命令的 handler 可以为空：**
   - 父命令只用于组织子命令
   - 实际逻辑在子命令中实现

3. **子命令继承父命令的选项：**
   ```bash
   opencode session new --log-level=DEBUG
   #                    ^^^^^^^^^^^^^^^^
   #                    全局选项在所有命令中可用
   ```

## 2.6 完整的执行流程回顾

现在让我们回到最初的问题：当用户输入 `opencode --version` 时，到底发生了什么？

```
用户输入: opencode --version
    ↓
【第一站：操作系统查找】
操作系统在 PATH 中查找 opencode
找到符号链接: /usr/local/bin/opencode
指向: node_modules/opencode/bin/opencode
    ↓
【第二站：Shebang 解释】
读取第一行: #!/usr/bin/env node
env 在 PATH 中查找 node
node 执行启动器脚本
    ↓
【第三站：启动器处理】
检查 OPENCODE_BIN_PATH 环境变量（无）
检测平台: darwin, 架构: arm64
构造包名: opencode-darwin-arm64
向上查找 node_modules
找到二进制文件: node_modules/opencode-darwin-arm64/bin/opencode
使用 spawnSync 执行，传递参数: ["--version"]
    ↓
【第四站：Yargs 解析】
hideBin 移除 node 和脚本路径
Yargs 解析参数: { version: true }
执行中间件: 初始化日志系统
匹配到 .version() 处理器
    ↓
【第五站：输出结果】
打印版本号: opencode version 1.1.39
退出码: 0
    ↓
用户看到输出
```

**这个看似简单的命令，背后经历了五个关键环节，每个环节都体现了精心的设计。**

## 2.7 设计洞察与最佳实践

通过追踪 `opencode --version` 的完整旅程，我们学到了什么？

### 2.7.1 分层架构的价值

```
启动器层 (bin/opencode)
    ↓ 职责：平台检测、二进制查找
中间件层 (middleware)
    ↓ 职责：初始化、日志、环境变量
命令层 (commands)
    ↓ 职责：业务逻辑
核心层 (core)
    ↓ 职责：实例管理、配置、工具
```

**每一层都有明确的职责边界，这使得：**
- ✅ 代码易于理解和维护
- ✅ 可以独立测试每一层
- ✅ 便于团队协作（不同人负责不同层）

### 2.7.2 权衡与决策

| 决策点 | 选项 A | 选项 B | OpenCode 的选择 | 原因 |
|--------|--------|--------|----------------|------|
| **Shebang 路径** | 硬编码 `/usr/local/bin/node` | 使用 `/usr/bin/env node` | B | 灵活性 > 确定性 |
| **参数解析** | 手动解析 | Yargs 框架 | B | 功能 > 体积 |
| **命令组织** | 单文件 | 模块化 | B | 可维护性 > 简单性 |
| **错误处理** | 统一捕获 | 分类处理 | B | 用户体验 > 实现简单 |

**每个决策都是在特定约束下的最优解。**

### 2.7.3 CLI 设计的黄金法则

1. **遵循 POSIX 约定：**
   - 短选项用单破折号：`-v`
   - 长选项用双破折号：`--version`
   - 参数用等号或空格：`--log-level=DEBUG` 或 `--log-level DEBUG`

2. **提供有用的帮助信息：**
   - 每个命令都应该有 `--help`
   - 描述应该简洁明了
   - 提供使用示例

3. **合理的默认值：**
   - 最常用的选项应该是默认值
   - 用户不应该为常见用例指定大量参数

4. **一致的命名：**
   - 动词-名词结构：`session new`、`mcp list`
   - 避免缩写（除非是行业标准）

5. **友好的错误信息：**
   - 说明问题是什么
   - 给出可能的原因
   - 提供解决方案

## 2.8 本章小结

在本章中，我们从用户输入 `opencode --version` 开始，追踪了命令执行的完整旅程，揭开了 CLI 系统的五个关键环节：

1. **package.json 的 bin 字段**：让普通文件变成全局命令
2. **Shebang 机制**：让文本文件可以像二进制程序一样执行
3. **启动器模式**：实现跨平台的二进制文件分发
4. **Yargs 框架**：提供强大的参数解析和中间件能力
5. **命令系统**：模块化组织 28 个命令

**关键收获：**
- ✅ CLI 不是简单的脚本，而是精心设计的系统
- ✅ 每个环节都有明确的职责和设计考量
- ✅ 好的 CLI 工具应该是跨平台、易用、可维护的
- ✅ 架构设计需要在多个维度上做权衡

**下一章预告：**

现在我们有了 CLI 骨架，但它还不能做任何有用的事情。在第三章中，我们将深入配置系统，学习如何使用 Zod 实现类型安全的配置验证，以及如何通过 Markdown 文件提供用户友好的配置方式。

我们将回答这些问题：
- 如何让用户配置 AI 模型和提供商？
- 如何实现配置的优先级和合并？
- 如何在运行时验证配置的正确性？
- 如何支持多项目的配置隔离？

**实践建议：**

在继续下一章之前，建议你：
1. 克隆 OpenCode 仓库，运行 `opencode --version`
2. 阅读 `bin/opencode` 和 `src/index.ts` 的源代码
3. 尝试添加一个自定义命令（如 `opencode hello`）
4. 使用 `which opencode` 和 `ls -la` 追踪符号链接

**只有真正理解了 CLI 的执行流程，才能在遇到问题时快速定位和解决。**


## 2.5 命令系统:从单一命令到 28 个子命令

### 2.5.1 问题:如何组织大量命令?

OpenCode 有 28 个命令:
- `opencode tui` - 启动 TUI 界面
- `opencode session new` - 创建会话
- `opencode models` - 列出模型
- `opencode mcp list` - 列出 MCP 服务器
- ...

如果把所有逻辑写在一个文件中,会导致:
- ❌ 文件过大(数千行)
- ❌ 难以维护
- ❌ 团队协作困难

### 2.5.2 解决方案:命令模块化

Yargs 支持通过 `.command()` 注册子命令:

```typescript
// src/index.ts
import { ModelsCommand } from "./cli/cmd/models"
import { SessionCommand } from "./cli/cmd/session"

const cli = yargs(hideBin(process.argv))
  .command(ModelsCommand)
  .command(SessionCommand)
  // ... 26 个其他命令
```

每个命令是一个独立的模块,遵循统一的接口:

```typescript
// src/cli/cmd/models.ts
export const ModelsCommand = {
  command: "models [provider]",
  describe: "list all available models",
  builder: (yargs) => {
    return yargs
      .positional("provider", {
        describe: "provider ID to filter models by",
        type: "string",
      })
      .option("verbose", {
        describe: "use more verbose model output",
        type: "boolean",
      })
  },
  handler: async (args) => {
    // 命令实现
  },
}
```


### 2.5.3 命令定义的四个关键部分

让我们深入分析 `ModelsCommand` 的结构:

#### 1. command:命令签名

```typescript
command: "models [provider]"
```

**语法规则:**
- `models`:命令名称(必需)
- `[provider]`:可选位置参数(用方括号包裹)
- `<required>`:必需位置参数(用尖括号包裹)

**示例:**
```bash
opencode models              # provider = undefined
opencode models openai       # provider = "openai"
```

#### 2. describe:命令描述

```typescript
describe: "list all available models"
```

这个描述会出现在:
- `opencode --help` 的命令列表中
- `opencode models --help` 的顶部

#### 3. builder:参数定义

```typescript
builder: (yargs) => {
  return yargs
    .positional("provider", {
      describe: "provider ID to filter models by",
      type: "string",
    })
    .option("verbose", {
      describe: "use more verbose model output",
      type: "boolean",
    })
}
```

**Builder 函数的职责:**
- 定义位置参数(`.positional()`)
- 定义选项参数(`.option()`)
- 设置参数验证规则
- 返回配置后的 yargs 实例

**位置参数 vs 选项参数:**
```bash
opencode models openai --verbose
#               ^^^^^^  ^^^^^^^^
#               位置参数  选项参数
```


### 2.4.4 添加全局选项：日志控制

OpenCode 需要两个全局选项来控制日志行为：

```typescript
const cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .version("1.0.0")
  .help()
  .option("print-logs", {
    describe: "print logs to stderr",
    type: "boolean",
  })
  .option("log-level", {
    describe: "log level",
    type: "string",
    choices: ["DEBUG", "INFO", "WARN", "ERROR"],
  })

cli.parse()
```

**选项定义的关键字段：**
- `describe`：帮助信息中的描述
- `type`：参数类型（`boolean`、`string`、`number`、`array`）
- `choices`：限制可选值（自动验证）

**测试：**

```bash
$ bun run src/index.ts --log-level=DEBUG
# 正常执行

$ bun run src/index.ts --log-level=INVALID
# 错误: Invalid values:
#   Argument: log-level, Given: "INVALID", Choices: "DEBUG", "INFO", "WARN", "ERROR"
```

**Yargs 自动验证了参数值！**

### 2.4.5 中间件：统一的初始化逻辑

现在我们有了参数解析，但每个命令都需要初始化日志系统。如果在每个命令中重复这些逻辑，会导致代码重复和维护困难。

**Yargs 的解决方案：中间件（Middleware）**

```typescript
const cli = yargs(hideBin(process.argv))
  .middleware(async (opts) => {
    // 1. 初始化日志系统
    await Log.init({
      print: opts.printLogs,
      level: opts.logLevel || "INFO",
    })
    
    // 2. 设置环境变量
    process.env.AGENT = "1"
    process.env.OPENCODE = "1"
    
    // 3. 记录命令执行
    Log.Default.info("opencode", {
      version: "1.0.0",
      args: process.argv.slice(2),
    })
  })
  .scriptName("opencode")
  .version("1.0.0")
  .help()
```

**中间件的执行时机：**

```
用户执行: opencode models --provider=openai
    ↓
Yargs 解析参数
    ↓
执行中间件 (opts = { provider: "openai" })
    ↓
执行命令处理器 (ModelsCommand.handler)
```

**中间件的设计优势：**
- ✅ 集中管理初始化逻辑
- ✅ 所有命令自动获得日志能力
- ✅ 可以访问解析后的参数（`opts`）
- ✅ 支持异步操作（`async/await`）

### 2.4.6 OpenCode 的实际中间件实现

让我们分析 OpenCode 的实际代码：

```typescript
.middleware(async (opts) => {
  await Log.init({
    print: process.argv.includes("--print-logs"),
    dev: Installation.isLocal(),
    level: (() => {
      if (opts.logLevel) return opts.logLevel as Log.Level
      if (Installation.isLocal()) return "DEBUG"
      return "INFO"
    })(),
  })

  process.env.AGENT = "1"
  process.env.OPENCODE = "1"

  Log.Default.info("opencode", {
    version: Installation.VERSION,
    args: process.argv.slice(2),
  })
})
```

**设计细节分析：**

1. **日志级别的三级优先级：**
   ```typescript
   level: (() => {
     if (opts.logLevel) return opts.logLevel as Log.Level  // 1. 用户指定
     if (Installation.isLocal()) return "DEBUG"            // 2. 本地开发
     return "INFO"                                          // 3. 生产环境
   })()
   ```
   
   这是一个**立即执行函数表达式（IIFE）**，实现了清晰的优先级逻辑。

2. **为什么检查 `process.argv` 而不是 `opts.printLogs`？**
   ```typescript
   print: process.argv.includes("--print-logs")
   ```
   
   因为中间件执行时，Yargs 可能还没有完全解析所有选项。直接检查 `process.argv` 更可靠。

3. **环境变量的作用：**
   ```typescript
   process.env.AGENT = "1"
   process.env.OPENCODE = "1"
   ```
   
   这些环境变量用于：
   - 标识当前进程是 OpenCode Agent
   - 某些功能会根据这些变量调整行为
   - 子进程可以继承这些变量

**现在你理解了第四个关键机制：Yargs 提供了强大的参数解析和中间件能力，让我们可以优雅地处理复杂的 CLI 逻辑。**

但这引出了下一个问题：如何组织 28 个不同的命令？


### 2.5.4 命令辅助函数:cmd()

你可能注意到,OpenCode 的实际代码使用了 `cmd()` 包装器:

```typescript
export const ModelsCommand = cmd({
  command: "models [provider]",
  describe: "list all available models",
  builder: (yargs) => { /* ... */ },
  handler: async (args) => { /* ... */ },
})
```

让我们看看 `cmd()` 的实现:

```typescript
// src/cli/cmd/cmd.ts
export function cmd<T>(definition: CommandModule<{}, T>): CommandModule<{}, T> {
  return {
    ...definition,
    handler: async (args) => {
      try {
        await definition.handler(args)
      } catch (error) {
        if (error instanceof NamedError) {
          UI.error(error.message)
          process.exit(1)
        }
        throw error
      }
    },
  }
}
```

**`cmd()` 的作用:**

1. **统一的错误处理:**
   - 捕获 `NamedError`(业务错误)
   - 使用 `UI.error()` 显示友好的错误信息
   - 退出码设置为 1

2. **保持类型安全:**
   - 泛型 `<T>` 保留原始参数类型
   - TypeScript 可以正确推断 `args` 的类型

3. **未捕获的错误会继续抛出:**
   - 系统错误(如内存溢出)不会被吞掉
   - 保留完整的堆栈跟踪

**为什么需要 `NamedError`?**

```typescript
// 业务错误:用户友好的错误信息
throw new NamedError("Provider not found: openai")
// 输出: ✖ Provider not found: openai

// 系统错误:保留堆栈跟踪
throw new Error("Unexpected null pointer")
// 输出: 完整的堆栈跟踪
```


### 2.5.5 嵌套命令:session 子命令

OpenCode 有一些命令包含子命令,如 `session`:

```bash
opencode session new          # 创建会话
opencode session list         # 列出会话
opencode session prompt       # 发送消息
```

实现嵌套命令:

```typescript
// src/cli/cmd/session.ts
export const SessionCommand = cmd({
  command: "session",
  describe: "manage sessions",
  builder: (yargs) => {
    return yargs
      .command(SessionNewCommand)
      .command(SessionListCommand)
      .command(SessionPromptCommand)
      .demandCommand(1, "You must specify a subcommand")
  },
  handler: () => {
    // 父命令不需要实现,只用于组织子命令
  },
})

export const SessionNewCommand = cmd({
  command: "new",
  describe: "create a new session",
  builder: (yargs) => {
    return yargs.option("agent", {
      describe: "agent to use",
      type: "string",
    })
  },
  handler: async (args) => {
    // 实现创建会话的逻辑
  },
})
```

**关键点:**

1. **`.demandCommand(1, ...)`**:
   - 要求至少指定一个子命令
   - 如果用户只输入 `opencode session`,会显示错误

2. **父命令的 handler 可以为空:**
   - 父命令只用于组织子命令
   - 实际逻辑在子命令中实现

3. **子命令继承父命令的选项:**
   ```bash
   opencode session new --log-level=DEBUG
   #                    ^^^^^^^^^^^^^^^^
   #                    全局选项在所有命令中可用
   ```


## 2.6 UI 系统:统一的输出格式

### 2.6.1 问题:为什么不直接使用 console.log?

在命令实现中,我们经常看到 `UI.error()` 而不是 `console.error()`:

```typescript
if (!provider) {
  UI.error(`Provider not found: ${args.provider}`)
  return
}
```

**直接使用 console 的问题:**
- ❌ 无法统一样式(颜色、格式)
- ❌ 难以测试(无法捕获输出)
- ❌ 无法适配不同终端环境

### 2.6.2 UI 抽象层的设计

OpenCode 的 `UI` 模块提供了统一的输出接口:

```typescript
// src/cli/ui.ts
export namespace UI {
  export function error(message: string) {
    process.stderr.write(Style.TEXT_ERROR + "✖ " + message + Style.TEXT_NORMAL + "\n")
  }
  
  export function success(message: string) {
    process.stdout.write(Style.TEXT_SUCCESS + "✔ " + message + Style.TEXT_NORMAL + "\n")
  }
  
  export function println(message: string) {
    process.stdout.write(message + "\n")
  }
}
```

**设计优势:**
- ✅ 统一的视觉风格(✖ 和 ✔ 符号)
- ✅ 自动添加 ANSI 颜色代码
- ✅ 可以在测试中 mock
- ✅ 支持不同的输出目标(stdout/stderr)


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


## 2.8 实战练习:实现自定义命令

现在让我们动手实现一个完整的命令,巩固所学知识。

### 练习 1:实现 `opencode hello` 命令

**需求:**
- 命令: `opencode hello [name]`
- 功能: 打印问候语
- 选项: `--uppercase` 将输出转为大写

**步骤 1:创建命令文件**

```typescript
// src/cli/cmd/hello.ts
import { cmd } from "./cmd"
import { UI } from "../ui"

export const HelloCommand = cmd({
  command: "hello [name]",
  describe: "print a greeting message",
  
  builder: (yargs) => {
    return yargs
      .positional("name", {
        describe: "name to greet",
        type: "string",
        default: "World",
      })
      .option("uppercase", {
        describe: "convert output to uppercase",
        type: "boolean",
        default: false,
      })
  },
  
  handler: async (args) => {
    let message = `Hello, ${args.name}!`
    
    if (args.uppercase) {
      message = message.toUpperCase()
    }
    
    UI.success(message)
  },
})
```

**步骤 2:注册命令**

```typescript
// src/index.ts
import { HelloCommand } from "./cli/cmd/hello"

const cli = yargs(hideBin(process.argv))
  // ... 其他配置
  .command(HelloCommand)
  // ... 其他命令
```

**步骤 3:测试**

```bash
bun run src/index.ts hello
# ✔ Hello, World!

bun run src/index.ts hello Alice
# ✔ Hello, Alice!

bun run src/index.ts hello Bob --uppercase
# ✔ HELLO, BOB!
```


### 练习 2:实现带子命令的 `config` 命令

**需求:**
- `opencode config get <key>` - 获取配置值
- `opencode config set <key> <value>` - 设置配置值
- `opencode config list` - 列出所有配置

**实现提示:**

```typescript
// src/cli/cmd/config.ts
export const ConfigCommand = cmd({
  command: "config",
  describe: "manage configuration",
  builder: (yargs) => {
    return yargs
      .command(ConfigGetCommand)
      .command(ConfigSetCommand)
      .command(ConfigListCommand)
      .demandCommand(1, "You must specify a subcommand")
  },
  handler: () => {},
})

export const ConfigGetCommand = cmd({
  command: "get <key>",
  describe: "get a configuration value",
  builder: (yargs) => {
    return yargs.positional("key", {
      describe: "configuration key",
      type: "string",
    })
  },
  handler: async (args) => {
    // TODO: 实现获取配置的逻辑
    UI.println(`${args.key} = ...`)
  },
})

// TODO: 实现 ConfigSetCommand 和 ConfigListCommand
```

## 2.9 技术难点与最佳实践

### 2.9.1 参数验证

Yargs 提供了强大的验证机制:

```typescript
builder: (yargs) => {
  return yargs
    .option("port", {
      type: "number",
      default: 3000,
    })
    .check((args) => {
      if (args.port < 1024 || args.port > 65535) {
        throw new Error("Port must be between 1024 and 65535")
      }
      return true
    })
}
```


### 2.9.2 异步命令处理

所有命令处理器都应该是 `async`:

```typescript
handler: async (args) => {
  // ✅ 正确:使用 await
  const data = await fetchData()
  
  // ❌ 错误:忘记 await
  const data = fetchData()  // 返回 Promise,不是实际数据
}
```

### 2.9.3 错误处理模式

```typescript
handler: async (args) => {
  try {
    const result = await riskyOperation()
    UI.success("Operation completed")
  } catch (error) {
    if (error instanceof NamedError) {
      // 业务错误:友好提示
      UI.error(error.message)
      process.exit(1)
    }
    // 系统错误:抛出以显示堆栈
    throw error
  }
}
```

### 2.9.4 进度显示

对于长时间运行的命令,应该提供进度反馈:

```typescript
import { UI } from "../ui"

handler: async (args) => {
  UI.println("Downloading models...")
  
  for (const model of models) {
    UI.println(`  - ${model.name}`)
    await downloadModel(model)
  }
  
  UI.success("All models downloaded")
}
```


## 2.10 企业级价值与工程洞察

### 2.10.1 为什么 CLI 优先?

OpenCode 选择 CLI 作为主要交互方式,而不是 GUI,原因包括:

1. **自动化友好:**
   ```bash
   # 可以轻松集成到脚本中
   opencode session new --agent=build | tee session.log
   ```

2. **远程访问:**
   ```bash
   # SSH 到服务器后直接使用
   ssh user@server
   opencode tui
   ```

3. **版本控制:**
   ```bash
   # 命令可以写入文档和脚本
   git commit -m "Add opencode integration"
   ```

4. **低资源消耗:**
   - 不需要图形界面
   - 适合容器和 CI/CD 环境

### 2.10.2 CLI 设计的黄金法则

1. **遵循 POSIX 约定:**
   - 短选项用单破折号: `-v`
   - 长选项用双破折号: `--version`
   - 参数用等号或空格: `--log-level=DEBUG` 或 `--log-level DEBUG`

2. **提供有用的帮助信息:**
   - 每个命令都应该有 `--help`
   - 描述应该简洁明了
   - 提供使用示例

3. **合理的默认值:**
   - 最常用的选项应该是默认值
   - 用户不应该为常见用例指定大量参数

4. **一致的命名:**
   - 动词-名词结构: `session new`, `mcp list`
   - 避免缩写(除非是行业标准)


### 2.10.3 从 OpenCode 学到的架构智慧

1. **启动器模式:**
   - 将平台检测逻辑与业务逻辑分离
   - 使用 Node.js 脚本作为跨平台启动器
   - 真正的二进制文件按平台分发

2. **中间件模式:**
   - 集中管理初始化逻辑
   - 避免在每个命令中重复代码
   - 支持横切关注点(日志、认证等)

3. **命令模块化:**
   - 每个命令一个文件
   - 统一的命令接口
   - 便于团队协作和维护

4. **类型安全:**
   - 使用 TypeScript 定义参数类型
   - Yargs 自动推断参数类型
   - 编译时捕获错误

## 2.11 本章小结

在本章中,我们从最简单的 "Hello World" 开始,逐步构建了一个完整的 CLI 系统:

1. **可执行文件机制:**
   - 理解 shebang 和 `package.json` 的 `bin` 字段
   - 实现跨平台的启动器
   - 掌握二进制文件查找算法

2. **Yargs 框架:**
   - 命令定义的四个部分(command、describe、builder、handler)
   - 位置参数和选项参数
   - 嵌套命令和子命令

3. **中间件机制:**
   - 统一的初始化逻辑
   - 日志系统集成
   - 环境变量管理

4. **命令系统:**
   - 模块化的命令组织
   - 统一的错误处理
   - UI 抽象层

**关键收获:**
- ✅ CLI 是 AI 编码助手的理想交互方式
- ✅ Yargs 提供了强大的命令行解析能力
- ✅ 模块化设计使得 28 个命令易于维护
- ✅ 类型安全贯穿整个 CLI 系统

**下一章预告:**

在第三章中,我们将深入配置系统,学习如何使用 Zod 实现类型安全的配置验证,以及如何通过 Markdown 文件提供用户友好的配置方式。

