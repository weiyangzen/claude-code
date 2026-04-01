# ShellProgressMessage.tsx 研究文档

## 场景与职责

`ShellProgressMessage.tsx` 负责渲染**正在执行中的 shell 命令**的实时进度 UI。它出现在 `BashTool`、`PowerShellTool` 以及 `BashModeProgress` 等场景中，向用户展示：

- 命令已产生的部分输出（默认显示最后 5 行，verbose 模式显示全部）
- 运行时长与超时倒计时
- 输出行数统计与字节大小
- “Running…” 占位状态（当尚无输出时）

该组件通过 `OffscreenFreeze` 对滚动出视口的内容进行冻结，避免后台定时更新导致终端 full reset 的性能问题。

## 功能点目的

### 1. 输出预览（非 verbose 模式）
当 `verbose === false` 时，组件从 `output` 中剥离 ANSI 码，取最后 5 行非空行展示，让用户既能看到最新进展，又不会被历史输出淹没。

### 2. 全量输出（verbose 模式）
当 `verbose === true` 时，直接展示 `fullOutput` 的完整内容（同样先 `stripAnsi` 清理）。

### 3. 元信息展示
- **行数状态**：若 `totalLines` 与 `totalBytes` 同时存在且非 verbose，显示 `~{totalLines} lines`；否则若存在隐藏行数，显示 `+{extraLines} lines`。
- **文件大小**：若 `totalBytes` 存在，显示经 `formatFileSize` 格式化后的大小（如 `1.5KB`）。
- **时间**：通过 `ShellTimeDisplay` 展示已运行时长与超时限制。

### 4. 视口外冻结
整个进度区域被 `OffscreenFreeze` 包裹。当内容滚动到终端视口上方（进入 scrollback）时，React 子树停止重渲染，防止每秒更新的进度/计时器触发 `log-update.ts` 的整屏重绘。

## 具体技术实现

### 关键流程

```
接收 props: output, fullOutput, elapsedTimeSeconds, totalLines, totalBytes, timeoutMs, taskId, verbose
    │
    ▼
strippedFullOutput = stripAnsi(fullOutput.trim())
strippedOutput     = stripAnsi(output.trim())
lines              = strippedOutput.split('\n').filter(Boolean)
    │
    ▼
displayLines = verbose ? strippedFullOutput : lines.slice(-5).join('\n')
    │
    ├─ lines.length === 0
    │      → 渲染 <Text dimColor>Running…</Text> + <ShellTimeDisplay />
    │        （包裹在 OffscreenFreeze 内）
    │
    └─ lines.length > 0
           → 渲染输出文本（Box 限高）
           → 渲染 lineStatus / ShellTimeDisplay / totalBytes
           → 整体包裹在 OffscreenFreeze + MessageResponse 中
```

### 数据结构

```ts
type Props = {
  output: string;               // 当前累积输出（可能截断）
  fullOutput: string;           // 完整输出
  elapsedTimeSeconds?: number;  // 已运行秒数
  totalLines?: number;          // 总行数（服务端统计）
  totalBytes?: number;          // 总字节数（服务端统计）
  timeoutMs?: number;           // 超时毫秒数
  taskId?: string;              // 后台任务 ID（当前组件内未使用）
  verbose: boolean;             // 是否展开全部
}
```

### 布局结构

```
<MessageResponse>
  <OffscreenFreeze>
    <Box flexDirection="column">
      {/* 输出预览区域 */}
      <Box height={min(5, lines.length)} flexDirection="column" overflow="hidden">
        <Text dimColor>{displayLines}</Text>
      </Box>
      {/* 元信息行 */}
      <Box flexDirection="row" gap={1}>
        {lineStatus && <Text dimColor>{lineStatus}</Text>}
        <ShellTimeDisplay elapsedTimeSeconds={...} timeoutMs={...} />
        {totalBytes && <Text dimColor>{formatFileSize(totalBytes)}</Text>}
      </Box>
    </Box>
  </OffscreenFreeze>
</MessageResponse>
```

