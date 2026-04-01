# 研究文档：src/utils/renderOptions.ts

## 场景与职责

`renderOptions.ts` 是 Claude Code TUI（基于 Ink）渲染层的**基础配置工厂**。Ink 的 `render()` 函数需要传入 `RenderOptions`，而当标准输入（stdin）被管道化（piped）时，Ink 无法直接读取键盘输入，导致交互组件（对话框、选择器等）失效。该模块解决的核心问题是：

> **在 stdin 为 pipe 的非 TTY 场景下，尝试打开 `/dev/tty` 作为替代输入源，并将该替代流注入到 Ink 的 `RenderOptions` 中。**

同时，它还要规避若干已知陷阱：CI 环境、MCP 模式、Windows 平台、以及 Bun 编译二进制对 `isTTY` 检测的兼容性问题。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getStdinOverride()` | 当 `process.stdin` 不是 TTY 时，尝试打开 `/dev/tty` 并返回一个 `ReadStream`，使 Ink 恢复交互能力。 |
| `getBaseRenderOptions()` | 为所有 `render()` 调用统一生成基础 `RenderOptions`，确保 `exitOnCtrlC` 与 `stdin` 覆盖一致。 |
| 缓存机制 | `cachedStdinOverride` 在进程生命周期内只计算一次，避免重复打开文件描述符。 |

---

## 具体技术实现

### 1. `getStdinOverride()` 的决策树

```ts
function getStdinOverride(): ReadStream | undefined {
  // 1. 缓存命中
  if (cachedStdinOverride !== null) return cachedStdinOverride

  // 2. stdin 已经是 TTY，无需覆盖
  if (process.stdin.isTTY) {
    cachedStdinOverride = undefined
    return undefined
  }

  // 3. CI 环境跳过（避免在 GitHub Actions 等环境中尝试打开 /dev/tty 报错）
  if (isEnvTruthy(process.env.CI)) {
    cachedStdinOverride = undefined
    return undefined
  }

  // 4. MCP 模式跳过（输入劫持会破坏 MCP 的 stdio 通信）
  if (process.argv.includes('mcp')) {
    cachedStdinOverride = undefined
    return undefined
  }

  // 5. Windows 没有 /dev/tty
  if (process.platform === 'win32') {
    cachedStdinOverride = undefined
    return undefined
  }

  // 6. 尝试打开 /dev/tty
  try {
    const ttyFd = openSync('/dev/tty', 'r')
    const ttyStream = new ReadStream(ttyFd)
    // Bun 编译二进制可能检测不到 isTTY，强制标记为 true
    ttyStream.isTTY = true
    cachedStdinOverride = ttyStream
    return cachedStdinOverride
  } catch (err) {
    logError(err as Error)
    cachedStdinOverride = undefined
    return undefined
  }
}
```

### 2. `getBaseRenderOptions()`

```ts
export function getBaseRenderOptions(exitOnCtrlC: boolean = false): RenderOptions {
  const stdin = getStdinOverride()
  const options: RenderOptions = { exitOnCtrlC }
  if (stdin) {
    options.stdin = stdin
  }
  return options
}
```

- `exitOnCtrlC` 默认 `false`，因为对话框等组件通常不希望按 Ctrl+C 直接退出进程。
- 所有调用 `render()` 的地方都应通过此函数获取基础选项，保证行为一致。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/renderOptions.ts:15-60` | `getStdinOverride()` 完整实现。 |
| `src/utils/renderOptions.ts:68-77` | `getBaseRenderOptions()` 公共 API。 |
| `src/ink.ts:18-23` | `render()` 包装函数，接收 `RenderOptions`。 |
| `src/main.tsx` | 多处通过 `getBaseRenderOptions()` 渲染对话框（如 `InvalidConfigDialog`）。 |
| `src/interactiveHelpers.tsx` | TUI 交互辅助函数，调用 `render()`。 |
| `src/services/remoteManagedSettings/securityCheck.tsx` | 安全提示弹窗渲染。 |
| `src/components/InvalidConfigDialog.tsx` | 配置错误弹窗。 |

---

## 依赖与外部交互

- **Node.js 内置模块**：
  - `fs.openSync` → 打开 `/dev/tty`
  - `tty.ReadStream` → 包装文件描述符为 TTY 可读流
- **内部依赖**：
  - `./envUtils.js`：`isEnvTruthy`
  - `./log.js`：`logError`
  - `../ink.js`：`RenderOptions` 类型
- **调用方**：
  - `src/main.tsx`
  - `src/interactiveHelpers.tsx`
  - `src/services/remoteManagedSettings/securityCheck.tsx`
  - `src/components/InvalidConfigDialog.tsx`
  - 以及所有需要 `render()` 的 TUI 组件

---

## 风险、边界与改进建议

### 风险与边界

1. **MCP 模式检测过于宽松**：`process.argv.includes('mcp')` 可能误伤包含 `mcp` 子串的无关参数（如 `--mcp-config`）。虽然当前 CLI 参数结构下 `mcp` 通常作为子命令出现，但严格来说应匹配 `argv[2] === 'mcp'` 或解析后的子命令。

2. **Bun 编译二进制兼容**：强制设置 `ttyStream.isTTY = true` 是为了绕过 Bun 的 `ReadStream` 检测缺陷。若未来 Bun 修复该问题，这段代码是安全的 no-op；若 Bun 行为进一步变化，可能需要额外适配。

3. **文件描述符泄漏**：`openSync('/dev/tty', 'r')` 打开的文件描述符由 `ReadStream` 管理。Node.js 的 `ReadStream` 在进程退出时会自动关闭，但若在长时间运行的守护进程中反复创建/销毁 Ink 实例（实际项目中不会），FD 可能累积。当前缓存机制已避免重复打开。

4. **Windows 完全放弃**：Windows 没有 `/dev/tty`，因此 stdin 为 pipe 时（如 `echo "prompt" | claude -p` 以外的场景）无法恢复交互。这是平台限制，设计上接受。

5. **CI 检测的 false negative**：某些 CI 环境（如自托管 runner）可能不设置 `CI=true`，此时尝试打开 `/dev/tty` 可能失败并产生 `logError` 噪音。

### 改进建议

1. **精确匹配 MCP 子命令**：将 `process.argv.includes('mcp')` 改为 `process.argv.slice(2)[0] === 'mcp' && process.argv.slice(2)[1] === 'serve'` 或利用 commander 解析后的状态，避免误伤。

2. **增加 `isatty` 二次确认**：在强制设置 `isTTY = true` 前，可调用 `fs.stat('/dev/tty')` 确认设备确实存在，虽然 `openSync` 已隐含此检查，但可提供更清晰的错误日志。

3. **暴露清理接口**：若未来需要支持 Ink 实例的反复创建/销毁，可考虑在模块中暴露 `closeStdinOverride()`，在 `render()` 结束或进程退出时主动关闭 `ReadStream`。

4. **文档化使用契约**：在 `ink.ts` 的 `render()` 注释中强调“所有调用方应优先使用 `getBaseRenderOptions()`”，目前已有部分调用方直接传 `exitOnCtrlC`，未显式处理 stdin 覆盖，虽然可能不影响（因为那些路径 stdin 通常是 TTY），但统一更为安全。
