# DesktopUpsellStartup.tsx 深度研究文档

> 研究对象：`src/components/DesktopUpsell/DesktopUpsellStartup.tsx`
> 研究范围：代码、调用方、被调用方、配置、测试、脚本、必要实现上下文
> 生成时间：2026-04-01

---

## 1. 场景与职责

`DesktopUpsellStartup.tsx` 是 Claude Code CLI 中负责**桌面端应用导流（Desktop Upsell）启动弹窗**的 React 组件模块。其核心职责包括：

1. **启动时条件判断**：在 REPL 主界面挂载后，根据平台、GrowthBook 动态配置、用户历史行为（全局配置）决定是否展示桌面端导流弹窗。
2. **弹窗渲染与交互**：以 `PermissionDialog` + `Select` 组合的形式，向用户展示三选项菜单（"Open in Claude Code Desktop" / "Not now" / "Don't ask again"）。
3. **状态与数据追踪**：记录弹窗展示次数（上限 3 次）、用户是否永久关闭（`desktopUpsellDismissed`），并上报分析事件。
4. **衔接桌面端迁移**：当用户选择 "try" 时，将当前 CLI 会话通过 `DesktopHandoff` 组件无缝迁移到 Claude Desktop 应用（通过 deep link）。

该组件属于**低优先级模态弹窗**（在 REPL 的 `getFocusedInputDialog` 中排在最后），仅在无更高优先级弹窗/通知时展示。

---

## 2. 功能点目的

### 2.1 产品目标
- **引导 CLI 用户迁移到 Desktop 应用**：Desktop 应用提供更丰富的功能（visual diffs、live app preview、parallel sessions 等）。
- **控制打扰频率**：通过全局配置限制最多展示 3 次，并提供 "Don't ask again" 永久关闭选项。
- **A/B 测试与动态开关**：通过 GrowthBook 动态配置 `tengu_desktop_upsell` 控制弹窗和快捷提示的启用状态。

### 2.2 功能拆分

| 功能 | 说明 |
|------|------|
| `getDesktopUpsellConfig()` | 读取 GrowthBook 动态配置，获取 `enable_shortcut_tip` 和 `enable_startup_dialog` 开关 |
| `shouldShowDesktopUpsellStartup()` | 同步判断当前是否满足展示条件（平台支持 + 配置开启 + 未达展示上限 + 未被永久关闭） |
| `DesktopUpsellStartup` 组件 | 实际渲染弹窗，处理用户选择并触发后续动作 |

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### `DesktopUpsellConfig`（本地类型）
```typescript
type DesktopUpsellConfig = {
  enable_shortcut_tip: boolean;   // 控制 spinner tip 中的 /desktop 提示
  enable_startup_dialog: boolean; // 控制启动弹窗是否展示
};
const DESKTOP_UPSELL_DEFAULT: DesktopUpsellConfig = {
  enable_shortcut_tip: false,
  enable_startup_dialog: false,
};
```

#### 全局配置字段（`GlobalConfig` 子集）
在 `src/utils/config.ts` 中定义：
```typescript
export type GlobalConfig = {
  // ...
  desktopUpsellSeenCount?: number;   // 已展示次数，上限 3
  desktopUpsellDismissed?: boolean;  // 用户是否点击了 "Don't ask again"
  // ...
};
```

#### 用户选择类型
```typescript
type DesktopUpsellSelection = 'try' | 'not-now' | 'never';
```

### 3.2 核心流程

#### 流程 A：展示条件判定（`shouldShowDesktopUpsellStartup`）
1. **平台过滤**：仅支持 `darwin`（macOS）和 `win32 x64`（Windows 64位）。Linux 被排除。
2. **动态配置过滤**：调用 `getDynamicConfig_CACHED_MAY_BE_STALE('tengu_desktop_upsell', DESKTOP_UPSELL_DEFAULT)`，检查 `enable_startup_dialog` 是否为 `true`。
3. **用户状态过滤**：
   - `desktopUpsellDismissed === true` → 不展示
   - `desktopUpsellSeenCount >= 3` → 不展示
4. 全部通过 → 返回 `true`。

#### 流程 B：组件挂载后的副作用（`useEffect`）
组件首次渲染时触发 `_temp` 函数：
1. 读取当前 `desktopUpsellSeenCount`，计算 `newCount = current + 1`
2. 调用 `saveGlobalConfig` 原子更新 `desktopUpsellSeenCount`
3. 上报分析事件：`logEvent("tengu_desktop_upsell_shown", { seen_count: newCount })`

> **注意**：该 `useEffect` 的依赖数组为空（`t1 = []`），仅在挂载时执行一次。

#### 流程 C：用户选择处理（`handleSelect`）
| 选项值 | 动作 |
|--------|------|
| `'try'` | `setShowHandoff(true)` → 渲染 `DesktopHandoff` 组件，开始会话迁移 |
| `'never'` | `saveGlobalConfig(_temp2)` 将 `desktopUpsellDismissed` 设为 `true`，然后 `onDone()` 关闭弹窗 |
| `'not-now'` | 直接 `onDone()` 关闭弹窗，不修改任何配置 |

