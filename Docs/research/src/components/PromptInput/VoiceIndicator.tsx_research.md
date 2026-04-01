# VoiceIndicator.tsx 研究文档

## 场景与职责

为语音模式（Voice Mode）提供输入区状态反馈，包含三个导出：
- `VoiceIndicator`：根据 `voiceState` 在 footer 显示状态
- `VoiceWarmupHint`：长按语音激活键的预热阶段显示 `keep holding…`
- `ProcessingShimmer`：语音转文字后的处理阶段显示脉冲灰度动画

被 `Notifications.tsx` 和 `PromptInputFooterLeftSide.tsx` 引用。

## 功能点目的

- 明确告知用户当前处于"聆听中"还是"处理中"
- 预热提示防止用户误以为已激活
- 处理动画与 `ThinkingShimmerText` 视觉统一
- 尊重 `prefersReducedMotion` 设置

## 具体技术实现

### 关键流程

1. **Feature Gate**：`feature("VOICE_MODE")`（`bun:bundle`）编译期消除；false 时返回 `null`
2. **状态分发**：`recording` → `<Text dimColor>listening…</Text>`；`processing` → `<ProcessingShimmer />`；`idle` → `null`
3. **ProcessingShimmer**：
   - `settings.prefersReducedMotion` 为 true 时返回静态文本
   - 否则 `useAnimationFrame(50)` 驱动正弦脉冲，`opacity = (sin(elapsedSec * π) + 1) / 2`
   - 在 `PROCESSING_DIM rgb(153,153,153)` 与 `PROCESSING_BRIGHT rgb(185,185,185)` 间插值

### 数据结构

```ts
type Props = { voiceState: 'idle' | 'recording' | 'processing' }
const PROCESSING_DIM = { r: 153, g: 153, b: 153 }
const PROCESSING_BRIGHT = { r: 185, g: 185, b: 185 }
const PULSE_PERIOD_S = 2
```

### 协议/命令

- `useSettings()` → `src/hooks/useSettings.ts`
- `useAnimationFrame()` → `src/ink.js`
- `interpolateColor` / `toRGBColor` → `src/components/Spinner/utils.ts`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/VoiceIndicator.tsx` | 本组件 |
| `src/components/PromptInput/Notifications.tsx` | 渲染 `VoiceIndicator` |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 渲染 `VoiceWarmupHint` |
| `src/hooks/useSettings.ts` | 设置订阅 |
| `src/components/Spinner/utils.ts` | 颜色插值工具 |
| `src/ink.js` | UI 组件与动画 hook |

## 依赖与外部交互

- `feature()` 是编译期常量，允许条件分支内调用 hook
- `Notifications.tsx` 通过 `useVoiceState()`（`src/context/voice.js`）获取 `voiceState`

## 风险、边界与改进建议

1. **预热静态化设计合理**：`VoiceWarmupHint` 故意无动画，避免 ~120ms 预热窗口与 50ms 动画定时器并发导致额外重渲染
2. **颜色硬编码**：`PROCESSING_DIM/BRIGHT` 是写死灰度 RGB，未走 theme 系统
3. **PULSE_PERIOD_S 全局共享注释**：常量仅在本文件使用，建议统一提取到动画常量文件
4. **测试建议**：补充 `prefersReducedMotion=true` 静态文本、`voiceState='idle'` 返回 null、动画颜色正弦变化断言
