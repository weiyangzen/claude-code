# PluginHintMenu.tsx 深度研究文档

> 研究对象：`src/components/ClaudeCodeHint/PluginHintMenu.tsx`  
> 研究范围：代码实现、调用链、依赖模块、配置持久化、数据协议与边界行为  
> 执行时间：2026-04-01

---

## 1. 场景与职责

`PluginHintMenu` 是 Claude Code 终端 UI 中用于展示**插件安装推荐弹窗**的 React 组件。它的核心职责是：

- **可视化插件推荐**：当底层 Shell 工具（Bash/PowerShell）扫描到 CLI/SDK 输出到 stderr 的 `<claude-code-hint />` 标签时，向用户弹出一个模态选择菜单，询问是否安装推荐的插件。
- **收集用户决策**：提供 "Yes" / "No" / "No, and don't show plugin installation hints again" 三种选项，并将结果回调给上层。
- **自动兜底**：若用户 30 秒内未操作，自动以 "no" 关闭弹窗，避免阻塞会话。

该组件属于 **Claude Code Hint 协议** 的前端呈现层，该协议是 CLIs/SDKs 与 Claude Code 之间的一个零 token 侧信道（side channel）—— hint 标签在到达模型前会被剥离，仅用于触发 UI 提示。

---

## 2. 功能点目的

### 2.1 弹窗内容展示
- 显示触发该提示的原始命令（`sourceCommand`），帮助用户识别是哪个工具在推荐插件。
- 显示插件名称（`pluginName`）、所属市场（`marketplaceName`）以及可选描述（`pluginDescription`）。

### 2.2 三态用户选择
| 选项值 | 用户意图 | 系统行为 |
|--------|----------|----------|
| `yes` | 同意安装 | 调用插件安装流程，从指定 marketplace 下载并缓存插件 |
| `no` | 拒绝安装 | 仅记录该插件已展示过（show-once），不再提示 |
| `disable` | 永久关闭 | 在全局配置中写入 `disabled: true`，此后所有 plugin hint 不再出现 |

### 2.3 自动关闭机制
- 常量 `AUTO_DISMISS_MS = 30_000`。
- 组件挂载后启动 `setTimeout`，超时自动调用 `onResponse('no')`。
- 组件卸载时清理 timeout，防止内存泄漏或回调穿透。

---

## 3. 具体技术实现

### 3.1 组件接口（Props）

```tsx
type Props = {
  pluginName: string;
  pluginDescription?: string;
  marketplaceName: string;
  sourceCommand: string;
  onResponse: (response: 'yes' | 'no' | 'disable') => void;
};
```

- 所有字段均为**纯数据**，无复杂对象引用，便于序列化和 memoization。
- `onResponse` 是单一出口回调，组件内部不直接触发副作用（安装、配置写入等），职责边界清晰。

### 3.2 自动关闭实现细节

```tsx
const onResponseRef = React.useRef(onResponse);
onResponseRef.current = onResponse;

React.useEffect(() => {
  const timeoutId = setTimeout(
    ref => ref.current('no'),
    AUTO_DISMISS_MS,
    onResponseRef
  );
  return () => clearTimeout(timeoutId);
}, []);
```

- 使用 ref 传递回调，避免将 `onResponse` 放入 effect deps 导致 timeout 重置。
- timeout 回调参数直接接收 ref，而非闭包捕获，保证超时触发时一定调用最新回调。

### 3.3 选项渲染

```tsx
const options = [
  {
    label: <Text>Yes, install <Text bold>{pluginName}</Text></Text>,
    value: 'yes'
  },
  { label: 'No', value: 'no' },
  {
    label: "No, and don't show plugin installation hints again",
    value: 'disable'
  }
];
```

- 选项通过 `Select` 组件渲染，`onCancel` 映射为 `onResponse('no')`，与 ESC/取消行为一致。

### 3.4 整体 JSX 结构

```tsx
<PermissionDialog title="Plugin Recommendation">
  <Box flexDirection="column" paddingX={2} paddingY={1}>
    {/* 说明文本 + 插件信息 */}
    <Select options={options} onChange={onSelect} onCancel={() => onResponse('no')} />
  </Box>
</PermissionDialog>
```

- 外层 `PermissionDialog` 提供统一的边框样式（`borderColor="permission"`）和标题栏。
- 内层 `Box` 控制内边距与垂直布局。

---

## 4. 关键代码路径与文件引用

### 4.1 调用链（从 Shell 输出到 UI 弹窗）

