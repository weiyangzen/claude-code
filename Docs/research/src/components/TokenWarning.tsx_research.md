# TokenWarning.tsx 深度研究文档

> **文件路径**：`src/components/TokenWarning.tsx`  
> **技术栈**：React / TypeScript / Ink（终端 UI） / Bun 构建时特性裁剪（`bun:bundle`）  
> **研究日期**：2026-04-01

---

## 1. 场景与职责

`TokenWarning` 是 Claude Code 终端 UI 中负责**上下文容量告警与压缩状态透传**的专用 React 组件。它常驻于主输入区底部（PromptInput 的 Footer / Notifications 区域），根据当前会话的 token 消耗量、模型上下文窗口大小以及多种特性开关，实时向用户展示以下三类信息：

1. **容量告警**：当上下文接近或达到模型上限时，提示剩余百分比、建议执行 `/compact` 命令，或推荐升级到更大上下文窗口的模型（如 `opus[1m]` / `sonnet[1m]`）。
2. **自动压缩状态**：在开启 Auto-Compact 或 Reactive-Compact 时，显示 "% until auto-compact" 或 "% context used" 等渐进式提示。
3. **上下文折叠（Context Collapse）进度**：在实验性的 `CONTEXT_COLLAPSE` 模式下，通过子组件 `CollapseLabel` 实时订阅折叠引擎的统计 store，展示已折叠 / 待折叠的 span 数量及错误/空闲告警。

该组件**纯展示、无交互**，所有状态均来自外部 props（`tokenUsage`、`model`）和全局 store / 特性开关。其渲染结果直接决定用户是否能在 Footer 看到容量相关的颜色提示（warning / error / dimColor）。

---

## 2. 功能点目的

### 2.1 核心设计目标

| 目标 | 说明 |
|------|------|
| **防止上下文溢出导致请求失败** | 在 token 使用量接近阈值时提前告警，让用户主动 `/compact` 或换模型。 |
| **降低用户焦虑** | 通过颜色分级（warning / error）和百分比量化，让用户对"还剩多少"有直观感知。 |
| **兼容多种压缩策略** | 同一组件需同时适配传统 Auto-Compact、Reactive-Only-Compact 和 Context-Collapse 三种策略的 UI 表达。 |
| **避免误报** | 成功压缩后通过 `compactWarningStore` 短暂抑制告警，防止旧 token 计数在新 API 响应回来前闪烁。 |
| **驱动模型升级** | 对具备 1M 上下文升级资格的用户，在告警文案中内嵌 `/model xxx[1m]` 快捷指令，提升功能发现率。 |

### 2.2 三种显示模式详解

组件内部通过特性开关将 UI 划分为三条互斥/优先分支：

1. **Normal 模式**（默认）  
   文案示例：`Context low (12% remaining) · Run /compact to compact & continue`  
   颜色：warning（黄色）或 error（红色）。

2. **Reactive-Only 模式**（`REACTIVE_COMPACT` + GrowthBook `tengu_cobalt_raccoon`）  
   文案示例：`87% context used` 或 `12% until auto-compact`  
   颜色：`dimColor`（低对比度，避免过度打扰）。  
   该模式下 proactive autocompact 被完全抑制，组件仅做被动提示。

3. **Collapse 模式**（`CONTEXT_COLLAPSE`）  
   文案示例：`42 / 50 summarized` 或 `42 / 50 summarized · collapse errors: 3`  
   由 `CollapseLabel` 子组件通过 `useSyncExternalStore` 实时渲染。

---

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 组件接口与入口

```tsx
type Props = {
  tokenUsage: number;   // 当前已用 token 数（来自最后一次 API 响应的估算值）
  model: string;        // 当前主循环模型标识（如 claude-sonnet-4-20250514）
};
```

### 3.2 主渲染流程（TokenWarning）

```
1. 调用 calculateTokenWarningState(tokenUsage, model)
   → 得到 percentLeft / isAboveWarningThreshold / isAboveErrorThreshold
2. 调用 useCompactWarningSuppression()
   → 若当前处于 suppression 状态，直接 return null
3. 若 tokenUsage 未达 warning threshold，return null
4. 评估特性开关：
   a. REACTIVE_COMPACT → 检查 GB flag tengu_cobalt_raccoon
   b. CONTEXT_COLLAPSE → 动态 require 并调用 isContextCollapseEnabled()
5. 若处于 reactiveOnlyMode 或 collapseMode：
   → 用 getEffectiveContextWindowSize(model) 重新计算 displayPercentLeft
   → 公式：max(0, round((effectiveWindow - tokenUsage) / effectiveWindow * 100))
6. 若 collapseMode 为真：
   → 渲染 <CollapseLabel upgradeMessage={upgradeMessage} />
7. 否则：
   → 根据 showAutoCompactWarning / reactiveOnlyMode / isAboveErrorThreshold
      组合最终文案与颜色，渲染 <Box flexDirection="row"><Text ... /></Box>
```

