# ShellCommand.ts 深度研究文档

> 文件路径：`src/utils/ShellCommand.ts`（约 465 行）  
> 研究日期：2026-04-01

---

## 1. 场景与职责

`ShellCommand.ts` 是 Claude Code CLI 中**子进程生命周期管理的核心封装层**。它将 Node.js 的 `ChildProcess` 包装为一个结构化的 `ShellCommand` 接口，向上层屏蔽了进程状态管理、超时控制、后台化（backgrounding）、信号处理、输出收集与资源回收等复杂细节。

该模块主要服务于以下两类执行场景：

- **Bash 命令执行（file mode）**：`stdout` 与 `stderr` 直接重定向到同一个文件描述符（fd），不经过 JavaScript 层的流处理。适用于常规的 shell 命令执行，依赖 `TaskOutput` 的文件尾轮询（tail polling）来获取进度。
- **Hook 执行（pipe mode）**：`stdout` 与 `stderr` 以 pipe 模式创建，通过 `StreamWrapper` 实时桥接到 `TaskOutput` 的内存缓冲区。适用于需要实时检测输出（如进度条、实时日志）的场景。

上层调用方主要包括：
- `src/utils/Shell.ts`：通过 `wrapSpawn()` 创建 `ShellCommand` 的主要入口；
- `src/tasks/LocalShellTask/LocalShellTask.tsx` 及其 `guards.ts`：负责任务的 UI 渲染与后台化决策；
- 各类内部工具与命令：凡涉及 shell 执行的地方，最终都会落到该模块。

---

## 2. 功能点目的

### 2.1 统一子进程抽象（`ShellCommand` 接口）

将底层 `ChildProcess` 封装为包含以下能力的对象：

| 成员 | 作用 |
|------|------|
| `background(taskId)` | 将前台运行的命令转为后台任务，停止超时计时，启动大小看门狗或溢写磁盘 |
| `result: Promise<ExecResult>` | 异步获取执行结果，包含 stdout、stderr、退出码、是否被中断、后台任务 ID 等 |
| `kill()` | 立即发送 `SIGKILL` 终止进程树 |
| `status` | 当前状态：`'running' \| 'backgrounded' \| 'completed' \| 'killed'` |
| `cleanup()` | 清理事件监听器、释放引用，防止内存泄漏 |
| `onTimeout?` | 可选回调注册点，用于 assistant 模式下的自动后台化 |
| `taskOutput` | 关联的 `TaskOutput` 实例，统一持有所有输出数据与进度信息 |

### 2.2 超时与自动后台化（Auto-Backgrounding）

默认超时时间为 **30 分钟**（`DEFAULT_TIMEOUT = 30 * 60 * 1000`，定义在 `Shell.ts` 第 44 行）。当命令运行时间超过阈值时：

- 若 `shouldAutoBackground === false`：直接调用 `tree-kill` 发送 `SIGTERM`（退出码 143），并在 `ExecResult.stderr` 中附加超时提示；
- 若 `shouldAutoBackground === true`（通常由 assistant 模式启用）：触发 `onTimeout` 回调，由上层决策是否调用 `background(taskId)` 将任务转为后台。这样模型仍可在后续看到部分输出，而不是直接杀死进程。

### 2.3 输出大小限制与看门狗（Size Watchdog）

为了防止后台任务因陷入死循环而无限追加输出、撑爆磁盘，模块在任务被后台化后启动一个**大小看门狗**（`#startSizeWatchdog`），每 **5 秒**轮询一次输出文件的 `size`：

- 上限为 `MAX_TASK_OUTPUT_BYTES = 5GB`（定义在 `src/utils/task/diskOutput.ts` 第 30 行）；
- 一旦超过，立即发送 `SIGKILL`（退出码 137），并在 `stderr` 中标记为被大小限制杀死。

该机制直接源于一次生产事故：注释中明确提到了 **"768GB incident"**（`ShellCommand.ts` 第 357 行），即某个后台任务的输出文件在未受限制的情况下膨胀到了 768GB。

### 2.4 预生成错误与中断处理

对于在 `spawn` 之前就失败的情况（如工作目录被删除），模块提供了两个工厂函数：

- `createAbortedCommand()`：返回一个已处于 `killed` 状态的 `AbortedShellCommand`，用于 `abortSignal` 已在 spawn 前触发的情况；
- `createFailedCommand(preSpawnError)`：返回一个已 `completed` 的纯对象，用于 spawn 前检测到不可恢复错误（如 cwd 不存在）。

---

## 3. 具体技术实现

### 3.1 StreamWrapper：流桥接与内存管理

