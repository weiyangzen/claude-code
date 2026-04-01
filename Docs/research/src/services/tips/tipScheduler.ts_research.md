# tipScheduler.ts 研究文档

## 场景与职责

`tipScheduler.ts` 是 Claude Code 提示系统的调度模块，负责：

1. **提示选择策略**：从可用提示中选择最久未展示的提示（LRU 策略）
2. **展示入口控制**：检查用户设置，决定是否展示提示
3. **展示后处理**：记录提示展示历史并发送分析事件

该模块是提示系统的"调度器"，连接了提示注册表（`tipRegistry`）和历史记录（`tipHistory`），确保提示以合理的频率和顺序展示给用户。

## 功能点目的

### 1. `selectTipWithLongestTimeSinceShown(availableTips: Tip[]): Tip | undefined`

**目的**：从可用提示列表中选择最久未展示的提示。

**算法**：
1. 如果列表为空，返回 `undefined`
2. 如果只有一个提示，直接返回
3. 否则，计算每个提示自上次展示以来的会话数
4. 按会话数降序排序，返回第一个（最久未展示）

**策略说明**：
- 使用 LRU（Least Recently Used）策略确保提示轮换展示
- 避免同一提示频繁重复出现
- 新提示（从未展示过）会获得 `Infinity` 的优先级，优先展示

### 2. `getTipToShowOnSpinner(context?: TipContext): Promise<Tip | undefined>`

**目的**：获取当前应该展示在加载动画（spinner）上的提示。

**流程**：
1. 检查用户设置 `spinnerTipsEnabled`，如果为 `false` 返回 `undefined`
2. 调用 `getRelevantTips(context)` 获取所有相关且冷却完成的提示
3. 调用 `selectTipWithLongestTimeSinceShown` 选择最久未展示的提示

**调用时机**：
- 在 `REPL.tsx` 中，当 Claude 开始处理用户请求时调用
- 每次新的对话轮次会选择一个新提示

### 3. `recordShownTip(tip: Tip): void`

**目的**：记录提示已被展示，用于历史追踪和分析。

**操作**：
1. 调用 `recordTipShown(tip.id)` 更新本地历史记录
2. 调用 `logEvent('tengu_tip_shown', {...})` 发送分析事件

**分析数据**：
- `tipIdLength`：提示 ID（用于追踪哪种提示被展示）
- `cooldownSessions`：该提示的冷却配置（用于分析冷却策略效果）

## 具体技术实现

### 关键流程

```
REPL.tsx: pickNewSpinnerTip()
    ↓
getTipToShowOnSpinner({ theme, readFileState, bashTools })
    ↓
[检查 spinnerTipsEnabled 设置]
    ↓
getRelevantTips(context)  // tipRegistry.ts
    ├── 筛选 isRelevant() === true 的提示
    └── 筛选 getSessionsSinceLastShown >= cooldownSessions 的提示
    ↓
selectTipWithLongestTimeSinceShown(tips)
    ├── 计算每个提示的 sessionsSinceLastShown
    ├── 按 sessions 降序排序
    └── 返回最久未展示的提示
    ↓
[REPL.tsx 获取提示内容并展示]
    ↓
recordShownTip(tip)
    ├── recordTipShown(tip.id)  // 更新本地历史
    └── logEvent('tengu_tip_shown', {...})  // 发送分析
```

### 数据结构

#### 提示对象（来自 tipRegistry）

```typescript
interface Tip {
  id: string;
  content: (ctx: TipContext) => Promise<string>;
  cooldownSessions: number;
  isRelevant: (ctx?: TipContext) => Promise<boolean>;
}
```

#### 提示上下文

```typescript
interface TipContext {
  theme: Theme;                    // 当前主题，用于颜色渲染
  readFileState?: FileStateCache;  // 已读取文件状态，用于上下文相关提示
  bashTools?: Set<string>;         // 已使用的 bash 工具，用于插件推荐
}
```

### 选择算法详解

```typescript
export function selectTipWithLongestTimeSinceShown(
  availableTips: Tip[],
): Tip | undefined {
  // 边界情况处理
  if (availableTips.length === 0) return undefined
  if (availableTips.length === 1) return availableTips[0]

  // 计算每个提示的"年龄"
  const tipsWithSessions = availableTips.map(tip => ({
    tip,
    sessions: getSessionsSinceLastShown(tip.id),  // Infinity 表示从未展示
  }))

  // 按年龄降序排序，最老的在前
  tipsWithSessions.sort((a, b) => b.sessions - a.sessions)
  
  return tipsWithSessions[0]?.tip
}
```

**复杂度分析**：
- 时间复杂度：O(n log n)，主要是排序
- 空间复杂度：O(n)，创建临时数组

## 关键代码路径与文件引用

### 导出函数

| 函数 | 导出类型 | 被引用文件 |
|------|----------|------------|
| `selectTipWithLongestTimeSinceShown` | 命名导出 | 仅内部使用（可被测试导入） |
| `getTipToShowOnSpinner` | 命名导出 | `REPL.tsx` |
| `recordShownTip` | 命名导出 | `REPL.tsx` |

### 依赖关系

```
tipScheduler.ts
├── 导入: tipHistory.ts (getSessionsSinceLastShown, recordTipShown)
├── 导入: tipRegistry.ts (getRelevantTips)
├── 导入: analytics/index.js (logEvent)
├── 导入: settings.ts (getSettings_DEPRECATED)
├── 被 REPL.tsx 导入
└── 被 main.tsx 间接使用
```