### 3.3 CollapseLabel 子组件（useSyncExternalStore 订阅）

`CollapseLabel` 被刻意拆分为独立子组件，原因是它内部使用了 `useSyncExternalStore` 订阅 `contextCollapse` 模块的 `subscribe` / `getStats`。React 规则禁止在条件分支中调用 Hook，因此父组件 `TokenWarning` 先通过 `feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled()` 判断是否进入 Collapse 模式，**无条件地**渲染 `CollapseLabel`（实际上仅在进入该模式时才会被挂载到树中，但子组件自身不依赖条件调用 Hook）。

**Snapshot 字符串协议**：
```ts
const snapshot = `${collapsedSpans}|${stagedSpans}|${totalErrors}|${totalEmptySpawns}|${idleWarn}`;
// 示例："42|8|0|0|0"
```

在组件内通过 `split('|').map(Number)` 反序列化为 5 元组：
- `collapsed`：已提交的折叠 span 数
- `staged`：待提交（staged）的折叠 span 数
- `errors`：折叠引擎累计错误数
- `emptySpawns`：空运行（未产生折叠）的累计次数
- `idleWarn`：`emptySpawnWarningEmitted` 的布尔标志（0/1）

**渲染优先级**：
1. 若 `errors > 0` 或 `idleWarn === 1`：以 `color="warning"` 高亮问题描述。
2. 若 `total === 0`（无任何折叠动作）：return `null`。
3. 正常情况：`dimColor` 显示 `collapsed / total summarized`，并可选拼接 `upgradeMessage`。

### 3.4 百分比与阈值计算（autoCompact.ts）

`calculateTokenWarningState` 返回的阈值体系：

```ts
{
  percentLeft,           // 基于 threshold 的剩余百分比
  isAboveWarningThreshold,  // tokenUsage >= threshold - 20_000
  isAboveErrorThreshold,    // tokenUsage >= threshold - 20_000（与 warning 同值，或后续可独立调整）
  isAboveAutoCompactThreshold,
  isAtBlockingLimit,
}
```

其中 `threshold` 的取值逻辑：
- 若 `isAutoCompactEnabled()` 为真 → `threshold = getAutoCompactThreshold(model)`
- 否则 → `threshold = getEffectiveContextWindowSize(model)`

`getEffectiveContextWindowSize` 的计算：
```ts
contextWindowForModel(model, sdkBetas)
  - min(getMaxOutputTokensForModel(model), 20_000)   // 为 summary 预留输出空间
  - 可选被 CLAUDE_CODE_AUTO_COMPACT_WINDOW 环境变量进一步压低
```

### 3.5 升级消息注入

`getUpgradeMessage('warning')` 来自 `utils/model/contextWindowUpgradeCheck.ts`：
- 检查当前用户模型设置是否为 `opus` 或 `sonnet`，并校验其是否具备 1M 上下文访问权限（`checkOpus1mAccess` / `checkSonnet1mAccess`）。
- 若满足条件，返回 `/model opus[1m]` 或 `/model sonnet[1m]` 的快捷指令字符串。
- 该字符串被拼接在各类告警文案之后，以 `·` 分隔。

### 3.6 告警抑制机制

`useCompactWarningSuppression` → `compactWarningStore`（`src/services/compact/compactWarningState.ts`）

- `suppressCompactWarning()`：在成功完成一次压缩后调用，将 store 设为 `true`。
- `clearCompactWarningSuppression()`：在新一次压缩尝试开始时调用，恢复为 `false`。
- 抑制期间 `TokenWarning` 完全隐藏，避免用户在压缩刚完成、但 token 计数尚未刷新时看到旧数据的告警。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件核心符号

| 符号 | 类型 | 职责 |
|------|------|------|
| `TokenWarning` | React 组件 | 主入口，决定渲染哪类告警文案 |
| `CollapseLabel` | React 子组件 | 订阅 context-collapse store，渲染实时折叠统计 |

### 4.2 直接调用链（上游 → 下游）

```
src/components/PromptInput/Notifications.tsx
  └── <TokenWarning tokenUsage={tokenUsage} model={mainLoopModel} />
      └── 仅在 !isBriefOnly 时渲染

src/query.ts
  └── 不直接渲染 TokenWarning，但调用 calculateTokenWarningState(...)
      └── 用于判断 isAtBlockingLimit，决定是否阻塞请求
```

