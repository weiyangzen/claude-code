# ShellTimeDisplay.tsx 研究文档

## 场景与职责

`ShellTimeDisplay.tsx` 是一个纯展示型组件，专门用于在终端 UI 中渲染 shell 命令的**时间信息**：

- 已运行时长（`elapsedTimeSeconds`）
- 超时限制（`timeoutMs`）

它本身不维护任何计时器状态，仅接收外部传入的数值并格式化为人类可读的字符串。该组件被 `ShellProgressMessage`（运行中进度）以及 `BashToolResultMessage`、`PowerShellTool/UI`（结果页）调用，用于在命令执行的不同阶段向用户展示时间上下文。

## 功能点目的

### 1. 运行时长展示
将秒数转换为 `formatDuration(elapsedTimeSeconds * 1000)` 的字符串，例如 `5s`、`1m 30s`、`2h 15m 10s`。

### 2. 超时限制展示
将毫秒数转换为 `formatDuration(timeoutMs, { hideTrailingZeros: true })`，例如 `30s`、`5m`、`1h`。

### 3. 组合展示
当同时存在运行时长和超时时，以 `(elapsed · timeout timeout)` 的格式一并展示，例如 `(5s · timeout 30s)`；若只有超时则展示 `(timeout 30s)`；若只有运行时长则展示 `(5s)`。

### 4. 空值回退
当 `elapsedTimeSeconds` 与 `timeoutMs` 均为空时，组件直接返回 `null`，避免渲染无意义的空节点。

## 具体技术实现

### 关键流程

```
接收 props: elapsedTimeSeconds?, timeoutMs?
    │
    ▼
两者皆无？ → return null
    │
    ▼
timeout = timeoutMs ? formatDuration(timeoutMs, { hideTrailingZeros: true }) : undefined
    │
    ├─ elapsedTimeSeconds === undefined
    │      → return <Text dimColor>(timeout {timeout})</Text>
    │
    ▼
elapsed = formatDuration(elapsedTimeSeconds * 1000)
    │
    ├─ timeout 存在
    │      → return <Text dimColor>({elapsed} · timeout {timeout})</Text>
    │
    └─ timeout 不存在
           → return <Text dimColor>({elapsed})</Text>
```

### 数据结构

```ts
type Props = {
  elapsedTimeSeconds?: number;  // 已执行秒数
  timeoutMs?: number;           // 超时毫秒数
}
```

### 格式化细节

- **运行时长**：调用 `formatDuration(t2)`（`t2 = elapsedTimeSeconds * 1000`），不传递 `hideTrailingZeros`，因此会保留完整精度（如 `1m 30s`）。
- **超时**：调用 `formatDuration(timeoutMs, { hideTrailingZeros: true })`，会隐藏末尾的零值字段（如 `1h` 而非 `1h 0m 0s`）。

## 关键代码路径与文件引用

### 本文件内部

| 行号 | 内容 |
|------|------|
| 5-8 | `Props` 类型定义 |
| 9-73 | `ShellTimeDisplay` 组件主体（React Compiler 产物） |

### 上游调用方

| 路径 | 用途 |
|------|------|
| `src/components/shell/ShellProgressMessage.tsx:8` | 在进度消息中展示时间/超时 |
| `src/components/shell/ShellProgressMessage.tsx:65` | 无输出时的 "Running…" 状态旁 |
| `src/components/shell/ShellProgressMessage.tsx:111` | 有输出时的元信息行中 |
| `src/tools/BashTool/BashToolResultMessage.tsx:7` | 结果页展示超时限制 |
| `src/tools/BashTool/BashToolResultMessage.tsx:169` | `timeoutMs && <ShellTimeDisplay timeoutMs={timeoutMs} />` |
| `src/tools/PowerShellTool/UI.tsx:8` | 结果页展示超时限制 |
| `src/tools/PowerShellTool/UI.tsx:116` | `<ShellTimeDisplay timeoutMs={timeoutMs} />` |

### 下游依赖

| 路径 | 用途 |
|------|------|
| `src/utils/format.ts` | `formatDuration` 时间格式化函数 |
| `src/ink.ts` | `Text` 组件 |

## 依赖与外部交互

- **`formatDuration`**（`src/utils/format.ts`）：核心格式化逻辑，支持毫秒到 `Xd Xh Xm Xs` 的转换，具备进位处理（如 59.5s 进位到 60s 再触发分钟进位）和 `hideTrailingZeros`、`mostSignificantOnly` 选项。
- **Ink `Text`**：使用 `dimColor` 样式，使时间信息以低对比度灰色呈现，避免抢夺主输出内容的视觉焦点。
- **React Compiler**：产物中使用了 `_c(10)` 等 memo cache，对纯展示组件的 props 变化做细粒度缓存。

## 风险、边界与改进建议

1. **`elapsedTimeSeconds * 1000` 的精度**  
   当 `elapsedTimeSeconds` 为整数秒时，乘以 1000 得到整毫秒，不会引入浮点误差。若未来改为传入浮点秒数（如 `1.5`），`formatDuration` 的 `< 60000` 分支会将其 `Math.floor` 为 `1s`，丢失小数精度。当前调用方（如 `BashTool`）通常以整数秒更新，因此无问题。

2. **timeout 单独展示时的语义**  
   在 `BashToolResultMessage` 和 `PowerShellTool/UI` 中，超时仅在结果页展示，且不与 `elapsedTimeSeconds` 同时出现（结果页通常无运行时长）。这与 `ShellProgressMessage` 中的组合展示形成互补，整体语义一致。

3. **无计时器，完全受控**  
   组件本身不持有 `setInterval` 或 `useEffect` 计时逻辑，所有时间值由父组件或全局状态驱动。这保证了组件的纯粹性和可测试性，但也意味着如果父组件更新频率低（如每 5 秒才推送新 props），时间显示会显得“跳跃”。

4. **扩展性**  
   若未来需要展示“剩余时间”而非“已运行时间 + 超时”，当前组件无法直接支持，需要新增 props 或在上层计算后传入。考虑到当前需求稳定，保持简单是合理选择。

5. **测试覆盖**  
   仓库中未检索到针对 `ShellTimeDisplay` 的单元测试。建议补充以下测试：
   - 各种 props 组合下的返回节点（`null`、仅 elapsed、仅 timeout、两者皆有）
   - `formatDuration` 的调用参数验证（特别是 `hideTrailingZeros` 的差异）
   - 边界值：`elapsedTimeSeconds = 0`、`timeoutMs = 0`、极大值
