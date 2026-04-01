# 研究文档：src/utils/format.ts

## 场景与职责

`format.ts` 是 Claude Code 的**纯显示格式化工具箱**，专门负责将各种原始数值（字节、毫秒、token 数、时间戳等）转换为人类可读的字符串。该模块被明确设计为 **leaf-safe**：不依赖 Ink 或其他 UI 框架，因此可以在任何层级安全导入，包括工具层、服务层和 CLI 输出层。

宽度感知的截断逻辑被拆分到同目录的 `truncate.ts`（需要 `stringWidth` 和 Ink），`format.ts` 仅通过 re-export 提供向后兼容。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `formatFileSize(bytes)` | 将字节数格式化为 `1.5KB`、`2.3MB`、`4.1GB` 等。 |
| `formatSecondsShort(ms)` | 将毫秒格式化为保留 1 位小数的秒（如 `1.2s`）。 |
| `formatDuration(ms, options?)` | 将毫秒格式化为 `2d 3h 15m`、`45s` 等，支持隐藏末尾零和只显示最高有效单位。 |
| `formatNumber(n)` | 使用 `Intl.NumberFormat` 的 compact 记法（如 `1.3k`、`900`）。 |
| `formatTokens(count)` | 在 `formatNumber` 基础上去掉 `.0` 后缀。 |
| `formatRelativeTime(date, options?)` | 输出相对时间（narrow 风格如 `3h ago`，long 风格用 `Intl.RelativeTimeFormat`）。 |
| `formatRelativeTimeAgo(date)` | 强制 past-tense 的 `formatRelativeTime` 包装。 |
| `formatLogMetadata(log)` | 格式化会话日志的元数据摘要（修改时间、分支、大小、标签等）。 |
| `formatResetTime(seconds, ...)` | 将秒级时间戳格式化为本地化的重置时间（支持跨天时显示日期）。 |
| `formatResetText(resetsAt, ...)` | `formatResetTime` 的字符串输入包装。 |
| re-exports from `truncate.ts` | `truncate`, `truncatePathMiddle`, `truncateStartToWidth`, `truncateToWidth`, `truncateToWidthNoEllipsis`, `wrapText` |

## 具体技术实现

### 文件大小格式化

```ts
export function formatFileSize(sizeInBytes: number): string {
  const kb = sizeInBytes / 1024
  if (kb < 1) return `${sizeInBytes} bytes`
  if (kb < 1024) return `${kb.toFixed(1).replace(/\.0$/, '')}KB`
  // ... MB, GB
}
```

- 使用 1024 进制。
- 去掉无意义的小数尾零（如 `2.0MB` → `2MB`）。

### 时长格式化（`formatDuration`）

- `< 1ms`：显示 `0.0x`s 或 `0.x`s（保留 1 位小数）。
- `< 60s`：整数秒。
- `≥ 60s`：逐级计算 days/hours/minutes/seconds，并处理 rounding carry-over（如 59.5s → 60s → 1m 0s）。
- `mostSignificantOnly` 选项只返回最大非零单位（如 `2d`）。
- `hideTrailingZeros` 选项省略末尾的零单位（如 `2d 3h` 而非 `2d 3h 0m`）。

### 数字紧凑记法（`formatNumber`）

缓存了两个 `Intl.NumberFormat` 实例：
- `useConsistentDecimals = true`（`≥ 1000`）：`minimumFractionDigits: 1`，保证 `1.0k` 与 `1.3k` 视觉对齐。
- `useConsistentDecimals = false`（`< 1000`）：`minimumFractionDigits: 0`，避免 `900.0` 这种不自然显示。

### 相对时间（`formatRelativeTime`）

- Narrow 风格（默认）：自定义短单位（`y`, `mo`, `w`, `d`, `h`, `m`, `s`）。
- Long 风格：调用 `getRelativeTimeFormat('long', numeric).format(value, unit)`。
- 对于天及以上的 long 风格，强制使用 `long` 无视传入的 `style` 参数（设计决策）。

### 重置时间（`formatResetTime`）