## 关键代码路径与文件引用

### 本文件内部

| 行号 | 内容 |
|------|------|
| 9-18 | `Props` 类型定义 |
| 19-146 | `ShellProgressMessage` 组件主体（React Compiler 产物） |
| 147-149 | `_temp` 过滤函数（`filter(Boolean)` 的等价体） |

### 上游调用方

| 路径 | 用途 |
|------|------|
| `src/tools/BashTool/UI.tsx:7` | 导入并渲染 bash 执行进度 |
| `src/tools/BashTool/UI.tsx:152` | 传入 `fullOutput/output/elapsedTimeSeconds/...` |
| `src/tools/PowerShellTool/UI.tsx:7` | 导入并渲染 PowerShell 执行进度 |
| `src/tools/PowerShellTool/UI.tsx:71` | 同上，传入进度数据 |
| `src/components/BashModeProgress.tsx:7` | bash 模式下的进度展示 |
| `src/components/BashModeProgress.tsx:34` | 传入 `ShellProgress` 数据 |

### 下游依赖

| 路径 | 用途 |
|------|------|
| `src/components/shell/ShellTimeDisplay.tsx` | 时间/超时展示 |
| `src/components/MessageResponse.tsx` | 统一消息前缀包装 |
| `src/components/OffscreenFreeze.tsx` | 视口外渲染冻结 |
| `src/utils/format.ts` | `formatFileSize` |
| `src/ink.ts` | `Box`、`Text` |
| `strip-ansi` (npm) | 清理 ANSI 转义序列 |

## 依赖与外部交互

- **`strip-ansi`**：用于在渲染前移除输出中的 ANSI 颜色/光标控制序列，确保进度预览的文本干净、高度可控。
- **`OffscreenFreeze`**：通过 `useTerminalViewport` 检测元素是否可见，结合 `InVirtualListContext` 排除虚拟列表场景，实现“出视口即冻结”。
- **React Compiler**：产物中使用了 `_c(30)` 等 memo cache，但 `OffscreenFreeze` 内部显式标记 `'use no memo'`，因此冻结机制不会被编译器优化破坏。

## 风险、边界与改进建议

1. **`taskId` 未在组件内使用**  
   `Props` 中声明了 `taskId?: string`，但组件实现中完全未引用。该字段可能是为后续功能（如点击跳转到后台任务管理）预留，或历史遗留。建议清理未使用字段，或在需要时实现对应交互。

2. **ANSI 全 stripping 导致颜色信息丢失**  
   进度预览使用 `stripAnsi` 完全移除 ANSI 序列，用户无法在实时预览中看到错误高亮（红色 stderr）。这与最终结果页 `OutputLine` 保留 ANSI 的行为不一致。若需求允许，可考虑仅对 stdout 保留 ANSI，或对 stderr 单独用颜色组件渲染。

3. **`lines.slice(-5)` 的空白行处理**  
   当前 `lines = strippedOutput.split('\n').filter(_temp)` 会过滤掉空字符串行。若输出中存在有意义的空行（如段落分隔），这些空行会被吞掉，导致最后 5 行与实际输出不完全对应。这是为了紧凑展示而做的取舍，但在某些场景（如表格、代码块）可能失真。

4. **Box 高度计算与行数对应关系** 
   非 verbose 模式下 `height = Math.min(5, lines.length)`，与 `slice(-5)` 逻辑匹配。但如果某行内容在终端中因换行而占据多行视觉高度，Box 的 `height` 可能不足以容纳，导致内容被截断显示。由于 `stripAnsi` 后已无色码，但长行仍可能触发终端软换行，这是已知限制。

5. **测试覆盖**  
   仓库中未检索到针对 `ShellProgressMessage` 的单元测试。建议补充：
   - `verbose` true/false 的输出差异
   - `lines.length === 0` 时的 "Running…" 回退
   - `totalLines`/`totalBytes` 存在时的元信息行渲染
   - `OffscreenFreeze` 的集成行为（若测试框架支持视口模拟）