### 4.3 依赖文件清单

| 文件路径 | 被引用的导出 | 作用 |
|----------|--------------|------|
| `src/services/compact/autoCompact.ts` | `calculateTokenWarningState`, `getEffectiveContextWindowSize`, `isAutoCompactEnabled` | 阈值计算、窗口大小、特性是否开启 |
| `src/services/compact/compactWarningHook.ts` | `useCompactWarningSuppression` | React Hook 订阅抑制 store |
| `src/services/compact/compactWarningState.ts` | `compactWarningStore` | 纯状态 store（被 Hook 封装） |
| `src/utils/model/contextWindowUpgradeCheck.ts` | `getUpgradeMessage` | 1M 模型升级提示文案 |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` | 非阻塞读取 GB 特性值 |
| `src/services/contextCollapse/index.js` | `getStats`, `subscribe`, `isContextCollapseEnabled` | **动态 require**，折叠引擎统计与开关 |
| `src/ink.js` | `Box`, `Text` | Ink 终端渲染基元 |

### 4.4 构建时特性开关（bun:bundle）

源码顶部 `import { feature } from 'bun:bundle'` 表明 `feature('...')` 是**编译期常量**。Bun 打包器会根据构建配置将未启用的特性分支做 Dead Code Elimination（DCE），从而：
- 在外部构建中彻底剔除 `CONTEXT_COLLAPSE`、`REACTIVE_COMPACT` 等实验代码。
- 避免实验性字符串泄露到生产包（对应 `excluded-strings.txt` 机制）。

这也解释了为何 `require('../services/contextCollapse/index.js')` 被放在 `if (feature('CONTEXT_COLLAPSE'))` 块内部——若特性关闭，该 `require` 调用连同整个模块引用都会在构建时被消除。

---

## 5. 依赖与外部交互

### 5.1 React 生态依赖

- **`useSyncExternalStore`**（来自 `react`）：
  - 在 `CollapseLabel` 中订阅 context-collapse 的自定义 store。
  - 在 `useCompactWarningSuppression` 中订阅 `compactWarningStore`。
  - 两者均利用 snapshot 机制保证 SSR/并发安全（尽管终端 UI 无真实 SSR，但符合 React 最佳实践）。

- **React Compiler（`_c`）**：
  从仓库中已编译的 `.tsx` 产物可见，该文件已被 React Compiler 处理，函数参数被改写为 `t0`，内部使用 `_c(n)` 进行自动 memoization。这意味着手动添加 `React.memo` 是冗余的。

### 5.2 Ink 终端 UI 依赖

- `<Box flexDirection="row">`：水平排列的容器。
- `<Text color="..." dimColor wrap="truncate">`：带颜色、截断换行的文本节点。
- `wrap="truncate"` 确保在窄终端中不会撑爆布局。

### 5.3 GrowthBook 特性标志交互

`TokenWarning` 使用 `getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)` 判定是否进入 Reactive-Only 模式。该函数：
- 优先读取内存缓存（`remoteEvalFeatureValues`）。
- 次优先读取磁盘缓存（`~/.claude.json` 中的 `cachedGrowthBookFeatures`）。
- **非阻塞**，即使 GrowthBook 尚未完成网络初始化也能立即返回默认值或旧缓存值。

### 5.4 Context Collapse 模块的动态耦合

```ts
const { isContextCollapseEnabled } =
  require('../services/contextCollapse/index.js') as typeof import('../services/contextCollapse/index.js');