### 3.3 UI 渲染结构

组件使用 React Compiler 编译后的代码（可见 `_c` 缓存机制），原始 JSX 结构等价于：

```tsx
<PermissionDialog title="Try Claude Code Desktop">
  <Box flexDirection="column" paddingX={2} paddingY={1}>
    <Box marginBottom={1}>
      <Text>Same Claude Code with visual diffs, live app preview, parallel sessions, and more.</Text>
    </Box>
    <Select
      options={[
        { label: "Open in Claude Code Desktop", value: "try" },
        { label: "Not now", value: "not-now" },
        { label: "Don't ask again", value: "never" },
      ]}
      onChange={handleSelect}
      onCancel={() => handleSelect("not-now")}
    />
  </Box>
</PermissionDialog>
```

当 `showHandoff === true` 时，提前返回 `<DesktopHandoff onDone={() => onDone()} />`。

### 3.4 与 REPL 的集成

在 `src/screens/REPL.tsx` 中：

```tsx
const [showDesktopUpsellStartup, setShowDesktopUpsellStartup] = useState(() => shouldShowDesktopUpsellStartup());
```

`getFocusedInputDialog()` 函数按优先级返回当前应聚焦的弹窗 ID：
```tsx
// Desktop app upsell (max 3 launches, lowest priority)
if (allowDialogsWithAnimation && showDesktopUpsellStartup) return 'desktop-upsell';
```

渲染条件：
```tsx
{focusedInputDialog === 'desktop-upsell' && (
  <DesktopUpsellStartup onDone={() => setShowDesktopUpsellStartup(false)} />
)}
```

---

## 4. 关键代码路径与文件引用

### 4.1 目标文件
- **`src/components/DesktopUpsell/DesktopUpsellStartup.tsx`**
  - 导出：`getDesktopUpsellConfig()`、`shouldShowDesktopUpsellStartup()`、`DesktopUpsellStartup` 组件

### 4.2 直接调用方
- **`src/screens/REPL.tsx`**（第 254、743、2062、4848 行）
  - 导入 `DesktopUpsellStartup` 和 `shouldShowDesktopUpsellStartup`
  - 在 `getFocusedInputDialog` 中赋予最低优先级
  - 在模态框插槽中条件渲染

- **`src/services/tips/tipRegistry.ts`**（第 10、452 行）
  - 导入 `getDesktopUpsellConfig`
  - 在 `desktop-shortcut` tip 的 `isRelevant` 中读取 `enable_shortcut_tip`

### 4.3 被调用依赖
- **`src/services/analytics/growthbook.ts`**
  - `getDynamicConfig_CACHED_MAY_BE_STALE()` → 实际委托给 `getFeatureValue_CACHED_MAY_BE_STALE('tengu_desktop_upsell', defaultValue)`
  - 读取 GrowthBook 磁盘缓存或内存缓存，支持 env override 和 config override

- **`src/services/analytics/index.ts`**
  - `logEvent("tengu_desktop_upsell_shown", { seen_count: newCount })`
  - 若 sink 未 attach，事件进入队列等待后续 drain

- **`src/utils/config.ts`**
  - `getGlobalConfig()`：内存缓存优先的同步读取
  - `saveGlobalConfig(updater)`：带文件锁、auth-loss guard、回退机制的原子更新
  - 涉及字段：`desktopUpsellSeenCount`、`desktopUpsellDismissed`

- **`src/components/DesktopHandoff.tsx`**
  - 当用户选择 "try" 时渲染
  - 负责检测 Claude Desktop 安装状态、版本兼容性、flush session storage、打开 deep link

- **`src/components/permissions/PermissionDialog.tsx`**
  - 提供带边框标题的模态容器

- **`src/components/CustomSelect/select.tsx`**
  - 提供终端内的选项选择交互

- **`src/ink.ts`**
  - 提供 `Box`、`Text` 等终端 UI 基元

### 4.4 相关命令
- **`src/commands/desktop/desktop.tsx`**
  - `/desktop` 命令入口，直接渲染 `DesktopHandoff` 组件
  - 与 `DesktopUpsellStartup` 共享同一会话迁移逻辑

---

## 5. 依赖与外部交互

### 5.1 动态配置系统（GrowthBook）
- **Feature Key**: `tengu_desktop_upsell`
- **读取方式**: `getDynamicConfig_CACHED_MAY_BE_STALE`
- **特性**: 非阻塞、优先读取内存缓存，其次磁盘缓存；支持 ant-only 的 env override (`CLAUDE_INTERNAL_FC_OVERRIDES`) 和 config override (`growthBookOverrides`)
- **默认值**: `{ enable_shortcut_tip: false, enable_startup_dialog: false }`

### 5.2 全局配置持久化
- **存储位置**: `~/.claude.json`（由 `getGlobalClaudeFile()` 决定）
- **读写机制**:
  - 读：`getGlobalConfig()` 使用内存缓存 `globalConfigCache`，带 freshness watcher（1秒轮询检测其他进程写入）
  - 写：`saveGlobalConfig` 使用文件锁 (`lockfile`)，带 auth-loss 保护（防止覆盖导致 OAuth 状态丢失）