### 调用链

```
REPL.tsx
├── pickNewSpinnerTip() 使用 getTipToShowOnSpinner
└── 展示提示后调用 recordShownTip
```

## 依赖与外部交互

### 与 REPL.tsx 的交互

在 `REPL.tsx` 中，提示系统与 UI 状态紧密集成：

```typescript
// REPL.tsx ~1531
const pickNewSpinnerTip = useCallback(() => {
  if (tipPickedThisTurnRef.current) return;  // 每轮只选一次
  tipPickedThisTurnRef.current = true;
  
  // 收集上下文信息
  void getTipToShowOnSpinner({
    theme,
    readFileState: readFileState.current,
    bashTools: bashTools.current
  }).then(async tip => {
    if (tip) {
      const content = await tip.content({ theme });
      setAppState(prev => ({ ...prev, spinnerTip: content }));
      recordShownTip(tip);
    }
  });
}, [setAppState, theme]);
```

### 与 Spinner 组件的交互

提示内容最终通过 `spinnerTip` prop 传递给 `SpinnerWithVerb` 组件：

```typescript
// REPL.tsx ~4587
<SpinnerWithVerb 
  mode={streamMode} 
  spinnerTip={spinnerTip}  // 来自 AppState
  // ...其他 props
/>
```

在 `Spinner.tsx` 中，提示与超时提示（如 `/clear`、 `/btw` 建议）结合：

```typescript
// Spinner.tsx ~259
const effectiveTip = contextTipsActive 
  ? undefined 
  : showClearTip && !nextTask 
    ? 'Use /clear to start fresh...'
    : showBtwTip && !nextTask 
      ? "Use /btw to ask a quick side question..."
      : spinnerTip;  // 来自 tipScheduler 的提示
```

### 与分析系统的交互

每次提示展示都会记录分析事件：

```typescript
logEvent('tengu_tip_shown', {
  tipIdLength: tip.id as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  cooldownSessions: tip.cooldownSessions,
})
```

注意：使用 `tipIdLength` 而非 `tipId` 是为了避免在分析中发送可能敏感的具体提示内容。

### 与设置系统的交互

检查用户是否禁用了提示：

```typescript
if (getSettings_DEPRECATED().spinnerTipsEnabled === false) {
  return undefined
}
```

## 风险、边界与改进建议

### 潜在风险

1. **每轮多次调用**：虽然使用了 `tipPickedThisTurnRef` 防护，但如果逻辑有 bug 可能导致同一轮次选择多个提示

2. **异步竞态**：`getTipToShowOnSpinner` 是异步的，如果用户在提示选择过程中取消请求，可能导致状态不一致

3. **空提示处理**：当没有可用提示时返回 `undefined`，调用方需要正确处理

4. **分析事件重复**：如果 `recordShownTip` 被多次调用（如 bug 或重试），会产生重复的分析事件

### 边界情况

| 场景 | 行为 |
|------|------|
| `spinnerTipsEnabled = false` | 立即返回 `undefined`，不展示提示 |
| 所有提示都在冷却中 | `getRelevantTips` 返回空数组，最终返回 `undefined` |
| 提示内容生成失败 | 依赖 `tip.content()` 的异常处理，REPL.tsx 中无显式捕获 |
| 同一轮次多次调用 | `tipPickedThisTurnRef` 防护，直接返回 |
| 新提示（从未展示） | `getSessionsSinceLastShown` 返回 `Infinity`，优先选择 |

### 改进建议

1. **提示展示频率限制**：
   ```typescript
   // 建议：添加每轮最多展示一次的全局控制
   const TIP_SHOW_INTERVAL_MS = 30000;  // 最少 30 秒展示一次新提示
   ```

2. **提示内容缓存**：
   ```typescript
   // 建议：缓存提示内容，避免重复生成
   const tipContentCache = new Map<string, string>()
   ```

3. **更好的错误处理**：
   ```typescript
   // 建议：包装提示内容生成，处理异常
   try {
     const content = await tip.content(context)
   } catch (error) {
     logForDebugging(`Failed to generate tip content: ${error}`)
     return undefined
   }
   ```

4. **提示展示确认机制**：
   ```typescript
   // 建议：确认提示真正被用户看到后才记录
   recordShownTip(tip, { confirmed: true })  // 需要 UI 反馈
   ```

5. **A/B 测试支持**：
   ```typescript
   // 建议：支持提示变体测试
   interface TipVariant {
     id: string
     content: string
     weight: number  // 展示权重
   }
   ```

6. **用户反馈机制**：
   ```typescript
   // 建议：允许用户对提示进行反馈
   logEvent('tengu_tip_feedback', {
     tipId: tip.id,
     helpful: boolean
   })
   ```

### 测试建议

1. **单元测试**：
   - `selectTipWithLongestTimeSinceShown` 的各种输入场景
   - `getTipToShowOnSpinner` 的设置检查逻辑
   - `recordShownTip` 的分析事件发送

2. **集成测试**：
   - 与 `tipRegistry` 和 `tipHistory` 的完整流程
   - 与 `REPL.tsx` 的集成，验证提示正确展示

3. **边界测试**：
   - 空提示列表
   - 所有提示冷却中
   - 设置禁用提示
   - 异步取消场景
