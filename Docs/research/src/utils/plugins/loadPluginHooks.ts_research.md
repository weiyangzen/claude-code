# loadPluginHooks.ts 深度研究文档

## 文件元数据
- **路径**: `src/utils/plugins/loadPluginHooks.ts`
- **大小**: 10,066 bytes
- **核心职责**: 插件 Hook 的加载、注册与热重载管理

---

## 一、场景与职责

### 1.1 功能定位
本模块是 Claude Code 插件系统的 **Hook 生命周期管理器**，负责：
1. 从已启用的插件中提取 Hook 配置
2. 将插件 Hook 转换为内部使用的 `PluginHookMatcher` 格式
3. 注册 Hook 到全局状态（`STATE.registeredHooks`）
4. 实现基于远程设置变更的 Hook 热重载机制
5. 支持插件禁用/卸载时的 Hook 清理（pruneRemovedPluginHooks）

### 1.2 业务场景
- **会话启动**: `SessionStart` 事件触发 Hook 加载
- **远程设置变更**: 企业策略更新时自动重载 Hook
- **插件管理**: 安装/卸载/启用/禁用插件后清理无效 Hook
- **Stop 事件**: 确保 Stop 钩子正确触发（修复 gh-29767）

---

## 二、功能点目的

### 2.1 Hook 事件类型
支持 25 种 Hook 事件（来自 `HookEvent` 类型）：
```typescript
type HookEvent = 
  | 'PreToolUse' | 'PostToolUse' | 'PostToolUseFailure'
  | 'PermissionDenied' | 'Notification' | 'UserPromptSubmit'
  | 'SessionStart' | 'SessionEnd' | 'Stop' | 'StopFailure'
  | 'SubagentStart' | 'SubagentStop' | 'PreCompact' | 'PostCompact'
  | 'PermissionRequest' | 'Setup' | 'TeammateIdle'
  | 'TaskCreated' | 'TaskCompleted' | 'Elicitation' | 'ElicitationResult'
  | 'ConfigChange' | 'WorktreeCreate' | 'WorktreeRemove'
  | 'InstructionsLoaded' | 'CwdChanged' | 'FileChanged'
```

### 2.2 Hook 匹配器结构
```typescript
type PluginHookMatcher = {
  matcher: string          // 匹配模式（如工具名、文件模式等）
  hooks: string[]          // 要执行的 hook ID 列表
  pluginRoot: string       // 插件根目录路径
  pluginName: string       // 插件名称
  pluginId: string         // 插件唯一标识（name@marketplace）
}
```

### 2.3 热重载机制
通过 `settingsChangeDetector` 订阅设置变更：
- **监控源**: `policySettings`（远程管理设置）
- **变更检测**: 比较 `getPluginAffectingSettingsSnapshot()` 的快照
- **影响字段**: 
  - `enabledPlugins` - 启用的插件列表
  - `extraKnownMarketplaces` - 额外的市场源
  - `strictKnownMarketplaces` - 严格模式市场白名单
  - `blockedMarketplaces` - 市场黑名单

---

## 三、具体技术实现

### 3.1 核心数据流

#### 3.1.1 Hook 加载流程（`loadPluginHooks`）
```
loadPluginHooks (memoized)
  └── loadAllPluginsCacheOnly() → 获取启用的插件
  └── 初始化 allPluginHooks（25 个事件类型的空数组）
  └── 遍历每个启用的插件
      ├── 检查 plugin.hooksConfig
      └── convertPluginHooksToMatchers(plugin)
          ├── 初始化 pluginMatchers（25 个空数组）
          ├── 遍历 hooksConfig 中的每个事件
          └── 将 matcher 包装为 PluginHookMatcher（添加 plugin 上下文）
      └── 合并到 allPluginHooks
  └── clearRegisteredPluginHooks() → 清除旧 Hook
  └── registerHookCallbacks(allPluginHooks) → 注册新 Hook
  └── 记录统计日志
```

#### 3.1.2 Hook 清理流程（`pruneRemovedPluginHooks`）
```
pruneRemovedPluginHooks
  └── 检查 getRegisteredHooks() → 无则提前返回
  └── loadAllPluginsCacheOnly() → 获取当前启用的插件
  └── 构建 enabledRoots Set（插件路径集合）
  └── 重新读取 current hooks（防止并发修改）
  └── 遍历所有已注册的 matcher
      └── 过滤保留 pluginRoot 仍在 enabledRoots 中的
  └── clearRegisteredPluginHooks()
  └── registerHookCallbacks(survivors)
```

