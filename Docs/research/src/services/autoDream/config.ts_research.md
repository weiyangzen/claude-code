# config.ts 深度研究文档

## 1. 场景与职责

### 1.1 模块定位
`config.ts` 是 `autoDream` 模块的**叶子配置模块（Leaf Config Module）**，设计为最小依赖的独立单元，专门负责提供 auto-dream 功能的启用状态判断。

### 1.2 设计意图
- **最小依赖**：避免引入 `autoDream.ts` 及其庞大的依赖链（forked agent、task registry、message builder 等）
- **UI 友好**：使 UI 组件（如 `MemoryFileSelector.tsx`）可以轻量读取配置状态
- **配置优先级**：支持用户本地设置覆盖服务器端默认配置

### 1.3 核心职责
1. **启用状态判断**：根据用户设置和远程配置决定 auto-dream 是否启用
2. **配置优先级管理**：`settings.json` > GrowthBook 默认值

---

## 2. 功能点目的

### 2.1 配置优先级链
```
用户设置 (settings.json autoDreamEnabled)
    ↓ (如果未设置)
GrowthBook 远程配置 (tengu_onyx_plover.enabled)
    ↓ (如果未返回)
默认: false
```

### 2.2 为什么需要这个独立模块
在 `autoDream.ts` 中直接实现 `isAutoDreamEnabled()` 会导致以下问题：
1. **循环依赖风险**：UI 组件需要读取配置，但 `autoDream.ts` 依赖大量 UI 相关模块
2. **启动性能**：加载 `autoDream.ts` 会触发整个 forked agent 依赖链的初始化
3. **测试隔离**：难以在单元测试中单独 mock 配置

### 2.3 与相关配置的关系
| 配置项 | 模块 | 用途 |
|--------|------|------|
| `autoDreamEnabled` | `config.ts` | auto-dream 专用开关 |
| `autoMemoryEnabled` | `paths.ts` | 自动记忆总开关 |
| `tengu_onyx_plover` | `growthbook.ts` | 远程动态配置（含 minHours/minSessions） |

---

## 3. 具体技术实现

### 3.1 代码实现
```typescript
import { getInitialSettings } from '../../utils/settings/settings.js'
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../analytics/growthbook.js'

export function isAutoDreamEnabled(): boolean {
  // 第一优先级：用户显式设置
  const setting = getInitialSettings().autoDreamEnabled
  if (setting !== undefined) return setting
  
  // 第二优先级：GrowthBook 远程配置
  const gb = getFeatureValue_CACHED_MAY_BE_STALE<{ enabled?: unknown } | null>(
    'tengu_onyx_plover',
    null,
  )
  return gb?.enabled === true
}
```

### 3.2 依赖分析

#### 3.2.1 `getInitialSettings()`
- **来源**：`src/utils/settings/settings.ts`
- **作用**：获取合并后的初始设置（包含 user/policy/flag/local/project 多层设置）
- **特点**：同步返回，已缓存，无 IO 操作

#### 3.2.2 `getFeatureValue_CACHED_MAY_BE_STALE()`
- **来源**：`src/services/analytics/growthbook.ts`
- **作用**：从 GrowthBook 获取远程配置值
- **特点**：
  - 非阻塞（使用磁盘缓存）
  - 可能返回过期值（等待网络初始化完成）
  - 支持环境变量覆盖（`CLAUDE_INTERNAL_FC_OVERRIDES`）

### 3.3 配置类型定义
```typescript
// src/utils/settings/types.ts:950-955
autoDreamEnabled: z
  .boolean()
  .optional()
  .describe(
    'Enable background memory consolidation (auto-dream). When set, overrides the server-side default.',
  ),
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用方
| 文件 | 调用位置 | 用途 |
|------|----------|------|
| `src/services/autoDream/autoDream.ts:99` | `isGateOpen()` | 检查 auto-dream 是否启用 |
| `src/components/memory/MemoryFileSelector.tsx:226` | 设置切换回调 | 更新用户设置 |

### 4.2 调用链
```
isAutoDreamEnabled (config.ts:13)
  ├── getInitialSettings() → settings.ts
  │     └── 合并多层设置源
  └── getFeatureValue_CACHED_MAY_BE_STALE() → growthbook.ts
        ├── 检查环境变量覆盖
        ├── 检查内存缓存
        └── 检查磁盘缓存
