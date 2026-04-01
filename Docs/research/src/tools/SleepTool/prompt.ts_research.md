# SleepTool/prompt.ts 深度研究文档

## 1. 场景与职责

### 1.1 功能定位

`src/tools/SleepTool/prompt.ts` 是 Claude Code 中 **SleepTool（睡眠工具）** 的提示词配置文件。该工具属于 **Proactive（主动式）模式** 和 **KAIROS 模式** 的核心组件，用于在自主运行模式下控制 AI 的"睡眠-唤醒"周期。

### 1.2 核心场景

| 场景 | 描述 |
|------|------|
| **Proactive 模式** | AI 在没有用户输入时自主运行，通过 SleepTool 控制检查频率 |
| **KAIROS 模式** | 助手模式，AI 持续监控并响应外部事件（如 Channel 通知） |
| **Tick 处理** | 接收 `<tick>` 周期性提示，决定继续睡眠或执行工作 |
| **资源优化** | 避免不必要的 API 调用，平衡响应速度与成本 |

### 1.3 与相关组件的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        SleepTool 生态                            │
├─────────────────────────────────────────────────────────────────┤
│  prompt.ts (本文件)                                              │
│    ├── 定义工具名称: SLEEP_TOOL_NAME = 'Sleep'                   │
│    ├── 定义工具描述: DESCRIPTION                                 │
│    └── 定义系统提示: SLEEP_TOOL_PROMPT                           │
├─────────────────────────────────────────────────────────────────┤
│  SleepTool.js (运行时实现，条件加载)                              │
│    ├── 由 tools.ts 在 feature('PROACTIVE')/feature('KAIROS') 时加载 │
│    ├── 实现 interruptBehavior() → 'cancel'                       │
│    └── 实现 isEnabled() → isProactiveActive()                    │
├─────────────────────────────────────────────────────────────────┤
│  调用方                                                          │
│    ├── constants/prompts.ts → getProactiveSection() 引用         │
│    ├── REPL.tsx → 检测可中断工具状态                              │
│    ├── handlePromptSubmit.ts → 中断逻辑处理                       │
│    └── channelNotification.ts → SleepTool 唤醒机制                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 功能点目的

### 2.1 导出常量说明

```typescript
// 工具名称常量 - 用于工具注册和系统提示引用
export const SLEEP_TOOL_NAME = 'Sleep'

// 工具简短描述 - 用于工具列表展示
export const DESCRIPTION = 'Wait for a specified duration'

// 完整的工具使用提示 - 指导模型如何正确使用 Sleep 工具
export const SLEEP_TOOL_PROMPT = `...`
```

### 2.2 SLEEP_TOOL_PROMPT 内容解析

提示词包含以下关键指导：

| 段落 | 目的 |
|------|------|
| **功能说明** | "Wait for a specified duration. The user can interrupt..." - 说明基本功能和可中断特性 |
| **使用场景** | "Use this when the user tells you to sleep..." - 定义何时应该使用 |
| **Tick 处理** | "You may receive <tick> prompts..." - 指导如何处理周期性检查 |
| **并发说明** | "You can call this concurrently with other tools..." - 说明可并行执行 |
| **最佳实践** | "Prefer this over `Bash(sleep ...)`..." - 推荐使用而非 Bash sleep |
| **成本提示** | "Each wake-up costs an API call..." - 提醒缓存过期和成本权衡 |

### 2.3 TICK_TAG 依赖

```typescript
import { TICK_TAG } from '../../constants/xml.js'
```

`TICK_TAG = 'tick'` 定义了周期性唤醒的 XML 标签名称，在 proactive 模式下系统会发送 `<tick>` 消息触发 AI 检查是否有工作要做。

---

## 3. 具体技术实现

### 3.1 条件加载机制

SleepTool 采用**条件编译/加载**策略，仅在特定功能标志启用时可用：

**文件: `src/tools.ts` (第 25-28 行)**
```typescript
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null
```

**文件: `src/tools.ts` (第 234 行)**
```typescript
...(SleepTool ? [SleepTool] : []),
```

### 3.2 工具池集成

SleepTool 被集成到基础工具池中：

```typescript
// getAllBaseTools() 返回的工具列表包含 SleepTool（当功能启用时）
export function getAllBaseTools(): Tools {
  return [
    // ... 其他工具
    ...(SleepTool ? [SleepTool] : []),
    // ... 其他工具
  ]
}
```

### 3.3 系统提示集成

**文件: `src/constants/prompts.ts`**

```typescript
// 第 57 行: 导入 SleepTool 名称
import { SLEEP_TOOL_NAME } from '../tools/SleepTool/prompt.js'

// 第 872 行: 在 Proactive 模式下指导使用 SleepTool
function getProactiveSection(): string | null {
  return `# Autonomous work

// ...

## Pacing

