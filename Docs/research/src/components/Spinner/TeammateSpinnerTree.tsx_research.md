# `src/components/Spinner/TeammateSpinnerTree.tsx` 研究

本研究仅基于当前仓库可见的代码、配置类型、hooks、任务实现、调用链与测试文件检索结果完成；未把 `README`、`Docs`、`docs`、其他 Markdown 文档作为研究输入。

## 场景与职责

`TeammateSpinnerTree` 是 teammate 多 agent 协作场景下的状态树容器组件，负责将 leader（当前用户会话）和所有运行中的 in-process teammate 组织成一个可选择、可折叠、可查看的终端树形列表。其核心职责包括：

1. ** teammate 聚合与排序**：从全局 `AppState.tasks` 中提取所有运行中的 teammate，并按 `agentName` 字母顺序排序。
2. ** leader 状态行渲染**：在 teammate 列表上方渲染 `team-lead` 行，展示 leader 的当前动词、token 数、idle 状态或选择提示。
3. ** teammate 行批量渲染**：为每个运行中的 teammate 渲染 `TeammateSpinnerLine`，注入正确的选择/高亮/预览状态。
4. ** 折叠控制**：在选择模式下额外渲染一个 `hide` 行，允许用户通过 Enter 键折叠整个 teammate 树。
5. ** 空状态短路**：当没有运行中的 teammate 时，直接返回 `null`，避免渲染空容器。

该组件是 `SpinnerWithVerbInner` 在 `showSpinnerTree` 状态下的主要子树之一，也是 `useBackgroundTaskNavigation` 键盘导航协议的视觉呈现层。

## 功能点目的

- **树形导航视觉化**：通过 box-drawing 字符（`╒═`/`┌─`、`╘═`/`└─`、`╞═`/`├─`）区分 leader 和 teammate，高亮当前选中或 foregrounded 的项。
- **索引语义统一**：`selectedIndex === -1` 代表 leader，`0..N-1` 代表 teammate，`N` 代表 hide 行。该语义与 `useBackgroundTaskNavigation.ts` 的 `stepTeammateSelection` 完全一致。
- **选择提示动态显示**：leader 或 teammate 被高亮时显示 `TEAMMATE_SELECT_HINT`（"shift + ↑/↓ to select"），被选中但未 foregrounded 时显示 "enter to view"。
- **leader 状态透传**：接收来自父组件 `SpinnerWithVerbInner` 的 `leaderVerb`、`leaderTokenCount`、`leaderIdleText`，确保 leader 行与主 spinner 状态同步。
- **React Compiler 缓存优化**：代码已被 React Compiler 转换，使用 `_c(N)` 缓存数组减少重渲染开销。

## 具体技术实现（关键流程/数据结构/协议/命令）

### Props 定义

```typescript
type Props = {
  selectedIndex?: number;
  isInSelectionMode?: boolean;
  allIdle?: boolean;
  leaderVerb?: string;
  leaderTokenCount?: number;
  leaderIdleText?: string;
};
```

### teammate 获取与排序

`src/components/Spinner/TeammateSpinnerTree.tsx:44`

```typescript
const teammateTasks = getRunningTeammatesSorted(tasks);
```

`getRunningTeammatesSorted` 来自 `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:123-124`，其实现为：

```typescript
export function getRunningTeammatesSorted(tasks: Record<string, TaskStateBase>): InProcessTeammateTaskState[] {
  return getAllInProcessTeammateTasks(tasks)
    .filter(t => t.status === 'running')
    .sort((a, b) => a.identity.agentName.localeCompare(b.identity.agentName));
}
```

### 空状态短路

`src/components/Spinner/TeammateSpinnerTree.tsx:45-48`

若 `teammateTasks.length === 0`，通过 `Symbol.for("react.early_return_sentinel")` 机制提前返回 `null`，不渲染任何 DOM。

### leader 状态计算

`src/components/Spinner/TeammateSpinnerTree.tsx:49-51`

```typescript
const isLeaderForegrounded = viewingAgentTaskId === undefined;
const isLeaderSelected = isInSelectionMode && selectedIndex === -1;
const isLeaderHighlighted = isLeaderForegrounded || isLeaderSelected;
```

- `viewingAgentTaskId === undefined` 表示当前没有查看任何 teammate 的 transcript，即 leader 处于 foreground 状态。
- `selectedIndex === -1` 表示键盘导航当前选中的是 leader。

### hide 行选择状态

`src/components/Spinner/TeammateSpinnerTree.tsx:52`

```typescript
const isHideSelected = isInSelectionMode === true && selectedIndex === teammateTasks.length;
```

当选择模式开启且选中索引等于 teammate 数量时，表示 "hide" 行被选中。

### leader 行渲染细节

`src/components/Spinner/TeammateSpinnerTree.tsx:54-148`

leader 行的 JSX 结构：

