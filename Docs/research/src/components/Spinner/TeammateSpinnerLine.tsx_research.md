# `src/components/Spinner/TeammateSpinnerLine.tsx` 研究

本研究仅基于当前仓库可见的代码、配置类型、hooks、任务实现、调用链与测试文件检索结果完成；未把 `README`、`Docs`、`docs`、其他 Markdown 文档作为研究输入。

## 场景与职责

`TeammateSpinnerLine` 是队友树（teammate spinner tree）中的叶子节点组件，负责渲染单个运行中 teammate 的紧凑状态行。它的核心职责包括：

1. ** teammate 状态可视化**：将 `InProcessTeammateTaskState` 转换为一行带树形前缀的终端文本，展示 teammate 当前是活跃、空闲、等待计划审批还是正在停止。
2. **响应式布局适配**：根据终端宽度动态决定是否显示 agent 名称、工具使用统计、选择提示和操作提示，确保在窄终端下不溢出。
3. **消息预览提取**：当全局开启 `showTeammateMessagePreview` 时，从 teammate 最近的消息历史中抽取最多 3 行内容作为预览，帮助用户在不切换视图的情况下快速了解 teammate 在做什么。
4. **时间显示管理**：区分"当前空闲了多久"和"全员空闲时本轮工作了多久"，避免在 all-idle 状态下时间计数继续跳动。

该组件被 `TeammateSpinnerTree` 按运行中 teammate 的排序结果逐行调用，是 teammate 多 agent 协作 UI 的最小渲染单元。

## 功能点目的

- **树形视觉层级**：通过 `├─`/`└─`（未高亮）或 `╞═`/`╘═`（高亮）的 box-drawing 字符，让 teammate 列表在终端中呈现清晰的树状结构。
- **选择状态反馈**：`isSelected` 时显示 `figures.pointer` 指针，`isForegrounded` 时显示高亮树形字符，两者共同构成 teammate 导航的视觉反馈。
- **渐进式信息折叠**：宽终端（≥80 列）显示完整信息；中等宽度（60-80 列）隐藏提示；窄终端（<60 列）隐藏 agent 名称，只保留核心状态文本。
- **最近活动摘要**：优先展示 `recentActivities` 的折叠摘要，其次是 `lastActivity.activityDescription`，最后回退到随机 spinner verb，确保状态行始终有可读文案。
- **消息预览**：从 `messages` 数组中逆向遍历，提取 `text` 内容块的最后非空行或 `tool_use` 块的描述字段，最多 3 行，按阅读顺序排列。

## 具体技术实现（关键流程/数据结构/协议/命令）

### Props 定义

```typescript
type Props = {
  teammate: InProcessTeammateTaskState;
  isLast: boolean;
  isSelected?: boolean;
  isForegrounded?: boolean;
  allIdle?: boolean;
  showPreview?: boolean;
};
```

### 消息预览提取算法（`getMessagePreview`）

`src/components/Spinner/TeammateSpinnerLine.tsx:29-71`

1. 从 `messages` 数组末尾开始逆向遍历，最多处理到满足 3 行预览为止。
2. 只处理 `type === 'user'` 或 `type === 'assistant'` 的消息。
3. 对每个消息的 `content` 块：
   - 若块类型为 `tool_use`，优先提取 `input.description` > `input.prompt` > `input.command` > `input.query` > `input.pattern`，取第一行，并用 `truncateToWidth` 截断到 80 字符。
   - 若块类型为 `text`，按 `\n` 分割，过滤空行，从文本末尾逆向取行，同样截断到 80 字符。
4. 收集完成后 `reverse()`，使最旧的一行在前，符合阅读顺序。

### 时间显示逻辑

`src/components/Spinner/TeammateSpinnerLine.tsx:89-118`

- `idleStartRef`：记录 teammate 进入 `isIdle` 状态的时间戳，用于显示 "Idle for X"。
- `frozenDurationRef`：当 `allIdle` 为真且首次检测到时，冻结当前 teammate 的实际工作时长（`Date.now() - startTime - totalPausedMs`），用于显示 "<pastTenseVerb> for X"。
- 离开 `allIdle` 状态时重置 `frozenDurationRef`。
- `useElapsedTime(idleStartRef.current ?? Date.now(), teammate.isIdle && !allIdle)` 只在非 all-idle 的空闲状态下计时。

### 响应式布局计算

`src/components/Spinner/TeammateSpinnerLine.tsx:120-157`