```
CLI/SDK stderr
    ↓
<claude-code-hint v="1" type="plugin" value="name@marketplace" />
    ↓
[src/tools/BashTool/BashTool.tsx]
[src/tools/PowerShellTool/PowerShellTool.tsx]
    ├── extractClaudeCodeHints(output, command)  // 扫描并剥离标签
    └── maybeRecordPluginHint(hint)              // 同步门控，决定是否入队
            ↓
[src/utils/claudeCodeHints.ts]
    ├── pending-hint store (single-slot, useSyncExternalStore)
    └── setPendingHint(hint)                     // 写入全局单槽
            ↓
[src/hooks/useClaudeCodeHintRecommendation.tsx]
    ├── useSyncExternalStore(subscribeToPendingHint, ...)
    ├── resolvePluginHint(hint)                  // 异步查 marketplace 缓存
    └── 设置 recommendation 状态
            ↓
[src/screens/REPL.tsx]
    ├── useClaudeCodeHintRecommendation()        // 获取 {recommendation, handleResponse}
    ├── getFocusedInputDialog()                  // 优先级调度
    └── focusedInputDialog === 'plugin-hint'
            ↓
[src/components/ClaudeCodeHint/PluginHintMenu.tsx]
    └── 渲染弹窗，收集用户选择
```

### 4.2 核心文件清单

| 文件路径 | 作用 |
|----------|------|
| `src/components/ClaudeCodeHint/PluginHintMenu.tsx` | **目标组件**：弹窗 UI 与自动关闭逻辑 |
| `src/screens/REPL.tsx` | 调用方：优先级调度、条件渲染、生命周期管理 |
| `src/hooks/useClaudeCodeHintRecommendation.tsx` | Hook：连接 hint store 与 UI，处理安装/禁用/分析 |
| `src/hooks/usePluginRecommendationBase.tsx` | 基础状态机：gate chain、async guard、通知 JSX |
| `src/utils/claudeCodeHints.ts` | Hint 协议解析器 + pending-hint 单槽 store |
| `src/utils/plugins/hintRecommendation.ts` | 插件推荐业务逻辑：门控、解析、show-once 记录 |
| `src/utils/plugins/pluginInstallationHelpers.ts` | 插件安装核心：`installPluginFromMarketplace` |
| `src/tools/BashTool/BashTool.tsx` | Bash 工具：扫描输出、触发 hint |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | PowerShell 工具：扫描输出、触发 hint |
| `src/utils/config.ts` | 全局配置定义，包含 `claudeCodeHints` 持久化结构 |
| `src/utils/signal.ts` | 轻量信号原语，用于 pending hint store 的订阅/通知 |
| `src/components/permissions/PermissionDialog.tsx` | 通用权限/提示对话框容器 |
| `src/components/CustomSelect/select.tsx` | 通用选择器组件，支持多种布局 |

---

## 5. 依赖与外部交互

### 5.1 UI 依赖

- **`PermissionDialog`** (`src/components/permissions/PermissionDialog.tsx`)
  - 提供带主题色边框的对话框容器，复用了权限请求的标题栏组件 `PermissionRequestTitle`。
- **`Select`** (`src/components/CustomSelect/select.tsx`)
  - 高度可定制的选择器，支持 `compact`/`expanded`/`compact-vertical` 布局。`PluginHintMenu` 使用默认的 `compact` 布局。
- **`Box`, `Text`** (`src/ink.js`)
  - Ink 渲染原语，用于终端 UI 布局。

### 5.2 数据与状态依赖

- **`useClaudeCodeHintRecommendation`**
  - 订阅 `claudeCodeHints.ts` 中的 pending-hint store。
  - 当 `pendingHint` 变化时，通过 `useEffect` 调用 `tryResolve`，异步解析 marketplace 数据。
  - 解析成功后设置 `recommendation`，REPL 检测到后渲染 `PluginHintMenu`。

- **`getFocusedInputDialog` (REPL.tsx 内部函数)**
  - 决定当前哪个模态对话框获得焦点。`plugin-hint` 的优先级位于 `lsp-recommendation` 之后、`desktop-upsell` 之前。
  - 受 `allowDialogsWithAnimation` 条件控制，仅在动画允许时展示。

### 5.3 配置持久化

- **全局配置键**：`GlobalConfig.claudeCodeHints`
  - `plugin?: string[]` — 已展示过的插件 ID 列表（show-once 语义）。
  - `disabled?: boolean` — 用户选择 disable 后永久关闭。
- **上限保护**：`MAX_SHOWN_PLUGINS = 100`，超过后不再记录也不再提示，防止配置无限增长。

### 5.4 分析（Analytics）

- `useClaudeCodeHintRecommendation.tsx` 中通过 `logEvent` 上报：
  - `tengu_plugin_hint_detected` — hint 被检测到并解析 marketplace 结果时。
  - `tengu_plugin_hint_response` — 用户在 `PluginHintMenu` 中做出选择时（携带 `plugin_name`, `marketplace_name`, `response`）。