`StreamWrapper`（第 66–104 行）是一个轻量级的私有类，负责将 `ChildProcess` 的 `Readable` 流桥接到 `TaskOutput`：

```typescript
class StreamWrapper {
  #stream: Readable | null
  #isCleanedUp = false
  #taskOutput: TaskOutput | null
  #isStderr: boolean
  #onData = this.#dataHandler.bind(this)

  constructor(stream: Readable, taskOutput: TaskOutput, isStderr: boolean) {
    this.#stream = stream
    this.#taskOutput = taskOutput
    this.#isStderr = isStderr
    stream.setEncoding('utf-8')   // 避免重复的 Buffer.toString()
    stream.on('data', this.#onData)
  }

  cleanup(): void {
    if (this.#isCleanedUp) return
    this.#isCleanedUp = true
    this.#stream!.removeListener('data', this.#onData)
    this.#stream = null           // 释放引用，允许 GC
    this.#taskOutput = null
    this.#onData = () => {}
  }
}
```

**关键设计点**：
- `setEncoding('utf-8')` 在流级别设置编码，避免每个 chunk 都进行 `Buffer.toString()`；
- `cleanup()` 不仅移除事件监听器，还将 `#stream` 和 `#taskOutput` 显式置为 `null`，打破闭包引用链，使 `StringDecoder` 和 `TaskOutput` 能被独立垃圾回收。

### 3.2 使用 `'exit'` 而非 `'close'` 事件

在 `#createResultPromise()` 方法（第 263–289 行）中，进程退出监听绑定的是 `'exit'` 事件，而非 Node.js 文档中更常见的 `'close'`：

```typescript
// 第 269–272 行
// Use 'exit' not 'close': 'close' waits for stdio to close, which includes
// grandchild processes that inherit file descriptors (e.g. `sleep 30 &`).
// 'exit' fires when the shell itself exits, returning control immediately.
this.#childProcess.once('exit', this.#exitHandler.bind(this))
this.#childProcess.once('error', this.#errorHandler.bind(this))
```

**原因分析**：
- `'close'` 会等待所有 stdio 流完全关闭后才触发；
- 在 shell 命令中，后台运行的孙子进程（如 `sleep 30 &`）会继承 shell 的文件描述符，导致即使 shell 本身已退出，`stdout`/`stderr` fd 仍被后台进程持有；
- 使用 `'close'` 将导致 `ShellCommand.result` 长时间挂起，无法及时返回控制权；
- `'exit'` 在 shell 进程本身终止时立即触发，符合 CLI 工具"命令执行完毕即返回"的语义。

### 3.3 自动后台化超时机制

`ShellCommandImpl` 的构造函数接收 `shouldAutoBackground` 参数（第 153 行）。当该标志为 `true` 时：

1. 暴露 `onTimeout` 方法（第 174–177 行），允许上层注册回调；
2. 在 `#createResultPromise()` 中设置 `setTimeout`（第 275–279 行），超时后调用静态方法 `#handleTimeout`（第 135–141 行）：

```typescript
static #handleTimeout(self: ShellCommandImpl): void {
  if (self.#shouldAutoBackground && self.#onTimeoutCallback) {
    self.#onTimeoutCallback(self.background.bind(self))
  } else {
    self.#doKill(SIGTERM)
  }
}
```

3. 上层（如 `LocalShellTask`）在 `onTimeout` 回调中决策：若用户或模型同意后台化，则调用传入的 `backgroundFn(taskId)`；否则放任超时，进程会被 `SIGTERM` 终止。

### 3.4 AbortSignal 的 'interrupt' 特例

`#abortHandler()`（第 186–193 行）处理 `AbortSignal` 的 `abort` 事件：

```typescript
#abortHandler(): void {
  // On 'interrupt' (user submitted a new message), don't kill — let the
  // caller background the process so the model can see partial output.
  if (this.#abortSignal.reason === 'interrupt') {
    return
  }
  this.kill()
}
```

**设计意图**：
- 当用户发送新消息（`reason === 'interrupt'`）时，系统**不立即杀死**正在运行的 shell 命令；
- 这是为了让调用方有机会将命令后台化，从而使模型在后续对话中能够读取到已产生的部分输出；
- 对于其他 abort 原因（如会话关闭、工具取消），则直接调用 `kill()` 发送 `SIGKILL`。

### 3.5 大小看门狗实现

`#startSizeWatchdog()`（第 239–261 行）在 `background()` 被调用后启动：