固定前缀宽度为 8（`paddingLeft(3)` + `pointer(1)` + `space(1)` + `treeChar(2)` + `space(1)`）。

- `showName`：终端宽度 ≥60 且剩余空间 ≥25 时显示 `@agentName`。
- `showViewHint`：仅当 `isSelected && !isForegrounded` 且空间充足时显示 "enter to view"。
- `showSelectHint`：当 `isHighlighted`（被选中或 foregrounded）且空间充足时显示 `TEAMMATE_SELECT_HINT`（"shift + ↑/↓ to select"）。
- `showStats`：空间足够时始终显示工具使用次数和 token 数（`toolUseCount` / `tokenCount`）。
- `activityMaxWidth`：剩余宽度减去所有 extras 的占用，确保主状态文本不会溢出。

### 状态渲染优先级（`renderStatus`）

`src/components/Spinner/TeammateSpinnerLine.tsx:172-195`

按以下优先级返回状态节点：

1. `shutdownRequested` → `[stopping]`（dimColor）
2. `awaitingPlanApproval` → `[awaiting approval]`（warning 色）
3. `isIdle && allIdle` → `<pastTenseVerb> for <displayTime>`（dimColor）
4. `isIdle && !allIdle` → `Idle for <idleElapsedTime>`（dimColor）
5. `isHighlighted` → `null`（高亮时主 spinner 已在上方显示动词，此处省略避免重复）
6. 默认 → `activityText…`（dimColor，若 activityText 不以 "…" 结尾则自动追加）

### 最终 JSX 结构

`src/components/Spinner/TeammateSpinnerLine.tsx:202-231`

```
<Box flexDirection="column">
  <Box paddingLeft={3}>
    <Text>{pointer}</Text>
    <Text>{treeChar}</Text>
    <Text>{@name}</Text>
    <Text>: </Text>
    {renderStatus()}
    <Text>{stats}</Text>
    <Text>{hints}</Text>
  </Box>
  {previewLines.map(line => <Box paddingLeft={3}><Text>{previewTreeChar}</Text><Text>{line}</Text></Box>)}
</Box>
```

## 关键代码路径与文件引用

- `src/components/Spinner/TeammateSpinnerLine.tsx:16-23`
  - Props 类型定义。
- `src/components/Spinner/TeammateSpinnerLine.tsx:29-71`
  - `getMessagePreview`：消息预览提取函数。
- `src/components/Spinner/TeammateSpinnerLine.tsx:72-79`
  - 组件入口，用 `useState` 初始化随机 verb（保证重渲染稳定）。
- `src/components/Spinner/TeammateSpinnerLine.tsx:80-83`
  - `isHighlighted` 与 `treeChar` 选择逻辑。
- `src/components/Spinner/TeammateSpinnerLine.tsx:89-118`
  - idle 时间追踪与 frozen duration 逻辑。
- `src/components/Spinner/TeammateSpinnerLine.tsx:127-130`
  - 从 `teammate.progress` 读取 `toolUseCount` 和 `tokenCount`。
- `src/components/Spinner/TeammateSpinnerLine.tsx:137-157`
  - 响应式宽度门控与 `activityMaxWidth` 计算。
- `src/components/Spinner/TeammateSpinnerLine.tsx:159-169`
  - `activityText` 优先级：recentActivities 摘要 > lastActivity > randomVerb。
- `src/components/Spinner/TeammateSpinnerLine.tsx:172-195`
  - `renderStatus` 的六层优先级。
- `src/components/Spinner/TeammateSpinnerLine.tsx:197-230`
  - 主行与预览行渲染。
- `src/tasks/InProcessTeammateTask/types.ts:22-76`
  - `InProcessTeammateTaskState` 的完整类型定义。
- `src/components/Spinner/TeammateSpinnerTree.tsx:149`
  - `TeammateSpinnerTree` 调用 `TeammateSpinnerLine` 的位置。

## 依赖与外部交互

### 直接依赖

- `figures`：用于 `figures.pointer` 选择指针。
- `lodash-es/sample`：初始化随机 verb。
- `React`（`useRef`, `useState`）。
- `../../constants/spinnerVerbs.js`：`getSpinnerVerbs()`。
- `../../constants/turnCompletionVerbs.js`：`TURN_COMPLETION_VERBS`。
- `../../hooks/useElapsedTime.js`：空闲时间计时。
- `../../hooks/useTerminalSize.js`：获取终端宽度。
- `../../ink/stringWidth.js`：计算字符串显示宽度。
- `../../ink.js`：`Box`, `Text`。
- `../../tasks/InProcessTeammateTask/types.js`：`InProcessTeammateTaskState` 类型。
- `../../utils/collapseReadSearch.js`：`summarizeRecentActivities()`。
- `../../utils/format.js`：`formatDuration`, `formatNumber`, `truncateToWidth`。
- `../../utils/ink.js`：`toInkColor()`。
- `./teammateSelectHint.js`：`TEAMMATE_SELECT_HINT`。

