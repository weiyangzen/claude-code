# PluginInstallationManager.ts 研究文档

## 场景与职责

`PluginInstallationManager.ts` 是 Claude Code 插件系统的**后台安装管理器**，专门负责在应用启动时**非阻塞式**地自动安装插件和市场（marketplace）。

### 核心定位
- **背景任务执行器**：在应用启动后后台运行，不阻塞主流程
- **市场协调器**：将声明的意图（settings）与实际状态（known_marketplaces.json）进行协调
- **状态同步器**：将安装进度映射到 AppState，供 REPL UI 展示

### 调用场景
1. **应用启动时**：由 `main.tsx` 或 REPL 初始化流程触发
2. **新市场声明后**：当用户在 settings 中声明了新的 marketplace，但尚未 materialize 到本地
3. **市场源变更后**：当 marketplace 的 source 发生变化需要重新克隆/更新

---

## 功能点目的

### 1. 后台插件安装协调 (`performBackgroundPluginInstallations`)

**目的**：在后台完成市场安装，并提供进度反馈。

**关键行为**：
- 计算声明市场与实际 materialized 市场的差异（diff）
- 为待安装的市场初始化 pending 状态到 AppState
- 调用 `reconcileMarketplaces()` 执行实际安装
- 安装完成后自动刷新插件或提示用户手动刷新

### 2. 安装状态管理 (`updateMarketplaceStatus`)

**目的**：将市场安装状态同步到全局 AppState。

**状态流转**：
```
pending → installing → installed/failed
```

### 3. 智能刷新策略

**目的**：区分新安装和更新场景，采取不同的后续动作：

| 场景 | 动作 | 原因 |
|------|------|------|
| 新市场安装 (`result.installed.length > 0`) | 自动调用 `refreshActivePlugins()` | 修复"plugin-not-found"错误（初始缓存加载时市场缓存为空） |
| 市场更新 (`result.updated.length > 0`) | 设置 `needsRefresh: true`，提示用户运行 `/reload-plugins` | 更新不紧急，让用户选择何时应用 |

---

## 具体技术实现

### 关键流程

```
performBackgroundPluginInstallations
├── 1. 计算差异
│   ├── getDeclaredMarketplaces()      # 从 settings 获取声明的市场
│   ├── loadKnownMarketplacesConfig()  # 加载已 materialized 的市场
│   └── diffMarketplaces()             # 计算 missing/sourceChanged
│
├── 2. 初始化 UI 状态
│   └── setAppState()                  # 设置 pending 状态
│
├── 3. 执行协调
│   └── reconcileMarketplaces({
│         onProgress: (event) => {     # 进度回调
│           updateMarketplaceStatus()  # 更新 AppState
│         }
│       })
│
└── 4. 后续处理
    ├── 新安装 → refreshActivePlugins()  # 自动刷新
    └── 更新 → needsRefresh = true       # 提示手动刷新
```

### 数据结构

```typescript
// AppState 中的安装状态
interface PluginInstallationState {
  installationStatus: {
    marketplaces: Array<{
      name: string
      status: 'pending' | 'installing' | 'installed' | 'failed'
      error?: string
    }>
    plugins: []  // 插件级别暂无 pending 状态（加载快）
  }
  needsRefresh: boolean  // 是否需要用户手动刷新
}
```

### 进度事件协议

```typescript
type ReconcileProgressEvent =
  | { type: 'installing'; name: string; action: 'install' | 'update'; index: number; total: number }
  | { type: 'installed'; name: string; alreadyMaterialized: boolean }
  | { type: 'failed'; name: string; error: string }
```

### 关键设计决策

1. **为什么插件没有 per-plugin pending 状态？**
   - 注释说明：插件加载快（缓存命中或本地拷贝），市场克隆才是慢操作

2. **为什么更新不自动刷新？**
   - 新安装必须立即刷新，否则插件找不到
   - 更新可以延后，让用户选择何时重启/刷新