```typescript
#startSizeWatchdog(): void {
  this.#sizeWatchdog = setInterval(() => {
    void stat(this.taskOutput.path).then(
      s => {
        if (
          s.size > this.#maxOutputBytes &&
          this.#status === 'backgrounded' &&
          this.#sizeWatchdog !== null
        ) {
          this.#killedForSize = true
          this.#clearSizeWatchdog()
          this.#doKill(SIGKILL)
        }
      },
      () => { /* ENOENT 或已删除，跳过本次 */ }
    )
  }, SIZE_WATCHDOG_INTERVAL_MS)  // 5000ms
  this.#sizeWatchdog.unref()
}
```

**关键细节**：
- 仅对 **file mode**（`taskOutput.stdoutToFile === true`）启动看门狗，因为 pipe mode 的数据经过 JS 层，已由 `DiskTaskOutput` 的 `#bytesWritten` 粗粒度限制；
- 看门狗使用 `unref()`，避免阻止 Node.js 进程自然退出；
- `stat` 是异步的，在回调中二次检查 `#sizeWatchdog !== null`，防止在 `stat` 飞行途中进程已自然退出而导致误杀；
- 上限 `MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024`（5GB），显示文本为 `'5GB'`。

### 3.6 结果构建与输出文件优化

`#handleExit()`（第 291–335 行）在进程退出后构建 `ExecResult`：

```typescript
async #handleExit(code: number): Promise<void> {
  this.#cleanupListeners()
  if (this.#status === 'running' || this.#status === 'backgrounded') {
    this.#status = 'completed'
  }

  const stdout = await this.taskOutput.getStdout()
  const result: ExecResult = {
    code,
    stdout,
    stderr: this.taskOutput.getStderr(),
    interrupted: code === SIGKILL,
    backgroundTaskId: this.#backgroundTaskId,
  }

  if (this.taskOutput.stdoutToFile && !this.#backgroundTaskId) {
    if (this.taskOutput.outputFileRedundant) {
      void this.taskOutput.deleteOutputFile()   // 小文件直接内联，删除冗余文件
    } else {
      result.outputFilePath = this.taskOutput.path
      result.outputFileSize = this.taskOutput.outputFileSize
      result.outputTaskId = this.taskOutput.taskId
    }
  }
  // ... 附加 killedForSize / SIGTERM 的 stderr 提示
}
```

**输出文件策略**：
- 若输出文件内容较小（能被 `getMaxOutputLength()` 一次性读完），`outputFileRedundant` 为 `true`，文件会被删除，结果完全内联在 `result.stdout` 中；
- 若输出文件过大，则在 `ExecResult` 中保留 `outputFilePath`、`outputFileSize`、`outputTaskId`，下游可通过 `<persisted-output>` 机制按需读取。

### 3.7 内存泄漏防护模式

`ShellCommandImpl` 在 `cleanup()`（第 368–381 行）中实施了多层引用释放：

```typescript
cleanup(): void {
  this.#stdoutWrapper?.cleanup()
  this.#stderrWrapper?.cleanup()
  this.taskOutput.clear()
  this.#cleanupListeners()
  // Release references to allow GC of ChildProcess internals and AbortController chain
  this.#childProcess = null!
  this.#abortSignal = null!
  this.#onTimeoutCallback = undefined
}
```

**防护要点**：
1. `StreamWrapper.cleanup()` 移除 `'data'` 监听器并释放流与 `TaskOutput` 引用；
2. `taskOutput.clear()` 清空内存缓冲区、取消磁盘写入、停止轮询、从全局注册表中注销；
3. `#cleanupListeners()` 清除超时定时器、大小看门狗、`AbortSignal` 的 `abort` 事件监听器；
4. 最后将 `#childProcess` 和 `#abortSignal` 强制置为 `null`（使用 `!` 类型断言绕过 TS 检查），确保 `ChildProcess` 内部对象和 `AbortController` 链能被垃圾回收。

**特别注意**（第 372–375 行注释）：`#cleanupListeners()` 必须在 `#childProcess` 和 `#abortSignal` 置空**之前**调用。否则，如果调用序列是 `kill()` → `cleanup()`，`kill()` 会排队一个微任务让 `#handleExit` 运行，而 `cleanup()` 若先置空了 `#abortSignal`，随后 `#handleExit` 调用 `#cleanupListeners()` 时会在 `null` 引用上调用 `removeEventListener`，导致崩溃。

---

## 4. 关键代码路径与文件引用

### 4.1 核心类型与类