```
<Box paddingLeft={3}>
  <Text color="suggestion" bold={isLeaderHighlighted}>{pointer}</Text>
  <Text dimColor={!isLeaderHighlighted} bold={isLeaderHighlighted}>{treeChar} </Text>
  <Text bold={isLeaderHighlighted} color={isLeaderSelected ? "suggestion" : "cyan_FOR_SUBAGENTS_ONLY"}>team-lead</Text>
  {leaderVerb && !isLeaderForegrounded && <Text dimColor>: {leaderVerb}…</Text>}
  {!leaderVerb && leaderIdleText && !isLeaderForegrounded && <Text dimColor>: {leaderIdleText}</Text>}
  {leaderTokenCount > 0 && <Text dimColor> · {formatNumber(leaderTokenCount)} tokens</Text>}
  {isLeaderHighlighted && <Text dimColor> · {TEAMMATE_SELECT_HINT}</Text>}
  {isLeaderSelected && !isLeaderForegrounded && <Text dimColor> · enter to view</Text>}
</Box>
```

注意：
- `treeChar` 为 `╒═`（高亮）或 `┌─`（普通）。
- `cyan_FOR_SUBAGENTS_ONLY` 是一个主题色键名，用于标识 leader 的默认颜色。
- `leaderVerb` 只在 `!isLeaderForegrounded` 时显示，避免 leader 处于 foreground 时与主 spinner 重复显示动词。

### teammate 行映射

`src/components/Spinner/TeammateSpinnerTree.tsx:149`

```typescript
teammateTasks.map((teammate, index) => (
  <TeammateSpinnerLine
    key={teammate.id}
    teammate={teammate}
    isLast={!isInSelectionMode && index === teammateTasks.length - 1}
    isSelected={isInSelectionMode && selectedIndex === index}
    isForegrounded={viewingAgentTaskId === teammate.id}
    allIdle={allIdle}
    showPreview={showTeammateMessagePreview}
  />
))
```

- `isLast` 在非选择模式下由最后一个 teammate 决定，影响树形字符是 `└─` 还是 `├─`。
- `isSelected` 和 `isForegrounded` 分别对应键盘选择和当前查看状态。

### HideRow 子组件

`src/components/Spinner/TeammateSpinnerTree.tsx:212-271`

`HideRow` 是一个独立的内部组件，渲染：

```
<Box paddingLeft={3}>
  <Text color="suggestion" bold={isSelected}>{pointer}</Text>
  <Text dimColor={!isSelected} bold={isSelected}>{treeChar} </Text>
  <Text dimColor={!isSelected} bold={isSelected}>hide</Text>
  {isSelected && <Text dimColor> · enter to collapse</Text>}
</Box>
```

- `treeChar` 为 `╘═`（选中）或 `└─`（未选中）。
- 选中时提示 "enter to collapse"，与 `useBackgroundTaskNavigation.ts:211-217` 的 Enter 处理逻辑对应。

### React Compiler 缓存模式

整个文件已被 React Compiler 编译，函数体被包裹在 `_c(61)` 缓存数组中。所有 JSX 节点和中间计算结果都通过 `$[N]` 数组进行 memoization，只有依赖变化时才会重新计算对应的子树。

## 关键代码路径与文件引用

- `src/components/Spinner/TeammateSpinnerTree.tsx:10-20`
  - Props 类型定义。
- `src/components/Spinner/TeammateSpinnerTree.tsx:21-202`
  - `TeammateSpinnerTree` 主组件，含 React Compiler 缓存逻辑。
- `src/components/Spinner/TeammateSpinnerTree.tsx:44-52`
  - teammate 获取、空状态短路、leader/hide 选择状态计算。
- `src/components/Spinner/TeammateSpinnerTree.tsx:54-148`
  - leader 行渲染，含颜色、动词、token、提示的条件渲染。
- `src/components/Spinner/TeammateSpinnerTree.tsx:149`
  - `TeammateSpinnerLine` 映射调用。
- `src/components/Spinner/TeammateSpinnerTree.tsx:176-201`
  - 最终容器组装：`<Box flexDirection="column" marginTop={1}>` 包裹 leader 行、teammate 行和 hide 行。
