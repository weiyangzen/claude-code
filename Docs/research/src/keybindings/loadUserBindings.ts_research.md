# loadUserBindings.ts 研究文档

## 场景与职责

`loadUserBindings.ts` 是 Claude Code 键盘快捷键系统的配置加载模块，负责从用户配置文件（`~/.claude/keybindings.json`）加载自定义绑定，并提供热重载支持。它是连接默认绑定和用户自定义的桥梁。

**核心职责：**
1. **配置加载**：从 `~/.claude/keybindings.json` 加载用户绑定
2. **访问控制**：通过 GrowthBook 功能标志限制自定义绑定功能（仅 Anthropic 员工）
3. **验证与警告**：解析和验证用户配置，生成警告/错误
4. **热重载**：使用 `chokidar` 监视文件变化，自动重新加载
5. **缓存管理**：缓存加载结果，优化同步访问性能
6. **遥测**：记录自定义绑定使用情况（每日一次）

**架构位置：**
```
defaultBindings.ts (默认绑定)
    ↓
loadUserBindings.ts (本文件) ← ~/.claude/keybindings.json
    ↓ 合并
KeybindingSetup
    ↓
KeybindingProvider
```

---

## 功能点目的

### 1. 访问控制

**功能标志检查：**
```typescript
export function isKeybindingCustomizationEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_keybinding_customization_release',
    false,
  )
}
```

**当前限制：**
- 仅 Anthropic 员工可用（`USER_TYPE === 'ant'`）
- 外部用户始终使用默认绑定
- 功能标志名：`tengu_keybinding_customization_release`

### 2. 配置加载

**异步加载（`loadKeybindings`）：**
- 用于后台加载和文件变更时
- 完整的错误处理和警告生成
- 返回 `Promise<KeybindingsLoadResult>`

**同步加载（`loadKeybindingsSync` / `loadKeybindingsSyncWithWarnings`）：**
- 用于 React 初始化（`useState` 初始化器）
- 使用缓存避免重复文件 I/O
- 同步版本返回 `ParsedBinding[]`
- 带警告版本返回 `KeybindingsLoadResult`

### 3. 文件监视（热重载）

**配置参数：**
```typescript
const FILE_STABILITY_THRESHOLD_MS = 500    // 文件写入稳定时间
const FILE_STABILITY_POLL_INTERVAL_MS = 200 // 轮询间隔
```

**chokidar 配置：**
```typescript
watcher = chokidar.watch(userPath, {
  persistent: true,
  ignoreInitial: true,           // 忽略初始添加事件
  awaitWriteFinish: {            // 等待写入完成
    stabilityThreshold: FILE_STABILITY_THRESHOLD_MS,
    pollInterval: FILE_STABILITY_POLL_INTERVAL_MS,
  },
  ignorePermissionErrors: true,
  usePolling: false,             // 使用原生 fs.watch
  atomic: true,                  // 原子写入支持
})
```

**事件处理：**
- `add`: 文件创建
- `change`: 文件修改
- `unlink`: 文件删除（重置为默认绑定）

### 4. 配置格式验证

**期望格式：**
```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+enter": "chat:submit",
        "ctrl+t": null
      }
    }
  ]
}
```

**验证步骤：**
1. JSON 解析
2. 检查顶层 `bindings` 属性
3. 验证每个块的 `context`（字符串）和 `bindings`（对象）
4. 解析并验证每个键绑定
5. 检查重复键
6. 检查保留快捷键冲突

### 5. 信号系统

**自定义信号实现：**
```typescript
const keybindingsChanged = createSignal<[result: KeybindingsLoadResult]>()
```

**使用方式：**
```typescript
// 订阅变更
const unsubscribe = subscribeToKeybindingChanges(result => {
  // 更新绑定
})

// 触发变更
keybindingsChanged.emit({ bindings, warnings })
```

---

## 具体技术实现

### 1. 缓存机制

**缓存变量：**
```typescript
let cachedBindings: ParsedBinding[] | null = null
let cachedWarnings: KeybindingWarning[] = []
```

**缓存策略：**
- 首次加载后缓存结果
- 文件变更时更新缓存
- 同步加载优先使用缓存

**缓存失效：**
- 文件变更时自动更新
- 文件删除时重置为默认
- 测试时可通过 `resetKeybindingLoaderForTesting()` 手动重置

### 2. 配置解析流程