### 3.2 关键函数实现

#### 3.2.1 `convertPluginHooksToMatchers`
```typescript
function convertPluginHooksToMatchers(
  plugin: LoadedPlugin
): Record<HookEvent, PluginHookMatcher[]>
```
- **输入**: 插件的 `hooksConfig`（来自插件配置）
- **输出**: 按事件类型分组的 `PluginHookMatcher` 数组
- **关键逻辑**: 为每个 matcher 添加 `pluginRoot`, `pluginName`, `pluginId` 上下文

#### 3.2.2 `getPluginAffectingSettingsSnapshot`
```typescript
export function getPluginAffectingSettingsSnapshot(): string
```
- **用途**: 生成用于变更检测的稳定快照字符串
- **排序**: 对 `enabledPlugins` 和 `extraKnownMarketplaces` 按键排序，确保比较确定性
- **字段**: 包含 4 个影响插件加载的设置字段

#### 3.2.3 `setupPluginHookHotReload`
```typescript
export function setupPluginHookHotReload(): void
```
- **幂等性**: 通过 `hotReloadSubscribed` 标志确保只订阅一次
- **初始快照**: 捕获初始状态用于后续比较
- **订阅回调**: 仅在 `policySettings` 变更且快照不同时触发重载

### 3.3 缓存与状态管理

| 变量/函数 | 类型 | 说明 |
|-----------|------|------|
| `loadPluginHooks` | memoized async | 主加载函数，缓存加载结果 |
| `hotReloadSubscribed` | boolean | 热重载订阅状态（模块级） |
| `lastPluginSettingsSnapshot` | string | 上次设置快照 |
| `clearPluginHookCache` | function | 仅清除 memoize 缓存 |
| `resetHotReloadState` | function | 重置订阅状态（测试用） |

---

## 四、关键代码路径与文件引用

### 4.1 入口点
| 函数 | 导出类型 | 调用方 |
|------|----------|--------|
| `loadPluginHooks` | memoized async | `src/setup.ts`, `src/utils/sessionStart.ts`, `src/utils/plugins/pluginLoader.ts` |
| `clearPluginHookCache` | function | `src/utils/plugins/cacheUtils.ts` |
| `pruneRemovedPluginHooks` | async function | `src/utils/plugins/cacheUtils.ts`, `src/utils/plugins/pluginLoader.ts` |
| `setupPluginHookHotReload` | function | `src/setup.ts` |
| `resetHotReloadState` | function | 测试代码 |

### 4.2 关键依赖
```typescript
// 核心依赖
import { loadAllPluginsCacheOnly, clearPluginCache } from './pluginLoader.js'
import { 
  clearRegisteredPluginHooks, 
  getRegisteredHooks, 
  registerHookCallbacks 
} from '../../bootstrap/state.js'
import { settingsChangeDetector } from '../settings/changeDetector.js'
import { getSettings_DEPRECATED, getSettingsForSource } from '../settings/settings.js'

// 类型定义
import type { HookEvent } from 'src/entrypoints/agentSdkTypes.js'
import type { LoadedPlugin } from '../../types/plugin.js'
import type { PluginHookMatcher } from '../settings/types.js'
```

### 4.3 文件引用关系
```
loadPluginHooks.ts
  ├── pluginLoader.ts           # 加载插件列表、清除插件缓存
  ├── ../../bootstrap/state.js  # Hook 注册/清除/获取
  ├── ../settings/changeDetector.js  # 设置变更检测
  ├── ../settings/settings.js   # 设置读取
  └── ../slowOperations.js      # jsonStringify
```

---

## 五、依赖与外部交互

### 5.1 上游依赖（被调用）
| 模块 | 用途 |
|------|------|
| `pluginLoader.ts` | 获取已启用的插件列表、清除插件缓存 |
| `bootstrap/state.js` | 注册/清除/获取全局 Hook 状态 |
| `settings/changeDetector.js` | 订阅设置变更事件 |
| `settings/settings.js` | 读取设置值 |

### 5.2 下游消费者（调用方）
| 模块 | 用途 |
|------|------|
| `src/setup.ts` | 初始化 Hook 热重载 |
| `src/utils/sessionStart.ts` | 会话启动时加载 Hook |
| `src/utils/plugins/pluginLoader.ts` | 插件加载流程中调用 |
| `src/utils/plugins/cacheUtils.ts` | 缓存清除时清理 Hook |
| `src/cli/print.ts` | CLI 打印时加载 Hook |