| 名称 | 位置 | 说明 |
|------|------|------|
| `ExecResult` | `ShellCommand.ts` 第 13–30 行 | 执行结果的数据结构 |
| `ShellCommand` | `ShellCommand.ts` 第 32–47 行 | 对外接口定义 |
| `StreamWrapper` | `ShellCommand.ts` 第 66–104 行 | 流桥接器 |
| `ShellCommandImpl` | `ShellCommand.ts` 第 114–382 行 | 主实现类 |
| `AbortedShellCommand` | `ShellCommand.ts` 第 408–435 行 | 预中断的静态实现 |

### 4.2 工厂函数

| 函数 | 位置 | 用途 |
|------|------|------|
| `wrapSpawn()` | `ShellCommand.ts` 第 387–403 行 | 从 `ChildProcess` 创建 `ShellCommandImpl` |
| `createAbortedCommand()` | `ShellCommand.ts` 第 437–445 行 | 生成预中断命令对象 |
| `createFailedCommand()` | `ShellCommand.ts` 第 447–465 行 | 生成预失败命令对象 |

### 4.3 `ShellCommandImpl` 关键方法索引

| 方法 | 行号 | 职责 |
|------|------|------|
| `constructor` | 148–180 | 初始化状态、包装流、注册超时 |
| `#createResultPromise` | 263–289 | 绑定 `exit`/`error`/abort 事件，创建结果 Promise |
| `#handleExit` | 291–335 | 进程退出后组装 `ExecResult` |
| `#abortHandler` | 186–193 | 处理中断信号，特殊处理 `'interrupt'` |
| `#exitHandler` | 195–203 | 将 `exit`/`signal` 映射为数字退出码 |
| `#doKill` | 337–343 | 调用 `tree-kill` 发送 `SIGKILL` |
| `kill` | 345–347 | 公开终止接口 |
| `background` | 349–366 | 转为后台任务，启动看门狗或溢写磁盘 |
| `#startSizeWatchdog` | 239–261 | 启动 5 秒轮询的文件大小看门狗 |
| `#cleanupListeners` | 218–230 | 清理定时器与 abort 监听器 |
| `cleanup` | 368–381 | 全面资源释放与防泄漏处理 |

### 4.4 调用链示例

```
Shell.ts:exec()
  └── 创建 ChildProcess (spawn)
  └── wrapSpawn() ──> new ShellCommandImpl()
        ├── 创建 StreamWrapper (pipe mode 时)
        ├── #createResultPromise() 绑定 'exit' / 'error' / timeout / abort
        └── result Promise 等待 #handleExit()

LocalShellTask.tsx
  └── shellCommand.onTimeout(callback)
        └── 超时后 callback(backgroundFn)
              └── background(taskId) ──> #startSizeWatchdog()
```

---

## 5. 依赖与外部交互

### 5.1 内部模块依赖

| 模块 | 路径 | 交互内容 |
|------|------|----------|
| `TaskOutput` | `src/utils/task/TaskOutput.ts` | 统一持有 stdout/stderr 数据、进度回调、文件读写、内存/磁盘溢写策略 |
| `diskOutput` | `src/utils/task/diskOutput.ts` | 提供 `MAX_TASK_OUTPUT_BYTES` (5GB)、`MAX_TASK_OUTPUT_BYTES_DISPLAY` ('5GB')、磁盘写入队列 `DiskTaskOutput` |
| `format` | `src/utils/format.ts` | 使用 `formatDuration()` 格式化超时时间 |
| `Task` | `src/Task.js` | 使用 `generateTaskId('local_bash')` 生成任务 ID |

### 5.2 外部 npm 依赖

- **`tree-kill`**：跨平台的进程树终止工具。`#doKill()` 中通过 `treeKill(this.#childProcess.pid, 'SIGKILL')` 确保不仅杀死直接子进程，还递归终止其所有后代进程，避免孤儿进程残留。

### 5.3 与 Node.js 内置模块的交互

- `child_process`：`ChildProcess` 类型；
- `stream`：`Readable` 类型；
- `fs/promises`：`stat()` 用于大小看门狗读取文件尺寸。

### 5.4 与上层调用方的交互

- **`src/utils/Shell.ts`**：唯一通过 `wrapSpawn()` 创建 `ShellCommandImpl` 的生产代码路径。负责 `spawn` 参数构建、cwd 恢复、沙箱包装、fd 关闭等；
- **`src/tasks/LocalShellTask/LocalShellTask.tsx`**：在 assistant 模式下，注册 `onTimeout` 回调，决定超时后是自动后台化还是终止；
- **`src/tasks/LocalShellTask/guards.ts`**：检查 `shellCommand.status` 以判断任务是否可被后台化或已结束。

---

## 6. 风险、边界与改进建议

### 6.1 已识别的风险与边界

#### 6.1.1 `'exit'` 与 `'close'` 的语义差异

