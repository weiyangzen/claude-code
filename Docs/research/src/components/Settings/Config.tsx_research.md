# 研究文档：src/components/Settings/Config.tsx

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 目标文件：`src/components/Settings/Config.tsx`（约 271 KB，1821 行）  
> 执行器：kimi（model=k2p5）  
> 日期：2026-04-01

---

## 1. 场景与职责

`Config.tsx` 是 Claude Code 终端应用**设置面板（Settings Dialog）**中的 **"Config" 标签页**组件。该应用基于 [Ink](https://github.com/vadimdemedes/ink)（React for terminals）构建，整个设置面板由 `Settings.tsx` 托管，内部包含 `Status`、`Config`、`Usage`、`Gates` 四个标签页。

### 1.1 核心职责

- **配置项展示与编辑**：以可搜索的列表形式展示 30+ 项用户可配置选项，涵盖主题、模型、权限模式、通知通道、输出风格、语言、IDE 集成、自动更新、队友模式（Agent Swarms）等。
- **即时持久化 + 可撤销**：绝大多数配置项在切换时**立即写入磁盘**（`global-config.json` 或 `settings.json`），但组件在挂载时会拍摄快照，用户按 `Esc` 可一键回滚所有修改。
- **子菜单导航**：对于复杂选项（如主题、模型、语言），选中后打开覆盖式子菜单（submenu），隐藏底层标签页头部（`setTabsHidden`）。
- **键盘驱动交互**：完全依赖键盘导航（↑/↓、Enter、Space、/ 搜索、Esc 取消），通过 `useKeybinding`/`useKeybindings` 接入可配置快捷键系统。

### 1.2 在调用链中的位置

```
用户执行 /config 或 /settings
  → src/commands/config/config.tsx
    → <Settings defaultTab="Config" />
      → src/components/Settings/Settings.tsx
        → <Config /> (本文件)
```

`Settings.tsx` 负责标签页容器、全局 Esc 关闭、Ctrl+C/D 退出；`Config.tsx` 负责具体的配置列表渲染与状态管理。

---

## 2. 功能点目的

### 2.1 配置项分类

文件内通过 `settingsItems: Setting[]` 数组声明所有选项，按类型分为三类：

| 类型 | 说明 | 示例 |
|------|------|------|
| `boolean` | 开关型，直接切换 true/false | Auto-compact、Thinking mode、Verbose output |
| `enum` | 枚举型，循环切换预设值 | Default permission mode、Editor mode、Diff tool |
| `managedEnum` | 由独立组件管理的枚举，值在列表中只读，Enter 打开子菜单 | Theme、Model、Output style、Language、Teammate model |

### 2.2 搜索过滤

- 默认进入搜索模式（`isSearchMode = true`），顶部显示 `SearchBox`。
- 输入字符即时过滤 `settingsItems`，匹配 `id` 或 `label/searchText`。
- 过滤后列表长度变化时，自动修正 `selectedIndex` 与 `scrollOffset`，防止选中项越界。

### 2.3 子菜单系统（SubMenu）

`SubMenu` 联合类型定义了 8 种子菜单：

```ts
type SubMenu = 'Theme' | 'Model' | 'TeammateModel' | 'ExternalIncludes' | 'OutputStyle' | 'ChannelDowngrade' | 'Language' | 'EnableAutoUpdates';
```

打开子菜单时：
1. `setShowSubmenu(...)` 切换渲染分支；
2. `setTabsHidden(true)` 隐藏 `Settings.tsx` 的标签页头部，避免子菜单与标签页快捷键冲突；
3. 子菜单通过 `onCancel`/`onComplete` 回调关闭并恢复 `tabsHidden`。

### 2.4 快照与回滚（Revert on Escape）

由于配置项**立即写盘**，取消操作需要主动回写旧值。组件在挂载时拍摄多份快照：

- `initialConfig`：`getGlobalConfig()` 快照（全局配置）。
- `initialSettingsData`：`getInitialSettings()` 快照（合并后的有效设置）。
- `initialLocalSettings` / `initialUserSettings`：按来源（`localSettings`、`userSettings`）的原始值，用于精确删除/恢复键。
- `initialAppState`：从 `useAppStateStore` 读取的 `mainLoopModel`、`verbose`、`thinkingEnabled`、`fastMode` 等字段快照。
- `initialThemeSetting`、`initialOutputStyle`、`initialLanguage`、`initialUserMsgOptIn`：独立状态快照。

当用户按 `Esc` 且 `isDirty.current === true` 时，调用 `revertChanges()`：
- 恢复主题（`setTheme`）；
- 覆盖写回全局配置（`saveGlobalConfig(() => initialConfig.current)`）；
- 按来源恢复设置文件（`updateSettingsForSource('localSettings', {...})`、`updateSettingsForSource('userSettings', {...})`），利用 `undefined` 触发删除语义；
- 批量恢复 `AppState`；
- 恢复 `userMsgOptIn`。

### 2.5 变更摘要（Save & Close）

按 `Enter` 保存关闭时，组件对比快照生成人类可读的变更摘要（如 `"Enabled auto-compact"`、`"Set theme to Dark mode"`），通过 `onClose(summary)` 回显到主对话流。

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### `Setting` 联合类型（行 68-83）

```ts
type Setting =
  | (SettingBase & { value: boolean; onChange(value: boolean): void; type: 'boolean' })
  | (SettingBase & { value: string; options: string[]; onChange(value: string): void; type: 'enum' })
  | (SettingBase & { value: string; onChange(value: string): void; type: 'managedEnum' });
```

所有配置项统一在此类型下渲染，列表切片（`slice(scrollOffset, scrollOffset + maxVisible)`）实现虚拟滚动/分页显示。

#### `changes` 对象（行 137-139）

```ts
const [changes, setChanges] = useState<{ [key: string]: unknown }>({});
```

用于记录用户显式修改过的键值对，最终生成关闭摘要。注意：很多配置项的 `onChange` 会同步写盘并更新 `changes`，但 `changes` 本身**不驱动持久化**，仅用于摘要展示。

### 3.2 关键流程

#### 3.2.1 挂载初始化流程

1. 读取全局配置：`getGlobalConfig()` → `globalConfig` state。
2. 读取合并设置：`getInitialSettings()` → `settingsData` state。
3. 读取输出风格、语言等派生状态。
4. 拍摄上述所有快照（`useRef` / `useState` 懒加载）。
5. 计算 `showAutoInDefaultModePicker`、`showDefaultViewPicker` 等功能开关。

#### 3.2.2 配置项切换流程（以 boolean 为例）

以 `autoCompactEnabled` 为例（行 266-283）：

```ts
onChange(autoCompactEnabled: boolean) {
  saveGlobalConfig(current => ({ ...current, autoCompactEnabled }));
  setGlobalConfig({ ...getGlobalConfig(), autoCompactEnabled });
  logEvent('tengu_auto_compact_setting_changed', { enabled: autoCompactEnabled });
}
```

- 调用 `saveGlobalConfig` 写磁盘（`~/.claude/global-config.json`）。
- 调用 `setGlobalConfig` 更新本地 React state，确保 UI 即时反馈。
- 发送 analytics 事件。

对于写入 `settings.json` 的项（如 `spinnerTipsEnabled`），则调用 `updateSettingsForSource('localSettings', ...)`。

#### 3.2.3 搜索模式键盘流程（行 1407-1448）

`handleKeyDown` 是 `Box` 的 `onKeyDown` 处理器，优先级：

1. 子菜单打开 → 直接返回（由子菜单自己处理）。
2. 标签页头部聚焦（`headerFocused`）→ 直接返回（由 `Tabs` 处理左右切换）。
3. **搜索模式**：
   - `Esc`：有查询字符则清空，无则退出搜索模式；
   - `Enter` / `↓` / `wheeldown`：退出搜索模式，选中第一项（`selectedIndex = 0`）。
4. **列表模式**：
   - `←` / `→` / `Tab`：切换当前选中项的值（`toggleSetting`）；
   - 可打印字符（排除 `j`/`k`/`/`）：进入搜索模式并设为首个字符。

#### 3.2.4 列表导航流程（行 1368-1402）

通过 `useKeybindings` 注册：

- `select:previous` / `select:next`：上下移动选中项；在顶部按 `↑` 进入搜索模式。
- `scroll:lineUp` / `scroll:lineDown`：鼠标滚轮映射为上下移动（不进入搜索）。
- `select:accept`：`Space` 默认绑定，触发 `toggleSetting`。
- `settings:close`：`Enter` 默认绑定，触发 `handleSaveAndClose`。
- `settings:search`：`/` 默认绑定，进入搜索模式。

### 3.3 渲染结构

组件返回一个 `Box`（Ink 容器），内部是条件渲染的巨型三元表达式链：

```tsx
<Box onKeyDown={handleKeyDown}>
  {showSubmenu === 'Theme' ? <ThemePicker ... /> :
   showSubmenu === 'Model' ? <ModelPicker ... /> :
   ...
   showSubmenu === 'ChannelDowngrade' ? <ChannelDowngradeDialog ... /> :
   /* 默认：搜索框 + 配置列表 + 底部快捷键提示 */ }
</Box>
```

列表渲染（行 1655-1734）：
- 每行固定 44 字符宽度显示标签；
- 右侧显示当前值或 `THEME_LABELS` / `permissionModeTitle` 等映射后的可读文本；
- `thinkingEnabled` 项在选中且存在 assistant 消息时显示黄色警告文本（行 1679-1684）。

### 3.4 特殊功能门控

文件大量使用 `feature('...')`（来自 `bun:bundle` 的编译期特性标志）和 `getFeatureValue_CACHED_MAY_BE_STALE(...)`（GrowthBook 动态特性）控制功能可见性：

- `TRANSCRIPT_CLASSIFIER`：控制 `auto` 权限模式、`useAutoModeDuringPlan`。
- `KAIROS` / `KAIROS_BRIEF`：控制 `defaultView` 选择器（chat/transcript）。
- `KAIROS_PUSH_NOTIFICATION` / `KAIROS_PUSH_NOTIFICATION`：控制推送通知相关开关。
- `BRIDGE_MODE` + `isBridgeEnabled()`：控制 `remoteControlAtStartup`。
- `tengu_terminal_sidebar`：控制 `showStatusInTerminalTab`。
- `tengu_chomp_inflection`：控制 `promptSuggestionEnabled`。
- `external === 'ant'`：控制内部功能如 `speculationEnabled`。

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖文件（按导入顺序）

| 导入路径 | 用途 |
|---------|------|
| `bun:bundle` (`feature`) | 编译期特性开关 |
| `../../ink.js` | Ink 组件（`Box`, `Text`, `useTheme`, `useThemeSetting`, `useTerminalFocus`） |
| `../../ink/events/keyboard-event.js` | `KeyboardEvent` 类型 |
| `react` | React 核心 |
| `../../keybindings/useKeybinding.js` | `useKeybinding`, `useKeybindings` |
| `figures` | 终端符号（箭头、指针） |
| `../../utils/config.js` | 全局配置读写（`GlobalConfig`, `saveGlobalConfig`, `getGlobalConfig`, `getCurrentProjectConfig`, `OutputStyle`, ...） |
| `../../utils/authPortable.js` | `normalizeApiKeyForConfig` |
| `chalk` | 终端字符串样式（变更摘要加粗） |
| `../../utils/permissions/PermissionMode.js` | 权限模式类型与转换函数 |
| `../../utils/permissions/permissionSetup.js` | 自动模式状态查询与过渡 |
| `../../utils/log.js` | `logError` |
| `src/services/analytics/index.js` | `logEvent` |
| `../../bridge/bridgeEnabled.js` | `isBridgeEnabled` |
| `../ThemePicker.js` | 主题选择子菜单 |
| `../../state/AppState.js` | `useAppState`, `useSetAppState`, `useAppStateStore` |
| `../ModelPicker.js` | 模型选择子菜单 |
| `../../utils/model/model.js` | `modelDisplayString`, `isOpus1mMergeEnabled` |
| `../../utils/extraUsage.js` | `isBilledAsExtraUsage` |
| `../ClaudeMdExternalIncludesDialog.js` | 外部 CLAUDE.md 包含确认弹窗 |
| `../ChannelDowngradeDialog.js` | 自动更新通道降级确认弹窗 |
| `../design-system/Dialog.js` | `Dialog` 组件 |
| `../CustomSelect/index.js` | `Select` 组件 |
| `../OutputStylePicker.js` | 输出风格选择子菜单 |
| `../LanguagePicker.js` | 语言选择子菜单 |
| `src/utils/claudemd.js` | `getExternalClaudeMdIncludes`, `getMemoryFiles` |
| `../design-system/KeyboardShortcutHint.js` | 快捷键提示组件 |
| `../ConfigurableShortcutHint.js` | 可配置快捷键提示 |
| `../design-system/Byline.js` | 底部提示行布局 |
| `../design-system/Tabs.js` | `useTabHeaderFocus` |
| `../../context/modalContext.js` | `useIsInsideModal` |
| `../SearchBox.js` | 搜索输入框 |
| `../../utils/ide.js` | `isSupportedTerminal`, `hasAccessToIDEExtensionDiffFeature` |
| `../../utils/settings/settings.js` | `getInitialSettings`, `getSettingsForSource`, `updateSettingsForSource` |
| `../../bootstrap/state.js` | `getUserMsgOptIn`, `setUserMsgOptIn` |
| `src/constants/outputStyles.js` | `DEFAULT_OUTPUT_STYLE_NAME` |
| `src/utils/envUtils.js` | `isEnvTruthy`, `isRunningOnHomespace` |
| `../../commands.js` | `LocalJSXCommandContext`, `CommandResultDisplay` 类型 |
| `../../services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` |
| `../../utils/agentSwarmsEnabled.js` | `isAgentSwarmsEnabled` |
| `../../utils/swarm/backends/teammateModeSnapshot.js` | `getCliTeammateModeOverride`, `clearCliTeammateModeOverride` |
| `../../utils/swarm/teammateModel.js` | `getHardcodedTeammateModelFallback` |
| `../../hooks/useSearchInput.js` | `useSearchInput` |
| `../../hooks/useTerminalSize.js` | `useTerminalSize` |
| `../../utils/fastMode.js` | 快模式相关工具函数 |
| `../../utils/fullscreen.js` | `isFullscreenEnvEnabled` |

### 4.2 配置持久化路径

- **全局配置**：`~/.claude/global-config.json`（通过 `src/utils/config.ts` 读写）。
- **用户设置**：`~/.claude/settings.json` 或 `cowork_settings.json`（通过 `src/utils/settings/settings.ts`）。
- **本地设置**：`$CWD/.claude/settings.local.json`。
- **项目设置**：`$CWD/.claude/settings.json`。
- **策略设置**：`managed-settings.json`、MDM、HKCU 等（只读，Config.tsx 不直接写入）。

### 4.3 相关命令入口

- `src/commands/config/config.tsx`：渲染 `<Settings defaultTab="Config" />`。
- `src/commands/settings/settings.tsx`：同样渲染 `Settings`，可能默认打开其他标签页。

---

## 5. 依赖与外部交互

### 5.1 状态管理

`Config.tsx` 同时操作三层状态，形成复杂的“写盘即生效”模型：

1. **React Local State**：`globalConfig`、`settingsData`、`currentOutputStyle`、`currentLanguage`、`changes`、`selectedIndex`、`searchQuery` 等，用于 UI 即时反馈。
2. **AppState（全局 Zustand-like Store）**：通过 `useAppState` / `useSetAppState` / `useAppStateStore` 读写。修改的字段包括 `mainLoopModel`、`verbose`、`thinkingEnabled`、`fastMode`、`promptSuggestionEnabled`、`isBriefOnly`、`replBridgeEnabled`、`settings`（如 `prefersReducedMotion`）等。
3. **文件系统（磁盘）**：通过 `saveGlobalConfig` 和 `updateSettingsForSource` 直接写 JSON 文件。

### 5.2 快捷键系统

依赖 `src/keybindings/useKeybinding.ts` 提供的声明式快捷键绑定：

- `useKeybinding('confirm:no', handleEscape, ...)`：Esc 取消。
- `useKeybinding('settings:close', handleSaveAndClose, ...)`：Enter 保存关闭。
- `useKeybindings({ 'select:previous': ..., 'select:next': ..., ... })`：列表导航与切换。

这些绑定通过 `KeybindingContext` 解析，支持用户自定义键位、chord 序列、上下文隔离。

### 5.3 分析（Analytics）

几乎每个配置项的 `onChange` 都会调用 `logEvent('tengu_xxx', {...})`，事件名遵循 `tengu_<feature>_setting_changed` 或 `tengu_config_changed` 规范。注意 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型用于强制开发者确认事件元数据不包含代码或文件路径。

### 5.4 特性标志系统

- **编译期**：`feature('FLAG_NAME')` 来自 `bun:bundle`，在构建时静态消除死代码（如 ant-only 功能在外部构建中被完全剔除）。
- **运行时**：`getFeatureValue_CACHED_MAY_BE_STALE(...)` 来自 GrowthBook，用于 A/B 测试或灰度发布。注释明确提示该值可能过期，因为 Config 面板不会实时重新拉取。

### 5.5 子菜单组件契约

子菜单组件统一接受 `onCancel` 和 `onSelect`/`onComplete`/`onDone` 回调：

- `ThemePicker`：`onThemeSelect(theme)` / `onCancel()`
- `ModelPicker`：`onSelect(model, effort)` / `onCancel()`，支持 `showFastModeNotice`。
- `OutputStylePicker`：`onComplete(style)` / `onCancel()`
- `LanguagePicker`：`onComplete(language)` / `onCancel()`
- `ClaudeMdExternalIncludesDialog`：`onDone()`
- `ChannelDowngradeDialog`：`onChoice(choice)`

所有子菜单关闭时都会调用 `setTabsHidden(false)` 恢复标签页头部。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 立即写盘与回滚不一致风险

由于配置项在切换时**立即写盘**，而 `revertChanges` 是在 `Esc` 时通过快照回写，存在以下边界：

- **并发修改**：如果用户在 Config 面板打开期间，另一个进程（或 CLI 命令）修改了同一配置文件，`revertChanges` 回写的是组件挂载时的旧快照，可能覆盖外部修改。
- **部分失败**：`revertChanges` 执行多个独立写操作（全局配置、localSettings、userSettings、AppState）。如果中间某一步失败（如磁盘权限），可能导致状态半恢复。
- **数组泄漏**：注释特别指出（行 1215-1224），`permissions` 的 `onChange` 曾通过 `...settingsData?.permissions` 将合并后的项目/策略 allow/deny 数组泄漏到 `userSettings`。当前通过显式展开 `iu.permissions` 快照修复，但类似模式在其他嵌套对象中仍可能复现。

#### 6.1.2 特性标志缓存过期

`getFeatureValue_CACHED_MAY_BE_STALE` 在组件生命周期内不会刷新。如果用户在 Config 面板打开期间，远程 GrowthBook 配置发生变化，面板中门控的选项不会动态出现或消失，需要关闭重开才能同步。

#### 6.1.3 模型切换与 Fast Mode 耦合

Fast Mode 开启时（行 351-377），组件不仅写 `userSettings.fastMode`，还会**强制覆盖** `AppState.mainLoopModel` 为快模式专用模型。关闭 Fast Mode 时只重置 `fastMode: false`，**不恢复**用户之前的模型选择，可能导致用户困惑。

#### 6.1.4 搜索模式与快捷键冲突

`handleKeyDown` 和 `useKeybindings` 同时监听键盘事件。虽然代码通过 `e.preventDefault()`、`stopImmediatePropagation` 和 `isActive` 条件做了隔离，但在某些终端模拟器或快速输入场景下，搜索模式的字符捕获与全局快捷键（如 `j`/`k` 导航）仍可能出现竞态。

#### 6.1.5 文件体积与可维护性

`Config.tsx` 长达 1821 行，单文件承载了：
- 30+ 配置项声明；
- 8 种子菜单条件渲染；
- 复杂的快照/回滚逻辑；
- 键盘事件处理；
- 多个辅助函数（`teammateModelDisplayString`、`NotifChannelLabel`、`THEME_LABELS`）。

这使得代码审查、测试覆盖和并行开发都较为困难。

### 6.2 边界行为

- **空列表**：当搜索无结果时，显示 `"No settings match \"<query>\""`。
- **越界保护**：`selectedIndex` 和 `scrollOffset` 通过 `useEffect` 在 `filteredSettingsItems.length` 变化时自动钳位（clamp）。
- **脏状态门控**：`isDirty.current` 是 `useRef`，首次用户可见修改后置为 `true`。如果用户只是打开面板再关闭（未修改），`Esc` 不会触发 `revertChanges`，避免无意义写盘。
- **子菜单 Esc 委托**：子菜单打开时，`useKeybinding('confirm:no', handleEscape)` 的 `isActive` 为 `false`，Esc 由子菜单内部处理，防止误关闭整个 Config 面板。

### 6.3 改进建议

#### 6.3.1 架构层面：拆分配置项定义与渲染

建议将 `settingsItems` 数组提取为独立的 `configDefinitions.ts` 模块，每个配置项只保留 `id`、`label`、`type`、`source`（global/settings）、`featureFlag` 等元数据。`Config.tsx` 只负责遍历渲染、键盘导航和生命周期管理。这样可以：
- 将文件长度缩减到 800 行以内；
- 方便为单个配置项编写单元测试；
- 支持未来动态注册插件配置项。

#### 6.3.2 状态层面：引入事务式提交

当前“即时写盘 + 快照回滚”模型复杂且容易出错。可考虑：
- 在 `Config` 组件内维护一个**本地草稿状态**（`draftConfig`、`draftSettings`）；
- 所有切换只修改草稿；
- `Enter` 时统一批量提交到磁盘和 `AppState`；
- `Esc` 时直接丢弃草稿，无需回写旧值。

这样能彻底消除并发覆盖和半恢复风险，但改动较大，需要协调 `AppState` 的即时反馈需求（如主题切换需要立即生效）。

#### 6.3.3 测试层面：补充集成测试

目前未找到针对 `Config.tsx` 的专用测试文件。建议补充：
- **快照/回滚测试**：模拟修改多个配置项后触发 `Esc`，断言磁盘状态和 `AppState` 恢复为初始值。
- **搜索过滤测试**：模拟输入查询字符，断言 `filteredSettingsItems` 长度和 `selectedIndex` 行为。
- **子菜单生命周期测试**：打开 `ModelPicker` 后取消，断言 `showSubmenu === null` 且 `tabsHidden === false`。
- **快捷键测试**：模拟 `Space`、`Enter`、`/`、`Esc` 等按键，验证 `toggleSetting`、`handleSaveAndClose`、`handleEscape` 被正确调用。

#### 6.3.4 性能层面：减少重复 `getGlobalConfig()` 调用

多个 `onChange` 回调中频繁调用 `getGlobalConfig()` 来构造新 state：

```ts
setGlobalConfig({ ...getGlobalConfig(), verbose: value_0 });
```

虽然 `getGlobalConfig` 有内部缓存，但可通过闭包直接复用 `current` 参数（`saveGlobalConfig` 的 updater 已提供）或本地 state 来避免额外调用。

#### 6.3.5 可访问性/健壮性

- `settingsItems` 中大量内联函数在每次渲染时重新创建，虽然对终端 UI 性能影响有限，但会导致 `React.useMemo`（`filteredSettingsItems`）的依赖数组频繁变化，触发不必要的过滤计算。建议将 `settingsItems` 也包裹在 `useMemo` 中。
- `NotifChannelLabel` 和主组件一样使用了 React Compiler 的 `_c` 缓存机制，但混合手写缓存与编译器缓存增加了心智负担，可考虑统一风格。

---

## 7. 附录：文件引用索引

| 文件路径 | 与本文件关系 |
|---------|-------------|
| `src/components/Settings/Config.tsx` | **目标文件** |
| `src/components/Settings/Settings.tsx` | 父组件，标签页容器 |
| `src/commands/config/config.tsx` | 命令入口，打开 Settings 并定位到 Config 标签 |
| `src/utils/config.ts` | 全局配置（`global-config.json`）读写 |
| `src/utils/settings/settings.ts` | 分层设置文件读写、合并、验证 |
| `src/utils/settings/types.ts` | `SettingsJson`、`SettingsSchema` 定义 |
| `src/state/AppState.tsx` | 全局状态 Provider 与 hooks |
| `src/state/AppStateStore.ts` | `AppState` 类型与默认状态 |
| `src/keybindings/useKeybinding.ts` | 声明式快捷键绑定 hook |
| `src/components/Settings/Usage.tsx` | 同目录兄弟组件（Usage 标签） |
| `src/components/Settings/Status.tsx` | 同目录兄弟组件（Status 标签） |