### 外部交互

- **被 `TeammateSpinnerTree` 调用**：接收排序后的 teammate 对象，按索引决定 `isLast`、`isSelected`、`isForegrounded`。
- **依赖全局 AppState**：`showPreview` 来自 `useAppState(s => s.showTeammateMessagePreview)`，由 `TeammateSpinnerTree` 注入。
- **键盘导航协议**：`isSelected` 和 `isForegrounded` 的语义由 `useBackgroundTaskNavigation.ts` 维护，用户通过 `Shift+↑/↓` 切换，`Enter` 进入视图，`f` 快速查看，`k` kill teammate。

## 风险、边界与改进建议

### 1. `getMessagePreview` 的遍历复杂度与消息类型假设

`getMessagePreview` 对 `messages` 做嵌套循环遍历，虽然上限是 3 行，但如果 `messages` 数组很长且末尾消息都没有 `user`/`assistant` 类型，会遍历大量数据。当前实现假设消息类型只有 `user` 和 `assistant` 会携带内容，若未来引入其他消息类型（如 `system`），预览可能为空。

**建议**：
- 考虑在消息结构变更时同步更新 `getMessagePreview` 的类型守卫。
- 若消息数组可能非常大，可限制外层循环的扫描深度。

### 2. `idleStartRef` 的条件赋值在 render 阶段执行

`src/components/Spinner/TeammateSpinnerLine.tsx:95-104` 在组件函数体中直接对 ref 赋值（`idleStartRef.current = Date.now()`），这在 React 的 render 阶段修改了 ref。虽然 ref 修改不会触发重渲染，但在并发特性或严格模式下可能导致非确定性行为。

**建议**：
- 将 idle 状态变化检测封装到 `useEffect` 中，或改用 `useMemo` + `useRef` 的组合模式，确保时间戳记录与渲染阶段解耦。

### 3. `allIdle` 与 `frozenDurationRef` 的同步假设

`frozenDurationRef` 的冻结逻辑依赖 `allIdle` prop 的稳定性。如果 `allIdle` 在极短时间内反复切换（如 teammate 状态抖动），`frozenDurationRef` 会被反复重置和重新计算，导致显示时间跳动。

**建议**：
- 为 `allIdle` 的切换增加防抖或最小保持时长，避免视觉抖动。

### 4. 消息预览的内存与隐私边界

预览功能从 `teammate.messages` 读取内容，该数组已被 `TEAMMATE_MESSAGES_UI_CAP = 50` 限制长度，但预览仍然暴露了 teammate 的最近对话片段。在共享屏幕或录屏场景下，这可能泄露敏感信息。

**建议**：
- 评估是否需要在设置中增加 "隐藏 teammate 消息预览" 的选项（虽然已有全局开关，但可考虑按 teammate 粒度控制）。
- 对 `tool_use` 的 `input` 字段做更积极的脱敏处理，避免暴露原始命令或查询内容。

### 5. `activityText` 的截断可能截断在 grapheme 中间

`truncateToWidth(activityText, activityMaxWidth)` 基于显示宽度截断，但如果 `activityText` 包含组合字符或 emoji，截断结果可能在视觉上不完整。虽然 `truncateToWidth` 内部可能已处理，但值得确认。

**建议**：
- 检查 `truncateToWidth` 是否使用 `getGraphemeSegmenter`（`GlimmerMessage` 已使用类似技术），若未使用，建议统一升级。

### 6. 缺少直接单元测试

当前仓库中未发现针对 `TeammateSpinnerLine` 的单元测试，尤其是 `getMessagePreview` 的多种消息结构、`renderStatus` 的优先级分支、响应式布局的宽度计算等逻辑都缺乏自动化验证。

**建议**：
- 为 `getMessagePreview` 编写纯函数测试，覆盖 text 块、tool_use 块、混合块、空消息等场景。
- 为响应式布局计算逻辑提取纯函数并测试不同终端宽度下的显示决策。