```

**设计原因**：`autoCompact.ts` 已经导出了 `getEffectiveContextWindowSize`，而 context-collapse 模块在初始化时可能反向依赖 `autoCompact.ts`，形成循环依赖。将 `require` 延迟到运行时 + 特性分支内部，可打破初始化时的模块循环。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 `displayPercentLeft` 在 Collapse/Reactive 模式下的语义漂移

当 `reactiveOnlyMode` 或 `collapseMode` 为真时，代码会覆盖 `displayPercentLeft`：

```ts
displayPercentLeft = Math.max(0, Math.round((effectiveWindow - tokenUsage) / effectiveWindow * 100));
```

这里使用的是 `getEffectiveContextWindowSize(model)`，而**非** `calculateTokenWarningState` 返回时所用的 `threshold`。若用户开启了 Auto-Compact，`threshold` 实际上是 `effectiveWindow - 13_000`，但此处直接用 `effectiveWindow` 做分母，导致：
- 在 reactive/collapse 模式下，百分比计算基准与 normal 模式不一致。
- 用户可能看到 "87% context used" 时实际上已经触发了 collapse 的 90% commit 阈值，存在认知差。

**建议**：统一使用同一基准函数，或在注释中明确说明两种百分比定义的差异。

#### 6.1.2 `CollapseLabel` 的 snapshot 字符串协议脆弱

`"${collapsedSpans}|${stagedSpans}|..."` 这种管道符拼接的序列化方式：
- 无版本号，若未来需要新增字段（如 `retryCount`），必须同步修改订阅端与发布端的解析逻辑，否则 `map(Number)` 会产生 `NaN`。
- 任何字段顺序调整都会导致 UI 错乱。

**建议**：改为 JSON 序列化（如 `JSON.stringify({c, s, e, es, iw})`），虽然多几个字节，但可扩展性和可读性更好；或至少将解析逻辑抽成独立函数并加单元测试。

#### 6.1.3 `feature('CONTEXT_COLLAPSE')` 与 `isContextCollapseEnabled()` 双重检查

组件内和 `autoCompact.ts` 内均重复出现：
```ts
if (feature('CONTEXT_COLLAPSE')) {
  const { isContextCollapseEnabled } = require('...');
  if (isContextCollapseEnabled()) { ... }
}
```

虽然 `feature()` 是编译期常量，但 `isContextCollapseEnabled()` 是运行时函数。两处逻辑若未来出现不一致（例如一处忘记加 `feature()` 包裹），可能导致外部构建意外加载被 DCE 掉的模块。

**建议**：封装一个统一的辅助函数 `isCollapseModeActive()`，将 `feature()` 判断和运行时 `require` 封装在一起，减少重复和出错概率。

#### 6.1.4 `getUpgradeMessage` 的缓存与模型切换竞态

`getUpgradeMessage('warning')` 在 `TokenWarning` 的 React Compiler memo cache 中被缓存（`$[4]`）。由于 `getUpgradeMessage` 内部读取的是用户配置（`getUserSpecifiedModelSetting`）和权限检查，这些值在单次会话中通常不变，因此缓存是安全的。但如果用户在会话中途通过 `/model` 切换模型，React Compiler 的缓存键未包含模型信息，可能导致升级提示延迟一帧更新。

**建议**：显式将 `model` 作为依赖传入 `getUpgradeMessage`，或在组件内将 `upgradeMessage` 的计算依赖声明为 `[model]`，确保 Compiler 能正确失效缓存。

### 6.2 边界行为

| 场景 | 行为 |
|------|------|
| `tokenUsage = 0` | 未达 warning threshold，组件返回 `null`，Footer 不显示任何容量提示。 |
| 成功压缩后 | `suppressCompactWarning` 为 `true`，组件立即隐藏，直到下次压缩尝试前恢复。 |
| 窄终端 | 所有 `Text` 均带 `wrap="truncate"`，超长文案会被截断，不会换行撑高 Footer。 |
| `CONTEXT_COLLAPSE` 关闭的构建 | `require` 分支被 DCE，`CollapseLabel` 永远不会出现在产物中。 |
| `tengu_cobalt_raccoon` 未缓存 | `getFeatureValue_CACHED_MAY_BE_STALE` 返回默认值 `false`，组件走 normal 逻辑。 |

### 6.3 改进建议

1. **统一百分比语义**：将 `displayPercentLeft` 的计算委托给 `calculateTokenWarningState` 的一个新重载/参数，确保三种模式下的百分比基于同一 `threshold` 定义，避免用户困惑。

2. **Snapshot 结构化**：将 `CollapseLabel` 的 snapshot 从管道字符串迁移为小型 JSON 对象，并配套类型守卫函数，提升可维护性。

3. **抽离模式判定逻辑**：当前 `TokenWarning` 内部混入了大量特性开关判定（`reactiveOnlyMode`、`collapseMode`）。建议将这些判定提取到 `services/compact/displayMode.ts` 中，返回一个联合类型 `DisplayMode = 'normal' | 'reactive' | 'collapse'`，让组件专注于渲染，降低认知负担。

4. **增加单元测试覆盖**：该组件涉及多模式分支、颜色切换、upgradeMessage 拼接，建议补充 Ink 的渲染测试（使用 `ink-testing-library`），验证：
   - 各 threshold 下的文案与颜色。
   - `suppressWarning` 为 `true` 时返回 `null`。
   - `CollapseLabel` 对错误态和空闲态的渲染。

5. **文档化动态 require 的循环依赖原因**：在代码注释中补充 `autoCompact.ts ↔ contextCollapse/index.js` 循环依赖的具体说明，方便后续维护者理解为何不能改为静态 `import`。

---

*文档结束*
