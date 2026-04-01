# 研究文档：src/utils/formatBriefTimestamp.ts

## 场景与职责

`formatBriefTimestamp.ts` 负责将 ISO 时间戳格式化为聊天界面中消息标签行的简短时间显示。它的设计目标是**像即时通讯应用一样，根据消息“年龄”动态调整显示粒度**，从而在保证信息密度的同时提升可读性。

该模块特别处理了 POSIX locale 环境变量，因为 Bun/V8 的 `toLocaleString(undefined)` 在 macOS 上会忽略 `LC_ALL`/`LC_TIME`/`LANG`，所以模块内部手动将这些环境变量转换为 BCP 47 locale tag。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `formatBriefTimestamp(isoString, now?)` | 将 ISO 字符串格式化为适合消息气泡/标签的简短时间文本。 |

显示规则：
- **同一天**：`1:30 PM` 或 `13:30`（取决于 locale 的 12h/24h 偏好）
- **6 天内**：`Sunday, 4:15 PM`
- **更久**：`Sunday, Feb 20, 4:30 PM`

## 具体技术实现

### 主函数逻辑

```ts
export function formatBriefTimestamp(isoString: string, now: Date = new Date()): string {
  const d = new Date(isoString)
  if (Number.isNaN(d.getTime())) return ''

  const locale = getLocale()
  const dayDiff = startOfDay(now) - startOfDay(d)
  const daysAgo = Math.round(dayDiff / 86_400_000)

  if (daysAgo === 0) {
    return d.toLocaleTimeString(locale, { hour: 'numeric', minute: '2-digit' })
  }
  if (daysAgo > 0 && daysAgo < 7) {
    return d.toLocaleString(locale, { weekday: 'long', hour: 'numeric', minute: '2-digit' })
  }
  return d.toLocaleString(locale, { weekday: 'long', month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit' })
}
```

### Locale 解析（`getLocale`）

```ts
function getLocale(): string | undefined {
  const raw = process.env.LC_ALL || process.env.LC_TIME || process.env.LANG || ''
  if (!raw || raw === 'C' || raw === 'POSIX') return undefined

  const base = raw.split('.')[0]!.split('@')[0]!
  const tag = base.replaceAll('_', '-')

  try {
    new Intl.DateTimeFormat(tag)
    return tag
  } catch {
    return undefined
  }
}
```

- 优先级：`LC_ALL > LC_TIME > LANG`
- 剥离 codeset（`.UTF-8`）和 modifier（`@euro`）。
- 将 POSIX 下划线格式（`en_GB`）转换为 BCP 47 连字符格式（`en-GB`）。
- 通过尝试构造 `Intl.DateTimeFormat` 验证 tag 有效性，无效时回退到系统默认（`undefined`）。

### 日期对齐（`startOfDay`）

```ts
function startOfDay(d: Date): number {
  return new Date(d.getFullYear(), d.getMonth(), d.getDate()).getTime()
}
```

- 使用本地时间的年/月/日构造新 Date，从而忽略时区对“同一天”判定的影响。

## 关键代码路径与文件引用

### 调用方

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/tools/BriefTool/UI.tsx` | `formatBriefTimestamp` | Brief 模式消息列表的时间标签。 |
| `src/components/messages/HighlightedThinkingText.tsx` | `formatBriefTimestamp` | Thinking 文本高亮组件中的时间显示。 |

### 被调用方

- `Intl.DateTimeFormat`：内置 API，用于 locale 验证和时间格式化。

## 依赖与外部交互

- 无网络依赖。
- 无持久化。
- 依赖 `process.env` 中的 POSIX locale 变量。

## 风险、边界与改进建议

### 风险

1. **环境变量读取频率**：每次调用 `formatBriefTimestamp` 都会重新读取 `process.env` 并构造 `Intl.DateTimeFormat`。虽然通常调用频率不高（消息列表渲染），但在大量消息滚动时可能造成不必要的 GC 压力。
2. **时区歧义**：`startOfDay` 基于本地时区计算 `daysAgo`，若消息时间戳来自不同时区（如跨时区协作），"同一天"的判定可能与用户直觉不符。
3. **无效 ISO 字符串**：对无法解析的输入返回空字符串 `''`，调用方需自行处理缺失时间的 UI fallback。

### 边界

- 只处理**过去**的时间戳；对未来时间（如 `daysAgo < 0`）会落入 `daysAgo < 7` 分支，显示 `weekday, time`，不会显示 "in X days"。
- 不显示秒级精度。
- 不显示年份，除非调用方在更外层处理。
- `now` 参数仅用于测试注入，生产代码通常不传。

### 改进建议

1. **缓存 locale**：将 `getLocale()` 结果缓存为模块级变量，并监听 `process.env` 变化（若可行）或在应用启动时只计算一次，减少重复解析。
2. **未来时间处理**：明确区分过去和未来，对未来时间显示 `in X days` 或完整日期，避免与过去时间混淆。
3. **时区感知**：若产品需要，可增加 `timeZone` 选项，让 `startOfDay` 基于指定时区而非本地时区计算。
4. **测试覆盖**：补充对跨 DST（夏令时）边界、无效 locale tag（如 `zh_CN.GB2312`）、以及 `C`/`POSIX` 环境的单元测试。