Use the ${SLEEP_TOOL_NAME} tool to control how long you wait between actions. 
Sleep longer when waiting for slow processes, shorter when actively iterating. 
Each wake-up costs an API call, but the prompt cache expires after 5 minutes 
of inactivity — balance accordingly.

**If you have nothing useful to do on a tick, you MUST call ${SLEEP_TOOL_NAME}.** 
Never respond with only a status message like "still waiting"...`
}
```

### 3.4 中断行为实现

**文件: `src/Tool.ts` (第 407-416 行)**

定义了工具的中断行为接口：

```typescript
/**
 * What should happen when the user submits a new message while this tool
 * is running.
 *
 * - `'cancel'` — stop the tool and discard its result
 * - `'block'`  — keep running; the new message waits
 *
 * Defaults to `'block'` when not implemented.
 */
interruptBehavior?(): 'cancel' | 'block'
```

SleepTool 实现 `interruptBehavior() → 'cancel'`，意味着当用户提交新消息时，正在进行的 Sleep 会被取消。

### 3.5 中断处理流程

**文件: `src/utils/handlePromptSubmit.ts` (第 319-332 行)**

```typescript
// Interrupt the current turn when all executing tools have
// interruptBehavior 'cancel' (e.g. SleepTool).
if (params.hasInterruptibleToolInProgress) {
  logForDebugging(
    `[interrupt] Aborting current turn: streamMode=${params.streamMode}`,
  )
  logEvent('tengu_cancel', {
    source: 'interrupt_on_submit',
    streamMode: params.streamMode,
  })
  params.abortController?.abort('interrupt')
}
```

**文件: `src/screens/REPL.tsx`**

REPL 组件跟踪可中断工具状态：
- 通过 `setHasInterruptibleToolInProgress` 设置状态
- 在 `handlePromptSubmit` 中检查此状态决定是否中断

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
src/tools/SleepTool/prompt.ts
    │
    ├── 导入 ───────────────────────────────┐
    │   └── src/constants/xml.ts (TICK_TAG)  │
    │                                        │
    ├── 被导入/引用 ─────────────────────────┤
    │   ├── src/tools.ts                     │
    │   │   └── 条件加载 SleepTool.js        │
    │   ├── src/constants/prompts.ts         │
    │   │   └── getProactiveSection()        │
    │   ├── src/query.ts                     │
    │   │   └── SLEEP_TOOL_NAME 引用         │
    │   ├── src/screens/REPL.tsx             │
    │   │   └── SLEEP_TOOL_NAME 引用         │
    │   ├── src/cli/print.ts                 │
    │   │   └── Proactive 模式处理           │
    │   ├── src/utils/handlePromptSubmit.ts  │
    │   │   └── 中断逻辑                     │
    │   └── src/utils/permissions/classifierDecision.ts
    │       └── 自动模式白名单               │
    │                                        │
    └── 运行时实现 ──────────────────────────┘
        └── src/tools/SleepTool/SleepTool.js (条件编译，非源码可见)
```

### 4.2 关键代码路径

#### 路径 1: 工具加载
```
main.tsx:1867 → maybeActivateProactive()
    ↓
tools.ts:234 → getAllBaseTools() 包含 SleepTool
    ↓
tools.ts:25-28 → 条件加载 SleepTool.js
```

#### 路径 2: 系统提示生成
```
constants/prompts.ts:444 → getSystemPrompt()
    ↓
constants/prompts.ts:860-914 → getProactiveSection()
    ↓
引用 SLEEP_TOOL_NAME 和 SLEEP_TOOL_PROMPT
```

#### 路径 3: 用户中断
```
REPL.tsx → handlePromptSubmit
    ↓
handlePromptSubmit.ts:319-332 → 检查 hasInterruptibleToolInProgress
    ↓
abortController.abort('interrupt') → 取消 SleepTool
```

#### 路径 4: Channel 通知唤醒
```
services/mcp/channelNotification.ts:9
    ↓
"SleepTool polls hasCommandsInQueue() and wakes within 1s"
```

### 4.3 类型定义引用

**文件: `src/types/textInputTypes.ts` (第 277-294 行)**

```typescript
/**
 * Queue priority levels. Same semantics in both normal and proactive mode.
 *
 *  - `now`   — Interrupt and send immediately...
 *  - `next`  — Mid-turn drain. Let the current tool call finish, then
 *              send this message between the tool result and the next API
 *              round-trip. Wakes an in-progress SleepTool call.
 *  - `later` — End-of-turn drain. Wait for the current turn to finish,
 *              then process as a new query. Wakes an in-progress SleepTool
 *              call (query.ts upgrades the drain threshold after sleep so
 *              the message is attached to the same turn).
 *
 * The SleepTool is only available in proactive mode, so "wakes SleepTool"
 * is a no-op in normal mode.
 */
