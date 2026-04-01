# usePromptInputPlaceholder.ts 研究文档

## 场景与职责

动态决定 PromptInput 的 placeholder 文本，按优先级返回：teammate 消息提示、队列编辑提示、或新用户示例命令。

## 功能点目的

- 查看 teammate 时显示 `Message @teammateName…`
- 命令队列有可编辑消息且提示次数不足时显示 `Press up to edit queued messages`
- 新用户首次提交前展示示例命令（如 `Try "how does <file> work?"`）
- `input !== ''` 时立即消失

## 具体技术实现

### 关键流程

1. 读取 `useCommandQueue()` 和 `useAppState(s => s.promptSuggestionEnabled)`
2. 在 `useMemo` 中按优先级计算：
   - **teammate**：`viewingAgentName` 存在时截断至 20 字符，返回 `Message @${name}…`
   - **queue hint**：`queuedCommands.some(isQueuedCommandEditable)` 且 `queuedCommandUpHintCount < 3` 时返回 `Press up to edit queued messages`
   - **example command**：`submitCount < 1` && `promptSuggestionEnabled` && `!proactiveModule?.isProactiveActive()` 时返回 `getExampleCommandFromCache()`
3. **proactive 条件导入**：`feature('PROACTIVE') || feature('KAIROS')` 开启时 `require('../../proactive/index.js')`

### 数据结构

```ts
type Props = { input: string; submitCount: number; viewingAgentName?: string }
const NUM_TIMES_QUEUE_HINT_SHOWN = 3
const MAX_TEAMMATE_NAME_LENGTH = 20
```

### 协议/命令

- `useCommandQueue()` → `src/hooks/useCommandQueue.ts`
- `isQueuedCommandEditable()` → `src/utils/messageQueueManager.ts`
- `getGlobalConfig()` → `src/utils/config.ts`
- `getExampleCommandFromCache()` → `src/utils/exampleCommands.ts`
- `proactiveModule?.isProactiveActive()` → `src/proactive/index.js`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/usePromptInputPlaceholder.ts` | 本 hook |
| `src/components/PromptInput/PromptInput.tsx` | 调用方 |
| `src/hooks/useCommandQueue.ts` | 队列订阅 |
| `src/utils/messageQueueManager.ts` | 可编辑队列判断 |
| `src/utils/exampleCommands.ts` | 示例命令缓存 |
| `src/utils/config.ts` | 全局配置 |
| `src/proactive/index.js` | proactive 状态（条件编译） |

## 依赖与外部交互

- `queuedCommandUpHintCount` 存储在全局配置中，跨会话记忆
- `getExampleCommandFromCache` 是 `memoize` 的，首次调用后固定

## 风险、边界与改进建议

1. **queue hint 计数分散**：读取在此，递增在 `PromptInput.tsx` 别处，维护成本高。建议集中管理
2. **proactive 测试复杂度**：`isProactiveActive()` 来自条件 `require`，测试需 mock `feature()` 或模块系统
3. **teammate 名称截断硬编码**：`MAX_TEAMMATE_NAME_LENGTH = 20` 未考虑终端宽度，极窄终端可能溢出
4. **测试建议**：验证 `input !== ''` 返回 undefined、teammate 覆盖 queue hint、submitCount >= 1 不显示 example、proactive active 抑制 example