```typescript
export async function loadKeybindings(): Promise<KeybindingsLoadResult> {
  const defaultBindings = getDefaultParsedBindings()
  
  // 1. 检查功能标志
  if (!isKeybindingCustomizationEnabled()) {
    return { bindings: defaultBindings, warnings: [] }
  }
  
  const userPath = getKeybindingsPath()
  
  try {
    // 2. 读取文件
    const content = await readFile(userPath, 'utf-8')
    const parsed: unknown = jsonParse(content)
    
    // 3. 验证格式
    if (!hasBindingsProperty(parsed)) {
      return { bindings: defaultBindings, warnings: [parseError] }
    }
    
    // 4. 验证结构
    if (!isKeybindingBlockArray(userBlocks)) {
      return { bindings: defaultBindings, warnings: [structureError] }
    }
    
    // 5. 解析绑定
    const userParsed = parseBindings(userBlocks)
    
    // 6. 合并（用户在后，覆盖默认）
    const mergedBindings = [...defaultBindings, ...userParsed]
    
    // 7. 验证
    const duplicateKeyWarnings = checkDuplicateKeysInJson(content)
    const warnings = [...duplicateKeyWarnings, ...validateBindings(userBlocks, mergedBindings)]
    
    return { bindings: mergedBindings, warnings }
  } catch (error) {
    // 文件不存在：使用默认
    if (isENOENT(error)) {
      return { bindings: defaultBindings, warnings: [] }
    }
    // 其他错误：返回警告
    return { bindings: defaultBindings, warnings: [parseError] }
  }
}
```

### 3. 文件路径解析

```typescript
export function getKeybindingsPath(): string {
  return join(getClaudeConfigHomeDir(), 'keybindings.json')
}
```

**配置目录：**
- 通过 `getClaudeConfigHomeDir()` 获取
- 通常是 `~/.claude/`

### 4. 遥测记录

**每日一次记录：**
```typescript
let lastCustomBindingsLogDate: string | null = null

function logCustomBindingsLoadedOncePerDay(userBindingCount: number): void {
  const today = new Date().toISOString().slice(0, 10)
  if (lastCustomBindingsLogDate === today) return
  lastCustomBindingsLogDate = today
  
  logEvent('tengu_custom_keybindings_loaded', {
    user_binding_count: userBindingCount,
  })
}
```

**目的：**
- 估计自定义绑定的用户比例
- 不记录具体绑定内容（隐私保护）
- 仅记录绑定数量

### 5. 类型守卫

```typescript
function isKeybindingBlock(obj: unknown): obj is KeybindingBlock {
  if (typeof obj !== 'object' || obj === null) return false
  const b = obj as Record<string, unknown>
  return (
    typeof b.context === 'string' &&
    typeof b.bindings === 'object' &&
    b.bindings !== null
  )
}

function isKeybindingBlockArray(arr: unknown): arr is KeybindingBlock[] {
  return Array.isArray(arr) && arr.every(isKeybindingBlock)
}
```

---

## 关键代码路径与文件引用

### 核心常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `FILE_STABILITY_THRESHOLD_MS` | 500 | 文件写入稳定等待时间 |
| `FILE_STABILITY_POLL_INTERVAL_MS` | 200 | 文件稳定性轮询间隔 |

### 关键依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `chokidar` | `FSWatcher` | 文件监视 |
| `fs` | `readFileSync` | 同步文件读取 |
| `fs/promises` | `readFile`, `stat` | 异步文件操作 |
| `path` | `dirname`, `join` | 路径处理 |
| `../services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` | 功能标志 |
| `../services/analytics/index.js` | `logEvent` | 遥测 |
| `../utils/cleanupRegistry.js` | `registerCleanup` | 清理注册 |
| `../utils/debug.js` | `logForDebugging` | 调试日志 |
| `../utils/envUtils.js` | `getClaudeConfigHomeDir` | 配置目录 |
| `../utils/errors.js` | `errorMessage`, `isENOENT` | 错误处理 |
| `../utils/signal.js` | `createSignal` | 信号系统 |
| `../utils/slowOperations.js` | `jsonParse` | JSON 解析 |
| `./defaultBindings.js` | `DEFAULT_BINDINGS` | 默认绑定 |
| `./parser.js` | `parseBindings` | 绑定解析 |
| `./types.js` | `KeybindingBlock`, `ParsedBinding` | 类型定义 |
| `./validate.js` | `checkDuplicateKeysInJson`, `validateBindings` | 验证 |

### 类型定义

```typescript
export type KeybindingsLoadResult = {
  bindings: ParsedBinding[]
  warnings: KeybindingWarning[]
}
```