虽然使用 `'exit'` 避免了后台孙子进程导致的挂起，但它也意味着：如果 shell 退出后仍有大量输出缓冲在管道中未读取，这些输出可能丢失。不过在该模块的设计中：

- **file mode** 的输出直接写入 fd，不经过 JS 管道，因此不存在此问题；
- **pipe mode** 的数据通过 `StreamWrapper` 实时读取，shell 退出时流通常已结束。

#### 6.1.2 大小看门狗的 5 秒窗口期

看门狗每 5 秒检查一次文件大小。对于一个写入速度极快的进程（如 `yes > file`），理论上可能在两次检查之间写入数十 GB 数据。虽然 5GB 的上限和 5 秒间隔在大多数场景下足够，但在极端 I/O 密集型环境中仍存在短暂超出的风险。

#### 6.1.3 `tree-kill` 的跨平台可靠性

`tree-kill` 在 Windows 上依赖 `taskkill /F /T`，在 Unix 上依赖 `ps` 命令解析进程树。在某些受限环境（如极简容器）中，`ps` 可能不可用，导致 `tree-kill` 失效。不过该风险已被项目其他部分接受。

#### 6.1.4 `SIGTERM` 与退出码 144 的自定义映射

`#exitHandler()`（第 195–203 行）对 `signal === 'SIGTERM'` 硬编码返回 144：

```typescript
const exitCode =
  code !== null && code !== undefined
    ? code
    : signal === 'SIGTERM'
      ? 144
      : 1
```

144 并非标准 Unix 退出码（标准 SIGTERM 是 128 + 15 = 143）。此处使用 144 可能是项目内部约定，但对外部工具或新开发者而言容易造成困惑。

#### 6.1.5 `prependStderr` 的格式覆盖

当命令因超时被杀或因大小限制被杀时，模块通过 `prependStderr()` 在原有 `stderr` **前面**插入提示信息。如果原有 `stderr` 为空，则只保留前缀；如果已有内容，则前缀与旧内容以空格分隔。这种前置方式可能导致模型在解析 stderr 时，需要额外处理前缀字符串。

### 6.2 改进建议

#### 6.2.1 考虑为看门狗引入更细粒度的检查

对于高速输出场景，可以考虑：
- 将看门狗间隔从 5 秒缩短到 1–2 秒；或
- 在 `DiskTaskOutput` 层引入更严格的逐块写入上限（目前仅按 `content.length` 粗算），虽然已有 5GB 上限，但 file mode 绕过 JS 层，只能依赖文件系统层面的 `fstat`。

#### 6.2.2 统一退出码语义文档

建议将 144、137、143 等特殊退出码的语义集中记录在 `AGENTS.md` 或代码注释中，避免维护者在调试时误解。特别是 144 与标准 SIGTERM 退出码 143 的差异应予以说明。

#### 6.2.3 增强 `cleanup()` 的幂等性保障

虽然 `StreamWrapper.cleanup()` 已有 `#isCleanedUp` 标志，`ShellCommandImpl.cleanup()` 本身没有重复调用保护。建议增加一个 `#isCleanedUp` 私有字段，防止上层在 `result` 解析前后重复调用 `cleanup()` 时出现竞态条件。

#### 6.2.4 评估 `'close'` 事件的混合策略

对于 pipe mode，可以考虑同时监听 `'exit'` 和 `'close'`：以 `'exit'` 触发结果解析保证响应速度，以 `'close'` 作为后备确保所有流数据已被消费。不过需要仔细评估是否会增加复杂度以及是否真的能解决当前不存在的问题。

#### 6.2.5 暴露更丰富的后台化原因

当前 `ExecResult` 中通过 `assistantAutoBackgrounded` 布尔值标记自动后台化。可以考虑将其扩展为枚举类型（如 `'user_backgrounded' \| 'assistant_auto_backgrounded' \| 'timeout_killed' \| 'size_killed'`），使下游消费方更容易进行分支处理。

---

## 7. 总结

`ShellCommand.ts` 是 Claude Code CLI 执行子进程的关键基础设施。它在约 465 行代码中精巧地处理了：

- **两种输出模式**（file mode / pipe mode）的适配；
- **超时与自动后台化**的协同；
- **AbortSignal 的差异化处理**（`'interrupt'` 不杀进程）；
- **5GB 大小看门狗**（源于 768GB 生产事故）的防护；
- **`'exit'` 优于 `'close'`** 的进程退出语义选择；
- **多层引用释放**以防止内存泄漏。

理解该模块的实现细节，对于排查命令超时、后台任务异常、输出文件膨胀以及进程泄漏等问题至关重要。