```

### 4.3 配置支持定义
```typescript
// src/tools/ConfigTool/supportedSettings.ts:64-70
autoDreamEnabled: {
  type: 'boolean',
  category: 'Memory',
  description: 'Enable background memory consolidation (auto-dream)',
  scope: ['userSettings', 'localSettings'],  // 注意：不支持 projectSettings
},
```

---

## 5. 依赖与外部交互

### 5.1 导入依赖
```typescript
import { getInitialSettings } from '../../utils/settings/settings.js'
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../analytics/growthbook.js'
```

### 5.2 被导入场景
```typescript
// autoDream.ts
import { isAutoDreamEnabled } from './config.js'

// MemoryFileSelector.tsx
import { isAutoDreamEnabled } from '../../services/autoDream/config.js'
```

### 5.3 配置作用域限制
**重要**：`autoDreamEnabled` 不支持在 `projectSettings`（`.claude/settings.json` 提交到仓库）中设置。

原因：
1. **安全性**：防止恶意仓库通过提交的配置强制启用/禁用记忆功能
2. **一致性**：与 `autoMemoryDirectory` 等其他记忆相关设置保持一致的安全策略

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 缓存不一致
- **风险**：`getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期值
- **影响**：用户修改远程配置后，本地可能不会立即生效
- **缓解**：GrowthBook 会定期刷新（外部 6 小时，ant 20 分钟）

#### 6.1.2 设置合并优先级
- **风险**：用户可能在多个设置源中设置冲突值
- **当前行为**：按 `policy > user > flag > local > project` 优先级合并
- **注意**：`getInitialSettings()` 返回的是合并后的最终值

### 6.2 边界条件

| 场景 | 返回值 | 说明 |
|------|--------|------|
| `settings.json`: `"autoDreamEnabled": true` | `true` | 用户显式启用 |
| `settings.json`: `"autoDreamEnabled": false` | `false` | 用户显式禁用 |
| `settings.json`: 未设置，GB `enabled: true` | `true` | 跟随服务器默认 |
| `settings.json`: 未设置，GB `enabled: false` | `false` | 跟随服务器默认 |
| `settings.json`: 未设置，GB 未初始化 | `false` | 安全默认值 |

### 6.3 改进建议

#### 6.3.1 增加调试日志
```typescript
export function isAutoDreamEnabled(): boolean {
  const setting = getInitialSettings().autoDreamEnabled
  if (setting !== undefined) {
    logForDebugging(`[autoDream] Enabled by settings: ${setting}`)
    return setting
  }
  const gb = getFeatureValue_CACHED_MAY_BE_STALE<{ enabled?: unknown } | null>(
    'tengu_onyx_plover',
    null,
  )
  const enabled = gb?.enabled === true
  logForDebugging(`[autoDream] Enabled by GrowthBook: ${enabled}`)
  return enabled
}
```

#### 6.3.2 考虑增加异步版本
如果需要确保获取最新的远程配置：
```typescript
export async function isAutoDreamEnabled_ASYNC(): Promise<boolean> {
  const setting = getInitialSettings().autoDreamEnabled
  if (setting !== undefined) return setting
  
  // 等待 GrowthBook 初始化完成
  const gb = await getFeatureValue_DEPRECATED('tengu_onyx_plover', null)
  return gb?.enabled === true
}
```

#### 6.3.3 配置验证
在设置保存时验证值类型，避免非布尔值被写入：
```typescript
// 在 settings.ts 的验证逻辑中
if (settings.autoDreamEnabled !== undefined && 
    typeof settings.autoDreamEnabled !== 'boolean') {
  errors.push({
    path: ['autoDreamEnabled'],
    message: 'Must be a boolean',
  })
}
```

### 6.4 测试建议
建议增加以下测试用例：
1. 用户设置 `true` 时，忽略 GrowthBook 返回值
2. 用户设置 `false` 时，忽略 GrowthBook 返回值
3. 用户未设置时，正确读取 GrowthBook 值
4. GrowthBook 未初始化时，返回 `false`
5. 设置被清除后，恢复使用 GrowthBook 值