3. **错误处理策略**
   - 自动刷新失败时，回退到 `needsRefresh` 通知
   - 清理缓存以便下次重试

---

## 关键代码路径与文件引用

### 核心函数

| 函数 | 行号 | 描述 |
|------|------|------|
| `performBackgroundPluginInstallations` | 60-184 | 主入口函数 |
| `updateMarketplaceStatus` | 30-48 | 更新 AppState 中的市场状态 |

### 依赖文件

```typescript
// 状态管理
import type { AppState } from '../../state/AppState.js'

// 调试与日志
import { logForDebugging } from '../../utils/debug.js'
import { logForDiagnosticsNoPII } from '../../utils/diagLogs.js'
import { logError } from '../../utils/log.js'

// 市场管理
import {
  clearMarketplacesCache,
  getDeclaredMarketplaces,
  loadKnownMarketplacesConfig,
} from '../../utils/plugins/marketplaceManager.js'

// 插件加载
import { clearPluginCache } from '../../utils/plugins/pluginLoader.js'

// 协调器
import {
  diffMarketplaces,
  reconcileMarketplaces,
} from '../../utils/plugins/reconciler.js'

// 刷新
import { refreshActivePlugins } from '../../utils/plugins/refresh.js'

// 分析
import { logEvent } from '../analytics/index.js'
```

### 调用方

- `src/main.tsx` - 应用启动时调用
- `src/repl/REPL.tsx` - REPL 初始化时调用（通过 useEffect）

---

## 依赖与外部交互

### 与 reconciler.ts 的关系

```
PluginInstallationManager.ts
    ↓ 调用
reconciler.ts: reconcileMarketplaces()
    ↓ 调用
marketplaceManager.ts: addMarketplaceSource()
```

### 与 refresh.ts 的关系

```
新市场安装后
    ↓
refreshActivePlugins()
    ↓
loadAllPlugins() + 更新 AppState
```

### 与 AppState 的交互

```typescript
// 设置 pending 状态
setAppState(prev => ({
  ...prev,
  plugins: {
    ...prev.plugins,
    installationStatus: {
      marketplaces: pendingNames.map(name => ({
        name,
        status: 'pending' as const,
      })),
      plugins: [],
    },
  },
}))

// 设置 needsRefresh
setAppState(prev => ({
  ...prev,
  plugins: { ...prev.plugins, needsRefresh: true },
}))
```

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**
   - 风险：如果用户在后台安装完成前手动运行 `/reload-plugins`，可能导致重复刷新
   - 缓解：refreshActivePlugins 内部有缓存清理，重复调用是幂等的

2. **网络超时**
   - 风险：市场克隆可能因网络问题超时
   - 缓解：由下层 `marketplaceManager.ts` 的 git 操作处理超时和重试

3. **缓存不一致**
   - 风险：自动刷新失败时，缓存可能处于不一致状态
   - 缓解：失败时调用 `clearPluginCache()` 和 `clearMarketplacesCache()` 清理

### 边界情况

| 场景 | 处理 |
|------|------|
| 无待安装市场 | 提前返回，不执行任何操作 |
| 所有安装失败 | 记录错误，不设置 needsRefresh |
| 部分成功部分失败 | 成功的触发刷新/needsRefresh，失败的记录错误 |
| 自动刷新失败 | 回退到 needsRefresh 通知 |

### 改进建议

1. **增加取消机制**
   - 当前：一旦开始无法取消
   - 建议：支持 AbortSignal，允许用户在长时间等待时取消

2. **更细粒度的进度**
   - 当前：只有 market-level 进度
   - 建议：对于大型市场，可考虑 plugin-level 进度

3. **失败重试**
   - 当前：失败后标记为 failed，依赖下次启动重试
   - 建议：增加指数退避重试机制

4. **并发控制**
   - 当前：顺序执行安装
   - 建议：对于独立市场，可考虑并发安装（需控制并发数）