- **并发安全**: 文件锁 + write-through cache + mtime 校验

### 5.3 分析事件
- **事件名**: `tengu_desktop_upsell_shown`
- **Payload**: `{ seen_count: number }`
- **Sink**: Datadog + 1P event logging（由 `attachAnalyticsSink` 决定）

### 5.4 桌面端 Deep Link 协议
- **协议格式**: `claude://resume?session={sessionId}&cwd={cwd}`（生产环境）或 `claude-dev://`（开发环境）
- **实现文件**: `src/utils/desktopDeepLink.ts`
- **最小版本要求**: `1.1.2396`
- **平台检测**:
  - macOS: 检查 `/Applications/Claude.app`
  - Windows: 检查注册表 `HKEY_CLASSES_ROOT\claude`
  - Linux: 被 `DesktopUpsellStartup` 主动排除，但 `desktopDeepLink.ts` 本身支持 `xdg-mime` 检测

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### R1: 平台支持逻辑不一致
- `DesktopUpsellStartup` 的 `isSupportedPlatform()` 仅支持 `darwin` 和 `win32 x64`，但 `DesktopHandoff` / `desktopDeepLink.ts` 实际上也支持 `linux`。
- **风险**: 如果未来 GrowthBook 配置对 Linux 用户开启 `enable_startup_dialog`，`shouldShowDesktopUpsellStartup` 会返回 `false`，导致 Linux 用户永远无法通过启动弹窗获知 Desktop 应用，但可以通过 `/desktop` 命令手动触发。
- **建议**: 统一平台判断逻辑，或明确注释说明 Linux 被排除的产品原因。

#### R2: `useEffect` 依赖数组为空，但读取了可能变化的状态
- 展示计数 `useEffect` 的依赖数组是空数组（`t1 = []`），这意味着即使 `getGlobalConfig()` 在组件挂载前被其他逻辑更新，该 effect 也只会执行一次。
- **风险**: 极低，因为 `shouldShowDesktopUpsellStartup` 在 `useState` 初始化时已经同步判断，组件只有在真正需要展示时才会挂载。

#### R3: 展示次数在组件挂载时即递增
- 用户可能看到弹窗但立即按 Esc（触发 `onCancel` → `"not-now"`），此时 `desktopUpsellSeenCount` 已经 +1。
- **风险**: 用户未真正"看到"弹窗内容即被计入展示次数，可能更快达到 3 次上限。
- **建议**: 考虑将计数逻辑延迟到用户首次交互（如按键/选择）后再递增，但这会牺牲分析数据的简洁性。

#### R4: 无单元测试覆盖
- 搜索整个代码库，未找到针对 `DesktopUpsellStartup`、`shouldShowDesktopUpsellStartup` 或 `getDesktopUpsellConfig` 的单元测试或集成测试。
- **风险**: 重构时容易破坏平台判断逻辑、配置字段交互或 GrowthBook 集成。
- **建议**: 补充测试，覆盖：
  - 各平台 `shouldShowDesktopUpsellStartup` 的返回值
  - 展示次数上限行为
  - `"never"` 选择后配置正确写入
  - `"try"` 选择后正确渲染 `DesktopHandoff`

#### R5: React Compiler 编译后代码的可读性
- 当前文件是 React Compiler（`react/compiler-runtime`）编译后的产物，包含大量 `_c` 缓存槽位逻辑。
- **风险**: 调试困难，手动修改容易破坏 memoization 语义。
- **建议**: 若需修改，优先修改原始 TSX 源码并重新编译，或在源码仓库中定位原始文件。

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| `NODE_ENV === 'test'` | `getGlobalConfig()` 返回 `TEST_GLOBAL_CONFIG_FOR_TESTING`，`saveGlobalConfig` 直接修改该对象 |
| GrowthBook 未初始化 | `getDynamicConfig_CACHED_MAY_BE_STALE` 回退到默认值 `{ enable_shortcut_tip: false, enable_startup_dialog: false }` |
| 用户配置文件中无 `desktopUpsellSeenCount` | 按 `0` 处理 |
| 用户选择 `"try"` 但 Desktop 未安装 | `DesktopHandoff` 会进入 `"prompt-download"` 状态，提示下载 |
| 用户选择 `"try"` 但 Desktop 版本过旧 | `DesktopHandoff` 会提示需要更新到 v1.1.2396+ |

### 6.3 改进建议

1. **测试覆盖**: 为 `shouldShowDesktopUpsellStartup` 和组件交互添加单元测试。
2. **平台统一**: 将 `isSupportedPlatform` 与 `desktopDeepLink.ts` 的平台能力对齐，或提取到共享常量。
3. **事件细化**: 除了 `tengu_desktop_upsell_shown`，可考虑增加 `tengu_desktop_upsell_selected` 事件，记录用户最终选择（`try`/`not-now`/`never`），以便更精确地衡量转化率。
4. **源码管理**: 确认仓库中是否保留了 `DesktopUpsellStartup.tsx` 的原始未编译源码；若仅有编译后产物，建议将原始源码纳入版本控制。

---

*文档结束*