- `src/components/Spinner/TeammateSpinnerTree.tsx:212-271`
  - `HideRow` 子组件实现。
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:123-124`
  - `getRunningTeammatesSorted` 排序规则。
- `src/hooks/useBackgroundTaskNavigation.ts:25-58`
  - `stepTeammateSelection` 的索引语义与 wrapping 逻辑。
- `src/components/Spinner.tsx:282`
  - `SpinnerWithVerbInner` 调用 `TeammateSpinnerTree` 的位置。

## 依赖与外部交互

### 直接依赖

- `figures`：选择指针字符。
- `React`。
- `../../ink.js`：`Box`, `Text`。
- `../../state/AppState.js`：`useAppState`（读取 `tasks`、`viewingAgentTaskId`、`showTeammateMessagePreview`）。
- `../../tasks/InProcessTeammateTask/InProcessTeammateTask.js`：`getRunningTeammatesSorted`。
- `../../utils/format.js`：`formatNumber`。
- `./TeammateSpinnerLine.js`：`TeammateSpinnerLine`。
- `./teammateSelectHint.js`：`TEAMMATE_SELECT_HINT`。

### 外部交互

- **父组件 `SpinnerWithVerbInner`**：注入 `selectedIndex`（来自 `selectedIPAgentIndex`）、`isInSelectionMode`（来自 `viewSelectionMode === 'selecting-agent'`）、`allIdle`、`leaderVerb`、`leaderTokenCount`、`leaderIdleText`。
- **键盘导航协议**：`useBackgroundTaskNavigation.ts` 维护 `selectedIPAgentIndex` 和 `viewSelectionMode`，并通过 `stepTeammateSelection` 实现 `Shift+↑/↓` 循环导航。`TeammateSpinnerTree` 的视觉状态必须与该 hook 的索引语义完全一致。
- **全局状态**：`viewingAgentTaskId` 决定当前是 leader foreground 还是某个 teammate foreground；`showTeammateMessagePreview` 控制 teammate 行是否显示消息预览。

## 风险、边界与改进建议

### 1. `getRunningTeammatesSorted` 的排序稳定性与索引一致性

`TeammateSpinnerTree` 和 `useBackgroundTaskNavigation.ts` 都依赖 `getRunningTeammatesSorted` 返回的数组顺序。如果该排序逻辑在未来被修改（例如改为按创建时间排序），但 `useBackgroundTaskNavigation.ts` 没有同步更新，会导致键盘导航的选中项与视觉呈现错位。

**建议**：
- 将排序逻辑与索引映射封装到一个共享 hook 或 selector 中，确保所有消费者使用同一来源。
- 在代码注释中明确标注："修改排序规则必须同步更新 useBackgroundTaskNavigation 和 PromptInput footer"。

### 2. `isLast` 在选择模式下的行为不一致

`src/components/Spinner/TeammateSpinnerTree.tsx:149` 中 `isLast={!isInSelectionMode && index === teammateTasks.length - 1}` 意味着在选择模式下所有 teammate 都使用 `├─` 树形字符，即使它是最后一个。这可能是为了避免选择模式下 `hide` 行作为实际最后一行时，最后一个 teammate 使用 `└─` 造成的视觉断裂。但该行为没有注释说明，维护者容易误改。

**建议**：
- 添加内联注释解释选择模式下 `isLast` 被强制为 `false` 的设计意图。

### 3. `cyan_FOR_SUBAGENTS_ONLY` 主题色键名的语义风险

leader 行使用了 `"cyan_FOR_SUBAGENTS_ONLY"` 作为默认颜色键名，这个键名暗示它原本是为 subagent 设计的，现在被借用来显示 leader。如果主题系统未来调整该颜色的语义或移除该键，leader 行的颜色会回退到默认色或报错。

**建议**：
- 为主题系统增加一个明确的 `"teamLead"` 或 `"leader"` 颜色键，替代这个带有历史包袱的键名。

### 4. React Compiler 缓存代码的可读性成本

编译后的代码使用大量 `_c(N)`、`$[M]`、`Symbol.for("react.early_return_sentinel")` 等模式，显著降低了人工阅读和维护的效率。虽然运行时性能更好，但调试和代码审查成本上升。

**建议**：
- 确保源码仓库保留未编译的原始 TSX（如果当前已经是编译后产物，则需要确认构建流程）。
- 在调试 teammate 树渲染问题时，优先使用 React DevTools Profiler 而不是直接阅读编译后代码。

### 5. `TeammateSpinnerTree` 与 `SpinnerWithVerbInner` 的 props 耦合

`leaderVerb`、`leaderTokenCount`、`leaderIdleText` 等 props 需要父组件预先计算并传入，这增加了父子组件之间的接口宽度。父组件需要在每次重渲染时重新评估这些值，即使 teammate 树本身没有变化。

**建议**：
- 考虑将 leader 状态计算下沉到 `TeammateSpinnerTree` 内部，通过 `useAppState` 直接读取所需状态，减少 props 传递。
- 或者使用 `React.memo` / React Compiler 确保父组件的非相关状态变化不会导致 `TeammateSpinnerTree` 重渲染。

### 6. 缺少 teammate 树交互的自动化测试

当前未发现针对 `TeammateSpinnerTree` 的测试，尤其是以下边界场景：
- teammate 数量从 0 到 1 再到 0 的切换。
- `selectedIndex` 超出当前 teammate 数量时的行为（虽然 `useBackgroundTaskNavigation` 会 clamp，但组件本身没有防御）。
- leader 和 hide 行的高亮/选择状态组合。

**建议**：
- 为 `TeammateSpinnerTree` 编写渲染测试，覆盖空状态、单 teammate、多 teammate、选择模式、hide 行选中状态。
- 测试 `getRunningTeammatesSorted` 的排序稳定性，确保与导航逻辑一致。