### 5.3 全局状态交互
```typescript
// bootstrap/state.js 中的相关状态
STATE.registeredHooks: Record<HookEvent, PluginHookMatcher[]>

// 操作函数
registerHookCallbacks(hooks: Partial<Record<HookEvent, PluginHookMatcher[]>>): void
clearRegisteredPluginHooks(): void
getRegisteredHooks(): Record<HookEvent, PluginHookMatcher[]> | null
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险与修复

#### 6.1.1 Stop Hook 失效问题（gh-29767）
- **问题**: `clearAllCaches()` 会调用 `clearPluginHookCache()`，后者旧实现会清除 `STATE.registeredHooks`，导致 Stop 事件触发时 Hook 已丢失
- **修复**: 将 `clearRegisteredPluginHooks()` 移至 `loadPluginHooks()` 内部，实现"清除+注册"原子操作
- **代码注释**: 
  ```typescript
  // Clear-then-register as an atomic pair... Doing the clear here makes the swap
  // atomic — old hooks stay valid until this point, new hooks take over.
  ```

#### 6.1.2 并发修改风险
- **场景**: `pruneRemovedPluginHooks` 中 `await loadAllPluginsCacheOnly()` 后重新读取 `getRegisteredHooks()`
- **原因**: 防止并发 `loadPluginHooks()`（热重载）在此期间修改了 hooks
- **处理**: 使用重新读取的引用计算幸存者

### 6.2 边界情况

| 场景 | 处理方式 |
|------|----------|
| 插件无 hooksConfig | 跳过，继续处理下一个插件 |
| matcher.hooks 为空数组 | 不添加到 pluginMatchers |
| 热重载时设置未变更 | 通过快照比较提前返回 |
| 无已注册 Hook | `pruneRemovedPluginHooks` 提前返回 |
| 测试环境重置 | `resetHotReloadState()` 重置订阅状态 |

### 6.3 改进建议

#### 6.3.1 性能优化
1. **增量 Hook 更新**: 当前热重载会重新加载所有 Hook，可优化为仅变更差异
2. **并行加载**: 插件 Hook 转换目前是串行，可改为并行
3. **快照优化**: `getPluginAffectingSettingsSnapshot` 每次调用都重新序列化，可缓存

#### 6.3.2 可靠性增强
1. **Hook 执行超时**: 当前无 Hook 执行超时机制，恶意/慢 Hook 可能阻塞事件
2. **Hook 错误隔离**: 单个 Hook 失败不应影响同事件的其他 Hook
3. **Hook 依赖排序**: 支持定义 Hook 执行顺序（优先级）

#### 6.3.3 可观测性
1. **Hook 执行指标**: 记录 Hook 执行时间、成功率
2. **Hook 调试模式**: 详细日志记录 Hook 匹配和执行过程
3. **Hook 追踪 ID**: 为 Hook 执行链添加追踪标识

#### 6.3.4 代码质量
1. **魔法数字**: `MAX_IGNORED_COUNT` 等常量应集中定义
2. **类型安全**: `PluginHookMatcher` 的 `matcher` 字段类型可更精确（支持不同事件的不同匹配模式）
3. **测试覆盖**: 热重载和并发场景需要更多测试

### 6.4 技术债务
1. **DEPRECATED API**: 使用 `getSettings_DEPRECATED`，需关注迁移计划
2. **字符串序列化**: 使用 `jsonStringify` 生成快照，存在字段顺序风险（已通过排序缓解）
3. **全局状态**: 依赖 `STATE.registeredHooks` 全局状态，增加测试复杂度

---

## 七、附录

### 7.1 Hook 配置示例（插件配置）
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["validate-bash-command"]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": ["cleanup-temp-files"]
      }
    ]
  }
}
```

### 7.2 热重载触发条件
```typescript
// 以下条件同时满足时触发重载
source === 'policySettings' && newSnapshot !== lastPluginSettingsSnapshot

// 重载操作序列
clearPluginCache('loadPluginHooks: plugin-affecting settings changed')
clearPluginHookCache()
void loadPluginHooks()  // fire-and-forget
```

### 7.3 关键 Bug 修复历史
| Issue | 描述 | 修复方案 |
|-------|------|----------|
| gh-29767 | Stop Hook 在缓存清除后失效 | 将 clear+register 改为原子操作 |
| gh-36995 | 禁用插件后 Hook 仍触发 | 添加 `pruneRemovedPluginHooks` |
| #23085 / #23152 | 远程设置变更未触发重载 | 扩展快照包含所有相关字段 |