- `pluginInstallationHelpers.ts` 中安装成功后上报 `tengu_plugin_installed`，`trigger` 字段为 `'hint'`。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

1. **单槽覆盖（Single-slot overwrite）**
   - `claudeCodeHints.ts` 中的 pending-hint store 只有一个槽位。如果多个命令在短时间内连续发射 hint，后写入的会覆盖前一个，用户只能看到最后一个。
   - *缓解*：设计如此，每个会话最多只展示一次 hint 弹窗（`shownThisSession` 标志），因此覆盖问题在实际场景中影响有限。

2. **Marketplace 缓存未命中导致静默丢弃**
   - `resolvePluginHint` 是异步的，如果插件不在 marketplace 缓存中，hint 会被静默丢弃，用户无感知。
   - *风险*：网络或缓存问题可能导致本应有效的推荐被忽略。

3. **自动关闭的默认行为一致性**
   - 30 秒超时默认选择 `'no'`，这与 ESC/取消行为一致，但用户可能在未阅读的情况下错过推荐。
   - *注意*：这是有意设计，避免阻塞会话流程。

4. **远程模式下的 Gate 绕过**
   - `usePluginRecommendationBase` 中检查了 `getIsRemoteMode()`，远程模式下不展示推荐。但 shell 工具中的 `extractClaudeCodeHints` 和 `maybeRecordPluginHint` 仍会在主线程执行，只是最终不会渲染 UI。
   - *风险*：低，因为 `shownThisSession` 和 `triedThisSession` 的副作用不会导致实际安装。

### 6.2 边界行为

| 边界条件 | 行为 |
|----------|------|
| 同一插件多次触发 | `triedThisSession` Set 去重；全局 `plugin[]` 数组 show-once |
| 用户选择 `disable` | 全局配置写入 `disabled: true`，此后所有 plugin hint 永久静默 |
| 配置数组达到 100 条 | `MAX_SHOWN_PLUGINS` 截断，不再记录新插件，hint 系统失效 |
| 非官方 marketplace | `isOfficialMarketplaceName` 硬编码过滤，v1 仅支持官方市场 |
| 插件已安装 | `isPluginInstalled` 门控直接丢弃，不展示弹窗 |
| 策略阻止 | `isPluginBlockedByPolicy` 门控直接丢弃 |
| 子代理输出 | `isMainThread` 检查确保仅主线程记录 hint，子代理输出被剥离但不触发弹窗 |

### 6.3 改进建议

1. **增加未命中 marketplace 缓存的调试可见性**
   - 当前仅通过 `logForDebugging` 输出。建议在开发/verbose 模式下向用户展示一条轻量通知（notification），说明某个 hint 因缓存缺失被跳过，便于排查。

2. **考虑将单槽 store 扩展为小型队列**
   - 如果未来需要支持一次会话中多个 hint（例如多步骤构建脚本），可将单槽扩展为容量受限的队列（如最多 3 个），按 FIFO 顺序展示。

3. **统一自动关闭时长配置化**
   - `AUTO_DISMISS_MS` 当前是硬编码的 30 秒。可考虑从 `GlobalConfig` 或环境变量读取，方便不同使用场景（如演示模式、无障碍需求）调整。

4. **增强 `PluginHintMenu` 的键盘可访问性**
   - 当前依赖 `Select` 组件的默认键盘行为。可考虑在弹窗出现时自动朗读插件名称和来源命令，提升屏幕阅读器体验。

5. **提取 `PluginHintMenu` 的纯逻辑到 headless hook**
   - 目前 `useClaudeCodeHintRecommendation` 已经承担了大部分逻辑，但 `PluginHintMenu` 仍包含 `options` 的 JSX 定义。若未来需要在其他界面（如 Web UI）复用相同选项逻辑，可将选项生成和 `onSelect` 映射进一步抽象为 hook。

---

## 7. 附录：关键类型定义

```ts
// src/components/ClaudeCodeHint/PluginHintMenu.tsx
type Props = {
  pluginName: string;
  pluginDescription?: string;
  marketplaceName: string;
  sourceCommand: string;
  onResponse: (response: 'yes' | 'no' | 'disable') => void;
};

// src/utils/plugins/hintRecommendation.ts
export type PluginHintRecommendation = {
  pluginId: string;
  pluginName: string;
  marketplaceName: string;
  pluginDescription?: string;
  sourceCommand: string;
};

// src/utils/claudeCodeHints.ts
export type ClaudeCodeHint = {
  v: number;
  type: 'plugin';
  value: string;        // e.g. "name@marketplace"
  sourceCommand: string;
};
```