- 若重置时间 > 24 小时：显示 `Mon, Feb 20, 4:30pm` 或带年份。
- 若 ≤ 24 小时：只显示时间 `4:30pm`。
- 统一去掉 AM/PM 前的空格并小写化（`1:30pm` 而非 `1:30 PM`）。
- 可选附加时区（如 `(PST)`）。

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/utils/status.tsx` | `formatNumber` | 状态显示。 |
| `src/utils/contextSuggestions.ts` | `formatTokens` | 上下文建议中的 token 数。 |
| `src/utils/pdf.ts` | `formatFileSize` | PDF 大小显示。 |
| `src/utils/statusNoticeDefinitions.tsx` | `formatNumber` | 状态通知。 |
| `src/utils/imageResizer.ts` | `formatFileSize` | 图片大小。 |
| `src/utils/profilerBase.ts` | `formatFileSize` | 性能分析器。 |
| `src/utils/logoV2Utils.ts` | `formatFileSize`, `formatDuration` 等 | Logo 相关工具。 |
| `src/utils/readFileInRange.ts` | `formatFileSize` | 文件范围读取提示。 |
| `src/utils/toolResultStorage.ts` | `formatFileSize` | 工具结果存储。 |
| `src/utils/ShellCommand.ts` | `formatDuration` | 命令执行时长。 |
| `src/utils/mcpOutputStorage.ts` | `formatFileSize` | MCP 输出存储。 |
| `src/utils/imageValidation.ts` | `formatFileSize` | 图片校验。 |
| `src/utils/sessionStorage.ts` | `formatFileSize` | 会话存储。 |
| `src/utils/messages.ts` | `formatNumber`, `formatTokens`, `formatFileSize` | 消息格式化。 |
| `src/utils/teleport.tsx` | `truncateToWidth` | 文本截断（re-export）。 |

### 被调用方

- `src/utils/intl.js`：`getRelativeTimeFormat`, `getTimeZone`
- `src/utils/truncate.js`：各种截断/换行函数（re-export）
- `Intl.NumberFormat` / `Intl.DateTimeFormat` / `Intl.RelativeTimeFormat`：浏览器/Node 内置。

## 依赖与外部交互

- 无网络依赖。
- 无持久化。
- 依赖 `Intl` API，在旧版 Node.js 或某些精简运行环境中可能行为不一致。

## 风险、边界与改进建议

### 风险

1. **`Intl` 缓存泄漏**：`numberFormatterForConsistentDecimals` 和 `numberFormatterForInconsistentDecimals` 是模块级单例，若运行环境切换 locale（极罕见），不会自动刷新。
2. **`formatDuration` 的 rounding 误差**：`Math.round((ms % 60000) / 1000)` 在边界值（如 59999ms）上可能产生 `60s`，虽然代码有 carry-over 处理，但复杂选项组合下仍可能出错。
3. **相对时间的 narrow 风格硬编码**：`mo` 作为月份缩写并非所有语言都通用，但当前实现未做 i18n 扩展。

### 边界

- `formatFileSize` 对小于 1KB 的文件显示 `N bytes`，不显示小数 KB。
- `formatNumber` 的 compact 记法在 `en-US` 下工作，未根据用户 locale 动态切换。
- `formatResetTime` 的时区显示依赖 `getTimeZone()`，后者在 `intl.js` 中实现，可能返回 `undefined`。

### 改进建议

1. **Locale 感知**：将 `en-US` 替换为从用户环境或配置中读取的 locale，提升非英语用户体验。
2. **测试覆盖**：补充对 `formatDuration` 边界值（如 59999ms、86399999ms、86400000ms）的单元测试，确保 carry-over 逻辑正确。
3. **性能优化**：`formatRelativeTime` 每次调用都重新遍历 `intervals` 数组，虽然数组很小，但高频调用（如列表渲染）可考虑预计算或缓存。
4. **类型安全**：`formatResetTime` 的 `showTime` / `showTimezone` 参数可考虑合并为 options 对象，提高可读性。