export type QueuePriority = 'now' | 'next' | 'later'
```

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `TICK_TAG` | `src/constants/xml.ts` | 周期性唤醒标签 |

### 5.2 反向依赖（引用本文件的模块）

| 模块 | 引用内容 | 用途 |
|------|----------|------|
| `src/tools.ts` | `SleepTool.js` 加载 | 工具注册 |
| `src/constants/prompts.ts` | `SLEEP_TOOL_NAME` | 系统提示生成 |
| `src/query.ts` | `SLEEP_TOOL_NAME` | 查询处理 |
| `src/screens/REPL.tsx` | `SLEEP_TOOL_NAME` | UI 交互 |
| `src/cli/print.ts` | `SleepTool` | 非交互模式处理 |
| `src/utils/handlePromptSubmit.ts` | 中断逻辑 | 用户输入处理 |
| `src/utils/permissions/classifierDecision.ts` | `SLEEP_TOOL_NAME` | 自动模式白名单 |
| `src/services/mcp/channelNotification.ts` | `SleepTool` | Channel 通知唤醒 |

### 5.3 功能标志依赖

| 标志 | 说明 |
|------|------|
| `feature('PROACTIVE')` | 主动模式开关 |
| `feature('KAIROS')` | 助手模式开关 |

### 5.4 运行时环境变量

| 环境变量 | 说明 |
|----------|------|
| `CLAUDE_CODE_PROACTIVE` | 启用 proactive 模式 |

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### 风险 1: 条件编译导致的类型不安全
```typescript
// tools.ts 第 25-28 行
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null
```
- **风险**: 使用动态 require 和条件加载，TypeScript 无法在编译时验证 SleepTool.js 的存在和类型
- **影响**: 如果 SleepTool.js 不存在或导出不匹配，运行时才报错

#### 风险 2: 提示缓存过期策略硬编码
```typescript
// prompt.ts 第 17 行
"Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity"
```
- **风险**: "5分钟"是硬编码的提示信息，与实际服务器端缓存策略可能不一致
- **影响**: 模型可能基于错误信息做出次优决策

#### 风险 3: 中断行为的副作用
- **风险**: SleepTool 被取消时，可能导致整个 turn 被中止
- **影响**: 用户输入时如果 SleepTool 正在运行，可能丢失当前的等待状态

### 6.2 边界情况

| 边界情况 | 行为 |
|----------|------|
| Proactive 模式未启用 | SleepTool 不会被加载，模型无法调用 |
| 用户在中断时提交输入 | SleepTool 被取消，新输入被处理 |
| Channel 通知到达时 | SleepTool 在 1 秒内被唤醒处理消息 |
| 并发调用 | SleepTool 可与其他工具并行执行 |
| 缓存过期 | 5 分钟后缓存失效，需要新的 API 调用 |

### 6.3 改进建议

#### 建议 1: 添加类型安全包装
```typescript
// 建议添加类型定义文件
interface SleepToolExports {
  SleepTool: Tool<z.ZodType<{ duration: number }>, void>
}

const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? (require('./tools/SleepTool/SleepTool.js') as SleepToolExports).SleepTool
    : null
```

#### 建议 2: 缓存过期时间配置化
```typescript
// 建议从配置读取而非硬编码
const CACHE_EXPIRY_MINUTES = process.env.CLAUDE_CODE_CACHE_EXPIRY ?? 5

export const SLEEP_TOOL_PROMPT = `...cache expires after ${CACHE_EXPIRY_MINUTES} minutes...`
```

#### 建议 3: 添加 SleepTool 状态监控
```typescript
// 建议添加调试/监控接口
interface SleepToolMonitor {
  getCurrentSleepDuration(): number
  getRemainingSleepTime(): number
  isInterrupted(): boolean
}
```

#### 建议 4: 文档化中断恢复策略
- 当前行为：中断后完全取消 SleepTool
- 建议：考虑添加"暂停/恢复"机制，允许中断后恢复剩余的睡眠时间

### 6.4 测试建议

| 测试场景 | 验证点 |
|----------|--------|
| Proactive 模式启用/禁用 | SleepTool 是否正确加载/卸载 |
| 用户中断 | SleepTool 是否正确取消，新输入是否处理 |
| Channel 通知 | SleepTool 是否在 1 秒内被唤醒 |
| 并发调用 | SleepTool 是否可与其他工具并行 |
| 长时间睡眠 | 超过 5 分钟后的缓存行为 |

---

## 7. 总结

`src/tools/SleepTool/prompt.ts` 是一个**配置型文件**，虽然代码量小（17 行），但在 Claude Code 的 Proactive/KAIROS 模式中扮演关键角色：

1. **定义了 Sleep 工具的契约** - 名称、描述、使用指南
2. **连接了多个子系统** - 工具系统、提示系统、中断系统、Channel 通知系统
3. **支持核心用户体验** - 自主运行模式下的资源优化和响应控制

该文件的设计体现了 Claude Code 的架构特点：**通过条件编译支持功能分级，通过清晰的提示词指导模型行为，通过标准接口集成到工具生态系统**。