### 导出符号

```typescript
export { isKeybindingCustomizationEnabled }     // 功能标志检查
export { loadKeybindings }                        // 异步加载
export { loadKeybindingsSync }                    // 同步加载（仅绑定）
export { loadKeybindingsSyncWithWarnings }        // 同步加载（带警告）
export { initializeKeybindingWatcher }            // 初始化监视器
export { disposeKeybindingWatcher }               // 清理监视器
export { subscribeToKeybindingChanges }           // 订阅变更
export { getKeybindingsPath }                     // 获取配置路径
export { getCachedKeybindingWarnings }            // 获取缓存警告
export { resetKeybindingLoaderForTesting }        // 测试重置
```

---

## 依赖与外部交互

### 1. 上游依赖（输入）

**文件系统：**
- `~/.claude/keybindings.json` - 用户配置文件

**功能标志：**
- `tengu_keybinding_customization_release` - 自定义绑定开关

**默认绑定：**
- `DEFAULT_BINDINGS` - 基础绑定配置

### 2. 下游消费（输出）

**被 KeybindingProviderSetup.tsx 消费：**
```typescript
import { 
  initializeKeybindingWatcher,
  loadKeybindingsSyncWithWarnings,
  subscribeToKeybindingChanges 
} from './loadUserBindings.js'
```

**被 shortcutFormat.ts 消费：**
```typescript
import { loadKeybindingsSync } from './loadUserBindings.js'
```

**被 /doctor 命令消费（间接）：**
- 通过 `getCachedKeybindingWarnings()` 获取警告

### 3. 生命周期管理

**初始化：**
1. `initializeKeybindingWatcher()` - 应用启动时调用
2. 检查功能标志，决定是否启用监视
3. 验证配置目录存在

**运行时：**
1. 文件变更触发 `handleChange()`
2. 重新加载绑定
3. 更新缓存
4. 通知订阅者

**清理：**
1. `disposeKeybindingWatcher()` - 应用关闭时调用
2. 关闭 `chokidar` 监视器
3. 清除信号订阅

---

## 风险、边界与改进建议

### 1. 已知风险

**功能标志缓存：**
- 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期值
- 如果标志在运行时变更，不会立即生效

**文件监视器竞态：**
- 快速连续保存文件可能导致多次重载
- `awaitWriteFinish` 配置缓解但不完全消除

**JSON 解析错误：**
- 无效 JSON 导致整个配置被忽略
- 用户可能困惑为什么绑定不生效

**同步加载阻塞：**
- `loadKeybindingsSync` 使用 `readFileSync`，阻塞事件循环
- 虽然文件很小，但在启动时仍可能影响性能

### 2. 边界情况

**文件不存在：**
- 首次使用或删除配置后，静默使用默认绑定
- 用户可通过 `/keybindings` 命令创建模板

**空配置文件：**
```json
{ "bindings": [] }
```
- 有效配置，使用所有默认绑定

**部分无效配置：**
- 某些块无效时，整个用户配置被拒绝
- 使用默认绑定并返回错误警告

**权限错误：**
- `ignorePermissionErrors: true` 忽略权限问题
- 可能静默失败，用户不知道绑定未加载

### 3. 改进建议

**错误报告：**
```typescript
// 建议：更详细的错误信息，包括行号
// 当前："Failed to parse keybindings.json: ..."
// 改进："Line 15: Unexpected token '}'"
```

**配置恢复：**
- 添加备份机制，保存上次有效配置
- 当前配置损坏时可恢复

**热重载优化：**
- 添加防抖，避免快速保存时的多次重载
- 当前 500ms 稳定时间可能不够

**访问控制改进：**
- 当前功能标志检查在运行时
- 建议构建时根据用户类型排除代码，减小包体积

### 4. 测试建议

- 测试文件不存在时的行为
- 测试无效 JSON 的处理
- 测试热重载（修改、删除、恢复文件）
- 测试功能标志开关
- 测试缓存一致性
- 测试并发加载（同步 + 异步）

### 5. 安全考虑

**路径遍历：**
- `getKeybindingsPath()` 使用 `getClaudeConfigHomeDir()`
- 确保不会读取用户指定路径外的文件

**配置注入：**
- 绑定动作是预定义的，用户不能定义任意动作
- 但 `command:` 前缀允许执行任意 slash 命令

**隐私保护：**
- 遥测仅记录绑定数量，不记录具体绑定
- 符合隐私政策
